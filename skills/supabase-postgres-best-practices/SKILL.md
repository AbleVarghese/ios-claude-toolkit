---
name: supabase-postgres-best-practices
description: "Query is slow, optimize database, add an index, RLS policy. Postgres query optimization, schema design, indexing, RLS policy performance, and connection pooling best practices. Trigger on: 'slow query', 'EXPLAIN ANALYZE', 'add an index', 'RLS is slow', 'optimize this query', 'N+1 problem', 'connection pooling', 'Supavisor', 'SELECT * performance', 'database health', 'Edge Function database access', or any question about making Postgres faster. For creating/repairing/pushing migrations, use atlas-database instead."
user-invocable: false
allowed-tools:
  - Read
  - Edit
  - Grep
  - Glob
  - Bash
---

# Supabase Postgres Best Practices

> **Iron Law:** "Every query touching user data must go through RLS. Service role keys never reach the client."

Comprehensive performance optimization guide for Postgres, maintained by Supabase. Contains rules across 8 categories, prioritized by impact to guide automated query optimization and schema design.

Source: [github.com/supabase/agent-skills](https://github.com/supabase/agent-skills)

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|----------------|-----------------|
| "The query is fast enough" | "Fast enough" today is slow tomorrow. Database performance degrades non-linearly: a query that's 50ms at 10K rows becomes 5 seconds at 1M rows. By the time you notice, it's a production incident. | Run EXPLAIN ANALYZE on every query that touches a table expected to exceed 10K rows. Add indexes proactively. |
| "We don't need indexes yet" | Sequential scans on unindexed columns are the #1 cause of Postgres performance issues. Adding indexes retroactively on large tables locks the table for writes (unless you use CONCURRENTLY, which you probably won't remember under pressure). | Add indexes at table creation time for every column used in WHERE, JOIN, and ORDER BY clauses. |
| "RLS is too slow" | RLS adds 0.1-0.5ms per query when policies use direct column comparisons with `auth.uid()`. That's negligible. RLS being "slow" almost always means the policy uses a subquery or function call that triggers per-row evaluation. The fix is better policies, not disabling RLS. | Write efficient RLS policies using direct comparisons. Index the columns used in RLS policies. Never disable RLS on user-facing tables. |
| "We'll optimize when we scale" | Database optimization at scale is 10x harder because you can't easily run migrations, you can't test with production-scale data locally, and every change risks downtime. Build the right patterns now when they're cheap. | Follow the patterns in this guide from the start. Proper schema design, indexing, and RLS policies are not premature optimization -- they're baseline engineering. |
| "SELECT * is fine for our use case" | SELECT * fetches all columns including large JSONB blobs, text fields, and columns you added "just in case." It also breaks if you add a column with sensitive data later -- suddenly your API returns it. | Select only the columns you need. Create specific views for different access patterns. |
| "Service role key is safe in our Edge Function" | Edge Functions run server-side, so service_role is acceptable there. But if you're passing the service_role key from a client to an Edge Function, or if your Edge Function uses service_role when anon would suffice, you've expanded your attack surface unnecessarily. | Use anon key (RLS-enforced) for client-initiated operations. Use service_role only when explicitly bypassing RLS for admin operations in trusted server-side code. |

---

## Red Flags -- STOP

- **SELECT * in production queries**: Fetches unnecessary data, breaks when schema changes, may expose sensitive columns. Always specify columns.
- **Missing RLS on user-facing tables**: Any table accessible through the Supabase client (anon key) without RLS is a complete data breach. Enable RLS and add policies immediately.
- **N+1 query patterns**: Querying in a loop (for each user, fetch their transfers) instead of a single JOIN query. This turns 1 query into N+1 queries and destroys performance.
- **Sequential scans on large tables**: EXPLAIN ANALYZE showing `Seq Scan` on a table with >10K rows means you're scanning every row. Add an index.
- **Service role key in client code**: The service_role key bypasses ALL RLS. If it's in your iOS app bundle, JavaScript, or any client-accessible code, every user can access every row in every table.
- **Missing connection pooling**: Direct connections from serverless functions exhaust the connection pool in minutes. Always use Supavisor (port 6543) for pooled connections.

---

## When NOT to Use This Skill

- **Migration file creation and management**: For creating, repairing, and pushing Supabase migrations, see **atlas-database**.
- **Stripe webhook data storage**: For designing webhook event storage with idempotent processing, see **stripe-best-practices** (but use schema patterns from this skill).
- **Application-level data access patterns**: This skill covers database-level optimization. For service/repository layer patterns in Swift, see **forge-development**.
- **Database backup and disaster recovery**: For backup strategies and point-in-time recovery, consult Supabase documentation directly.

---

## Decision Framework

### Query Optimization Decision Tree

```
Is the query slow (>100ms for simple, >500ms for complex)?
  |
  YES --> Run EXPLAIN ANALYZE
  |       |
  |       Is there a Seq Scan on a large table?
  |         YES --> Add index on the filtered/joined column
  |                 Use partial index if filtering on a subset (e.g., WHERE status = 'active')
  |         NO  --> Is there a Nested Loop Join on large datasets?
  |                   YES --> Verify JOIN columns are indexed
  |                           Consider using a CTE or subquery to reduce the join input
  |                   NO  --> Is actual rows >> estimated rows?
  |                             YES --> Run ANALYZE on the table to update statistics
  |                             NO  --> Check for unnecessary columns (SELECT *)
  |                                     Check for missing LIMIT on unbounded queries
  |
  NO --> Is it a frequently executed query?
          YES --> Monitor over time. Small queries add up.
          NO  --> No action needed.
```

### Index Selection Decision Tree

```
What column am I indexing?

Equality comparison (WHERE col = value)?
  --> B-tree index (default): CREATE INDEX idx ON table(col);

Range comparison (WHERE col > value, BETWEEN)?
  --> B-tree index: CREATE INDEX idx ON table(col);

Full-text search (WHERE col @@ to_tsquery(...))?
  --> GIN index: CREATE INDEX idx ON table USING gin(col);

JSONB containment (WHERE col @> '{"key": "value"}')?
  --> GIN index: CREATE INDEX idx ON table USING gin(col);

JSONB specific path (WHERE col->>'key' = 'value')?
  --> B-tree on expression: CREATE INDEX idx ON table((col->>'key'));

Geometric / spatial queries?
  --> GiST index: CREATE INDEX idx ON table USING gist(col);

Array containment (WHERE col @> ARRAY[...])?
  --> GIN index: CREATE INDEX idx ON table USING gin(col);

Only need index for a subset of rows?
  --> Partial index: CREATE INDEX idx ON table(col) WHERE condition;
```

---

## EXPLAIN ANALYZE Reading Guide

Understanding query plans is the single most important database optimization skill.

### Running EXPLAIN ANALYZE

```sql
-- Always use ANALYZE (executes the query) for accurate timing
-- Use BUFFERS for I/O information
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT t.id, t.amount, u.full_name
FROM transfers t
JOIN users u ON t.sender_id = u.id
WHERE t.status = 'completed'
  AND t.created_at > now() - interval '30 days'
ORDER BY t.created_at DESC
LIMIT 20;
```

### Reading the Output

```
Limit  (cost=0.86..45.32 rows=20 width=48) (actual time=0.128..0.342 rows=20 loops=1)
  ->  Nested Loop  (cost=0.86..1234.56 rows=555 width=48) (actual time=0.126..0.338 rows=20 loops=1)
        ->  Index Scan Backward using idx_transfers_created_at on transfers t
              (cost=0.43..789.12 rows=555 width=40) (actual time=0.089..0.198 rows=20 loops=1)
              Filter: (status = 'completed')
              Rows Removed by Filter: 5
        ->  Index Scan using users_pkey on users u
              (cost=0.43..0.80 rows=1 width=16) (actual time=0.006..0.006 rows=1 loops=20)
              Index Cond: (id = t.sender_id)
Planning Time: 0.245 ms
Execution Time: 0.389 ms
Buffers: shared hit=65
```

### What Each Field Means

| Field | Meaning | What to Look For |
|-------|---------|-----------------|
| `cost=X..Y` | Estimated startup..total cost (arbitrary units) | Higher = more expensive |
| `rows=N` | Estimated number of rows output | Compare with `actual rows` |
| `actual time=X..Y` | Real execution time in ms | Total time of operation |
| `actual rows=N` | Real rows returned | If >> estimated, run ANALYZE |
| `loops=N` | How many times this node executed | High loops in nested loops = expensive |
| `Buffers: shared hit=N` | Pages read from cache | Higher = more I/O |
| `Buffers: shared read=N` | Pages read from disk | High = needs more cache or index |

### Warning Signs in EXPLAIN ANALYZE

| Sign | Problem | Solution |
|------|---------|----------|
| `Seq Scan` on large table | Full table scan, no index used | Add index on filtered column |
| `actual rows` >> `estimated rows` | Stale statistics | Run `ANALYZE table_name;` |
| `Nested Loop` with high `loops` | N+1 join pattern | Add index on JOIN column |
| `Sort` with `external merge` | Sort spilling to disk | Add index matching ORDER BY, or increase `work_mem` |
| `Hash Join` with high memory | Large hash table in memory | May need more `work_mem` or different join strategy |
| `Filter: Rows Removed` (large) | Index fetch then discard | Use partial index or refine WHERE clause |

---

## Rule Categories by Priority

| Priority | Category | Impact | Reference |
|----------|----------|--------|-----------|
| 1 | Query Performance | CRITICAL | `references/query-patterns.md` |
| 2 | Connection Management | CRITICAL | `references/query-patterns.md` |
| 3 | Security & RLS | CRITICAL | Below |
| 4 | Schema Design | HIGH | `references/schema-patterns.md` |
| 5 | Concurrency & Locking | MEDIUM-HIGH | `references/schema-patterns.md` |
| 6 | Migration Best Practices | MEDIUM | `references/migration-patterns.md` |
| 7 | Monitoring & Diagnostics | LOW-MEDIUM | Below |
| 8 | Edge Functions Patterns | LOW | Below |

---

## 3. Security & RLS (CRITICAL)

### Always Enable RLS

```sql
-- MANDATORY for all user-facing tables
ALTER TABLE transfers ENABLE ROW LEVEL SECURITY;

-- Deny all access by default
-- Then add specific policies
```

### Write Efficient RLS Policies

```sql
-- BAD: Subquery in policy (executes per row)
CREATE POLICY "Users see own transfers" ON transfers
FOR SELECT USING (
  sender_id IN (SELECT id FROM users WHERE auth.uid() = id)
);

-- GOOD: Direct comparison (uses index)
CREATE POLICY "Users see own transfers" ON transfers
FOR SELECT USING (sender_id = auth.uid());
```

### RLS Policy Performance Patterns

```sql
-- Pattern 1: Direct uid comparison (fastest)
CREATE POLICY "own_data" ON transfers
FOR ALL USING (user_id = auth.uid());

-- Pattern 2: Role-based with JWT claims (fast)
CREATE POLICY "admin_access" ON transfers
FOR ALL USING (
  auth.jwt()->>'role' = 'admin'
  OR user_id = auth.uid()
);

-- Pattern 3: Team-based with indexed lookup (acceptable)
-- REQUIRES: CREATE INDEX idx_team_members_user ON team_members(user_id);
CREATE POLICY "team_access" ON documents
FOR SELECT USING (
  team_id IN (
    SELECT team_id FROM team_members WHERE user_id = auth.uid()
  )
);
-- This subquery is OK because it filters a small set per user

-- Pattern 4: AVOID -- function call per row
CREATE POLICY "complex_access" ON transfers
FOR SELECT USING (
  has_access(auth.uid(), id)  -- Function called for EVERY row
);
```

### Use Security Definer Functions Carefully

```sql
-- Use security definer for admin operations that bypass RLS
CREATE OR REPLACE FUNCTION admin_get_transfer(transfer_id uuid)
RETURNS transfers
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  -- Always validate inputs in security definer functions
  IF transfer_id IS NULL THEN
    RAISE EXCEPTION 'transfer_id cannot be null';
  END IF;

  RETURN (SELECT * FROM transfers WHERE id = transfer_id);
END;
$$;
```

### Service Role Key Safety

- Never expose the `service_role` key to the client
- Use `anon` key for client-side operations (respects RLS)
- Use `service_role` only in server-side code (Edge Functions, webhooks)
- In Edge Functions, prefer using the user's JWT to call Supabase (RLS-enforced) rather than service_role when possible

---

## 7. Monitoring & Diagnostics (LOW-MEDIUM)

### Key Queries to Monitor

```sql
-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Index usage (find unused indexes)
SELECT
  schemaname, tablename, indexname,
  idx_scan AS times_used,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Cache hit ratio (should be > 99%)
SELECT
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit) as heap_hit,
  ROUND(sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0)::numeric, 4) as ratio
FROM pg_statio_user_tables;

-- Slow queries (requires pg_stat_statements extension)
SELECT
  query,
  calls,
  ROUND(mean_exec_time::numeric, 2) AS avg_ms,
  ROUND(total_exec_time::numeric, 2) AS total_ms,
  rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Active connections by state
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state;

-- Long-running queries (>5 seconds)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - pg_stat_activity.query_start > interval '5 seconds'
ORDER BY duration DESC;
```

---

## 8. Edge Functions Patterns

### Database Access from Edge Functions

```typescript
// GOOD: Use Supabase client with proper error handling
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,  // Use anon key, not service_role
    {
        global: {
            headers: { Authorization: req.headers.get('Authorization')! }
        }
    }
);

const { data, error } = await supabase
    .from('transfers')
    .select('id, amount, status, created_at')  // Never SELECT *
    .eq('sender_id', userId)
    .order('created_at', { ascending: false })
    .limit(20);

if (error) throw new Error(`Database error: ${error.message}`);
```

### Webhook Processing

```typescript
// GOOD: Verify webhook signatures, process idempotently
import Stripe from 'stripe';

const signature = req.headers.get('stripe-signature');
const event = await stripe.webhooks.constructEventAsync(
    body, signature!, webhookSecret
);

// Use upsert for idempotent processing
const { error } = await supabase
    .from('webhook_events')
    .upsert({
        event_id: event.id,
        type: event.type,
        processed_at: new Date().toISOString()
    }, { onConflict: 'event_id' });
```

---

## Phase Gate: QUERY REVIEW ✓

**Before deploying any new query:**
- [ ] EXPLAIN ANALYZE run on representative data
- [ ] No sequential scans on tables > 10,000 rows (add index)
- [ ] No N+1 patterns (one query per parent row)
- [ ] RLS policies verified for the query's access pattern

**Hard Stop**: `SELECT *` in production code — STOP. Select only the columns you need. SELECT * sends unnecessary data over the network, prevents index-only scans, and breaks if columns are added.

### Phase Gate: RLS VERIFICATION ✓

**Cannot deploy table without:**
- [ ] RLS enabled: `ALTER TABLE x ENABLE ROW LEVEL SECURITY`
- [ ] Policies tested with anon role (should see nothing or only public data)
- [ ] Policies tested with authenticated role (should see only own data)
- [ ] Service role key NOT used in client-side code

**Hard Stop**: A table without RLS is accessible to anyone with the anon key. This is a data breach waiting to happen. Enable RLS on EVERY user-facing table.

### Self-Audit

1. Did I run EXPLAIN ANALYZE on my queries?
2. Is RLS enabled on every table I modified?
3. Is the service role key absent from all client-side code?
4. Are there any N+1 query patterns in my implementation?

---

## Quality Gates (Before Marking Complete)

1. RLS enabled on all user-facing tables with policies tested
2. No SELECT * in production queries -- all queries specify columns
3. Indexes exist on all columns used in WHERE, JOIN, ORDER BY
4. EXPLAIN ANALYZE run on all queries touching tables > 10K rows
5. Connection pooling configured (Supavisor port 6543, not direct port 5432)
6. Service role key not present in any client-side code
7. All RLS policies use direct comparisons or indexed subqueries (no per-row function calls)
8. N+1 queries replaced with JOINs or batch queries
9. Migrations tested for zero-downtime (no table locks in production)
10. Cache hit ratio verified > 99%

---

## Cross-Skill References

- **atlas-database**: Creating, repairing, and pushing Supabase migrations.
- **stripe-best-practices**: Webhook event storage patterns and idempotent processing in Postgres.
- **cipher-security**: Database security review, encryption at rest, and secure connection configuration.
- **forge-development**: Application-level repository/service patterns for database access in Swift.

---

## References

- Query patterns and connection management: `references/query-patterns.md`
- Schema design and concurrency: `references/schema-patterns.md`
- Migration best practices: `references/migration-patterns.md`
- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [Supabase Documentation](https://supabase.com/docs)
- [Supabase RLS Guide](https://supabase.com/docs/guides/auth/row-level-security)
- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/using-explain.html)
