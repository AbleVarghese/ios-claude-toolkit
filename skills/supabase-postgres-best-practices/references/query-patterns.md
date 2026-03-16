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
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT t.id, t.amount, u.full_name
FROM transfers t
JOIN users u ON t.sender_id = u.id
WHERE t.status = 'completed'
ORDER BY t.created_at DESC
LIMIT 20;
```

**How to read the output:**

```
Limit  (cost=0.86..45.32 rows=20 width=48) (actual time=0.128..0.342 rows=20 loops=1)
  ->  Nested Loop  (cost=0.86..1234.56 rows=555 width=48) (actual time=0.126..0.338 rows=20 loops=1)
        Buffers: shared hit=65
Planning Time: 0.245 ms
Execution Time: 0.389 ms
```

| Field | Good | Bad |
|-------|------|-----|
| Node type | Index Scan, Index Only Scan | Seq Scan on large table |
| actual rows vs rows | Close match | 10x+ discrepancy (run ANALYZE) |
| Buffers: shared hit | High (from cache) | shared read high (disk I/O) |
| Execution Time | < 10ms simple, < 100ms complex | > 500ms for any user-facing query |

### EXPLAIN ANALYZE Warning Patterns

```sql
-- WARNING: Seq Scan with filter removing most rows
Seq Scan on transfers  (actual time=0.025..145.326 rows=50 loops=1)
  Filter: (status = 'pending')
  Rows Removed by Filter: 999950
-- FIX: CREATE INDEX idx_transfers_status ON transfers(status);

-- WARNING: Sort using external merge (spilling to disk)
Sort  (actual time=1234.567..1456.789 rows=100000 loops=1)
  Sort Method: external merge  Disk: 45000kB
-- FIX: CREATE INDEX idx_transfers_created_at ON transfers(created_at DESC);
-- Or increase work_mem: SET work_mem = '256MB';

-- WARNING: Nested Loop with high loops count
Nested Loop  (actual time=0.050..2345.678 rows=50000 loops=1)
  ->  Seq Scan on transfers  (actual rows=50000 loops=1)
  ->  Index Scan on users  (actual rows=1 loops=50000)
-- FIX: Already using index, but 50K lookups. Consider batch query or materialize.
```

### JOIN Optimization

```sql
-- BAD: Joining without indexes on join columns
SELECT t.*, u.full_name
FROM transfers t
JOIN users u ON t.sender_id = u.id;
-- If sender_id is not indexed, this triggers nested loop with seq scan

-- GOOD: Ensure both sides of JOIN are indexed
CREATE INDEX idx_transfers_sender_id ON transfers(sender_id);
-- users.id is already indexed (primary key)

-- GOOD: Use appropriate JOIN type
-- INNER JOIN: Only rows with matches in both tables (default)
-- LEFT JOIN: All rows from left table, NULL for non-matches
-- Don't use LEFT JOIN when you actually mean INNER JOIN (wastes resources)

-- GOOD: Filter before joining when possible
SELECT t.id, t.amount, u.full_name
FROM (
    SELECT id, amount, sender_id
    FROM transfers
    WHERE status = 'completed' AND created_at > now() - interval '30 days'
) t
JOIN users u ON t.sender_id = u.id;
```

### CTE Performance Considerations

```sql
-- CAUTION: CTEs in Postgres < 12 are optimization fences (always materialized)
-- In Postgres 12+, simple CTEs are inlined (optimized)

-- GOOD: Use CTE for readability when Postgres can inline it
WITH recent_transfers AS (
    SELECT id, amount, sender_id, status
    FROM transfers
    WHERE created_at > now() - interval '30 days'
)
SELECT rt.id, rt.amount, u.full_name
FROM recent_transfers rt
JOIN users u ON rt.sender_id = u.id
WHERE rt.status = 'completed';

-- CAUTION: MATERIALIZED forces the CTE to execute separately
-- Only use when you deliberately want to avoid re-optimization
WITH MATERIALIZED transfer_stats AS (
    SELECT sender_id, count(*) as cnt, sum(amount) as total
    FROM transfers
    GROUP BY sender_id
)
SELECT * FROM transfer_stats WHERE cnt > 10;

