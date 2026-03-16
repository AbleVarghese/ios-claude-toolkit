# Schema Design & Concurrency Patterns

Reference patterns for Supabase Postgres schema design and concurrency management.

## 4. Schema Design (HIGH)

### Use Appropriate Data Types

```sql
-- GOOD: Use UUID for primary keys
CREATE TABLE transfers (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  amount numeric(12,2) NOT NULL,       -- Exact decimal for money
  currency text NOT NULL CHECK (currency IN ('USD', 'CAD')),
  status text NOT NULL DEFAULT 'pending',
  metadata jsonb DEFAULT '{}'::jsonb,  -- Extensibility field
  created_at timestamptz DEFAULT now() NOT NULL,
  updated_at timestamptz DEFAULT now() NOT NULL,
  deleted_at timestamptz             -- Soft delete
);

-- BAD: Using float for money, varchar without constraints
CREATE TABLE transfers (
  id serial PRIMARY KEY,
  amount float,
  currency varchar(255)
);
```

### Why These Types Matter

| Column | Type | Why |
|--------|------|-----|
| Primary key | `uuid` | Prevents enumeration, merge-safe, no sequence contention |
| Money | `numeric(12,2)` | Exact decimal arithmetic, no floating point errors |
| Timestamps | `timestamptz` | Always stores UTC, converts to user's timezone on display |
| Status | `text` with CHECK | Validated at DB level, flexible for new values |
| Metadata | `jsonb` | Schema-free extensibility without migrations |
| Soft delete | `timestamptz NULL` | NULL = active, timestamp = when deleted |

### Use CHECK Constraints

```sql
-- Validate at the database level -- not just the application
ALTER TABLE transfers ADD CONSTRAINT positive_amount CHECK (amount > 0);
ALTER TABLE transfers ADD CONSTRAINT valid_status
  CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'cancelled'));
ALTER TABLE transfers ADD CONSTRAINT valid_currency
  CHECK (currency IN ('USD', 'CAD', 'EUR', 'GBP', 'MXN'));

-- Compound CHECK constraints
ALTER TABLE transfers ADD CONSTRAINT completed_requires_completed_at
  CHECK (
    (status = 'completed' AND completed_at IS NOT NULL) OR
    (status != 'completed')
  );
```

### Foreign Key Best Practices

```sql
-- Always specify ON DELETE behavior
ALTER TABLE transfers
ADD CONSTRAINT fk_sender
FOREIGN KEY (sender_id) REFERENCES users(id) ON DELETE RESTRICT;

-- Use RESTRICT for financial data (prevent accidental deletion)
-- Use CASCADE only for dependent data (e.g., notifications)
-- Use SET NULL when the reference is optional

-- GOOD: Index foreign key columns (Postgres doesn't auto-index these)
CREATE INDEX idx_transfers_sender_id ON transfers(sender_id);
CREATE INDEX idx_transfers_recipient_id ON transfers(recipient_id);
```

### Updated_at Trigger

```sql
-- Automatically update updated_at on any row modification
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
BEFORE UPDATE ON transfers
FOR EACH ROW
EXECUTE FUNCTION update_updated_at();
```

### Table Partitioning

For tables expected to exceed 10M+ rows, consider partitioning:

```sql
-- Range partition by date (most common for time-series data)
CREATE TABLE transfer_events (
  id uuid DEFAULT gen_random_uuid(),
  transfer_id uuid NOT NULL,
  event_type text NOT NULL,
  payload jsonb,
  created_at timestamptz DEFAULT now() NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE transfer_events_2026_01
  PARTITION OF transfer_events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE transfer_events_2026_02
  PARTITION OF transfer_events
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Automate partition creation with pg_partman extension (if available)
```

When to partition:
- Table exceeds 10M rows and growing
- Queries consistently filter by the partition key
- Old data needs to be archived or dropped efficiently
- NOT when the table is small or queries don't filter by partition key

### Logical Replication Considerations

When using Supabase Realtime or external replication:

```sql
-- Ensure tables have REPLICA IDENTITY set
-- Default (primary key) is fine for most cases
ALTER TABLE transfers REPLICA IDENTITY DEFAULT;

-- For tables without primary key (avoid this), use FULL
ALTER TABLE audit_log REPLICA IDENTITY FULL;

-- For tables with updates on non-PK columns, ensure PK is sufficient
-- to identify the row for the replication target
```

