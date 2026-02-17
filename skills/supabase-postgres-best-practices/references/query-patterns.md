# Query Performance & Connection Management

Reference patterns for Supabase Postgres query optimization and connection management.

## 1. Query Performance (CRITICAL)

### Use Appropriate Indexes

```sql
-- BAD: Full table scan on frequently queried column
SELECT * FROM transfers WHERE sender_id = 'abc-123';

-- GOOD: Create index on frequently queried columns
CREATE INDEX idx_transfers_sender_id ON transfers(sender_id);
```

### Avoid SELECT *

```sql
-- BAD: Fetches all columns including large payloads
SELECT * FROM transfers WHERE status = 'pending';

-- GOOD: Select only needed columns
SELECT id, amount, currency, status, created_at
FROM transfers WHERE status = 'pending';
```

### Use EXPLAIN ANALYZE

Always analyze query plans before deploying:

```sql
EXPLAIN ANALYZE
SELECT t.id, t.amount, u.full_name
FROM transfers t
JOIN users u ON t.sender_id = u.id
WHERE t.status = 'completed'
ORDER BY t.created_at DESC
LIMIT 20;
```

Look for: Sequential scans on large tables, nested loop joins on large datasets, high actual vs estimated rows discrepancy.

### Avoid N+1 Queries

```sql
-- BAD: Querying in a loop (N+1 pattern)
-- For each user, SELECT * FROM transfers WHERE sender_id = user.id

-- GOOD: Single query with JOIN
SELECT u.id, u.full_name, t.amount, t.currency
FROM users u
JOIN transfers t ON t.sender_id = u.id
WHERE u.id = ANY($1);
```

### Use Partial Indexes

```sql
-- GOOD: Index only active/pending records (smaller, faster)
CREATE INDEX idx_transfers_pending
ON transfers(created_at)
WHERE status = 'pending';
```

### Pagination Best Practices

```sql
-- BAD: OFFSET-based pagination (slow for large offsets)
SELECT * FROM transfers ORDER BY created_at DESC OFFSET 10000 LIMIT 20;

-- GOOD: Cursor-based pagination (consistent performance)
SELECT * FROM transfers
WHERE created_at < $1  -- cursor from last result
ORDER BY created_at DESC
LIMIT 20;
```

---

## 2. Connection Management (CRITICAL)

### Use Connection Pooling

- Always use Supabase's built-in PgBouncer connection pooler
- Use port `6543` for pooled connections (not `5432` for direct)
- Set pool mode to `transaction` for serverless/Edge Functions
- Use `session` mode only when prepared statements or advisory locks are needed

### Edge Function Connection Patterns

```typescript
// GOOD: Use Supabase client (handles pooling automatically)
import { createClient } from '@supabase/supabase-js'
const supabase = createClient(url, key)

// BAD: Direct Postgres connections from Edge Functions
// (each invocation creates a new connection, exhausting pool)
```

### Connection Limits

- Free tier: 60 direct connections
- Pro tier: 200 direct connections
- Always monitor connection usage via `pg_stat_activity`

```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';

-- Find long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```