-- GOOD: Use CTE for recursive queries (the primary CTE use case)
WITH RECURSIVE referral_chain AS (
    SELECT id, referrer_id, 1 as depth
    FROM users WHERE id = $1
    UNION ALL
    SELECT u.id, u.referrer_id, rc.depth + 1
    FROM users u
    JOIN referral_chain rc ON u.referrer_id = rc.id
    WHERE rc.depth < 10
)
SELECT * FROM referral_chain;
```

### Avoid N+1 Queries

```sql
-- BAD: Querying in a loop (N+1 pattern)
-- For each user, SELECT * FROM transfers WHERE sender_id = user.id

-- GOOD: Single query with JOIN
SELECT u.id, u.full_name, t.amount, t.currency
FROM users u
JOIN transfers t ON t.sender_id = u.id
WHERE u.id = ANY($1);

-- GOOD: Batch query with array parameter
SELECT * FROM transfers
WHERE sender_id = ANY($1::uuid[])
ORDER BY created_at DESC;
```

### Use Partial Indexes

```sql
-- GOOD: Index only active/pending records (smaller, faster)
CREATE INDEX idx_transfers_pending
ON transfers(created_at)
WHERE status = 'pending';

-- GOOD: Partial index for soft-deleted records
CREATE INDEX idx_users_active
ON users(email)
WHERE deleted_at IS NULL;

-- GOOD: Partial unique index
CREATE UNIQUE INDEX idx_users_active_email
ON users(email)
WHERE deleted_at IS NULL;
-- Allows "deleted" users to have duplicate emails
```

### Covering Indexes

```sql
-- GOOD: Include columns needed by the query to avoid table lookup
CREATE INDEX idx_transfers_status_covering
ON transfers(status)
INCLUDE (id, amount, created_at);

-- This allows Index Only Scan when querying:
SELECT id, amount, created_at FROM transfers WHERE status = 'pending';
-- No need to fetch from the heap (table data pages)
```

### Pagination Best Practices

```sql
-- BAD: OFFSET-based pagination (slow for large offsets)
SELECT * FROM transfers ORDER BY created_at DESC OFFSET 10000 LIMIT 20;
-- Postgres must scan and discard 10,000 rows before returning 20

-- GOOD: Cursor-based pagination (consistent performance)
SELECT * FROM transfers
WHERE created_at < $1  -- cursor from last result
ORDER BY created_at DESC
LIMIT 20;

-- GOOD: Keyset pagination with composite cursor for ties
SELECT * FROM transfers
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

---

## 2. Connection Management (CRITICAL)

### Use Connection Pooling

- Always use Supabase's built-in Supavisor connection pooler
- Use port `6543` for pooled connections (not `5432` for direct)
- Set pool mode to `transaction` for serverless/Edge Functions
- Use `session` mode only when prepared statements or advisory locks are needed

### Supavisor vs PgBouncer

Supabase has migrated from PgBouncer to Supavisor:
- Supavisor: native Postgres protocol support, better connection management, built-in load balancing
- Same port (6543), same connection string format
- Transaction mode (default): connection returned to pool after each transaction
- Session mode: connection held for entire session (use sparingly)

### Edge Function Connection Patterns

```typescript
// GOOD: Use Supabase client (handles pooling automatically)
import { createClient } from '@supabase/supabase-js'
const supabase = createClient(url, key)

// BAD: Direct Postgres connections from Edge Functions
// (each invocation creates a new connection, exhausting pool)
import { Client } from 'pg'
const client = new Client(connectionString)  // DON'T DO THIS
```

### Connection Limits

| Tier | Direct Connections | Pooled (Supavisor) |
|------|-------------------|-------------------|
| Free | 60 | 200 |
| Pro | 200 | 1,500 |
| Team | 200 | 1,500 |

Always monitor connection usage:

```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';

-- Connections by state
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state;

-- Find long-running queries (potential connection leaks)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Kill a stuck query if needed
SELECT pg_terminate_backend(pid);
```

### Connection String Configuration

```
# Pooled connection (use this for app/Edge Functions)
postgresql://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres

# Direct connection (use for migrations only)
postgresql://postgres.[project-ref]:[password]@db.[project-ref].supabase.co:5432/postgres
```