### Row Estimation for Capacity Planning

```sql
-- Estimated row count (faster than COUNT(*) on large tables)
SELECT relname AS table_name,
       reltuples::bigint AS estimated_rows,
       pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
  AND relkind = 'r'
ORDER BY reltuples DESC;

-- Actual row count (slow on large tables, use sparingly)
SELECT count(*) FROM transfers;

-- Dead tuple ratio (indicates need for VACUUM)
SELECT relname,
       n_live_tup,
       n_dead_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## 5. Concurrency & Locking (MEDIUM-HIGH)

### Avoid Long Transactions

```sql
-- BAD: Long transaction holding locks
BEGIN;
  UPDATE wallets SET balance = balance - 100 WHERE user_id = $1;
  -- ... long processing (API call, email, etc.) ...
  UPDATE wallets SET balance = balance + 100 WHERE user_id = $2;
COMMIT;

-- GOOD: Short transaction, external processing before/after
-- Step 1: Validate (no locks)
-- Step 2: Short transaction
BEGIN;
  SELECT pg_advisory_xact_lock(hashtext($1::text));
  UPDATE wallets SET balance = balance - 100 WHERE user_id = $1;
  UPDATE wallets SET balance = balance + 100 WHERE user_id = $2;
  INSERT INTO transfer_log (...) VALUES (...);
COMMIT;
-- Step 3: Post-processing (notifications, etc.) -- no locks needed
```

### Use Row-Level Locking

```sql
-- GOOD: Lock only the rows needed
SELECT * FROM wallets WHERE user_id = $1 FOR UPDATE NOWAIT;
-- NOWAIT returns error immediately if row is locked (instead of waiting)

-- GOOD: Skip locked rows (for queue processing)
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Returns next unlocked row, prevents duplicate processing

-- GOOD: FOR UPDATE with specific tables in a JOIN
SELECT t.*, w.balance
FROM transfers t
JOIN wallets w ON w.user_id = t.sender_id
WHERE t.id = $1
FOR UPDATE OF w;  -- Only lock the wallet row, not the transfer
```

### Advisory Locks for Application-Level Coordination

```sql
-- Transaction-level advisory lock (released at COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(hashtext('process_transfer_' || $1::text));

-- Session-level advisory lock (must be explicitly released)
SELECT pg_advisory_lock(hashtext('singleton_job'));
-- ... do work ...
SELECT pg_advisory_unlock(hashtext('singleton_job'));

-- Try lock (non-blocking, returns boolean)
SELECT pg_try_advisory_lock(hashtext('batch_process'));
-- Returns true if acquired, false if already locked
```

### Deadlock Prevention

```sql
-- RULE: Always lock resources in the same order
-- BAD: Two transactions locking in opposite order causes deadlock
-- TX1: Lock user A, then user B
-- TX2: Lock user B, then user A

-- GOOD: Always lock in ID order
BEGIN;
  -- Sort user IDs and lock in order
  SELECT * FROM wallets
  WHERE user_id IN ($1, $2)
  ORDER BY user_id  -- Consistent order prevents deadlock
  FOR UPDATE;

  UPDATE wallets SET balance = balance - 100 WHERE user_id = $1;
  UPDATE wallets SET balance = balance + 100 WHERE user_id = $2;
COMMIT;
```

### Monitoring Locks

```sql
-- Find blocked queries
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
  AND kl.database IS NOT DISTINCT FROM bl.database
  AND kl.relation IS NOT DISTINCT FROM bl.relation
  AND kl.page IS NOT DISTINCT FROM bl.page
  AND kl.tuple IS NOT DISTINCT FROM bl.tuple
  AND kl.virtualxid IS NOT DISTINCT FROM bl.virtualxid
  AND kl.transactionid IS NOT DISTINCT FROM bl.transactionid
  AND kl.classid IS NOT DISTINCT FROM bl.classid
  AND kl.objid IS NOT DISTINCT FROM bl.objid
  AND kl.objsubid IS NOT DISTINCT FROM bl.objsubid
  AND kl.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE NOT bl.granted;
```
