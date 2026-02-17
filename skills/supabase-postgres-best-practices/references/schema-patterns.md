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
  created_at timestamptz DEFAULT now() NOT NULL,
  updated_at timestamptz DEFAULT now() NOT NULL
);

-- BAD: Using float for money, varchar without constraints
CREATE TABLE transfers (
  id serial PRIMARY KEY,
  amount float,
  currency varchar(255)
);
```

### Use CHECK Constraints

```sql
-- Validate at the database level
ALTER TABLE transfers ADD CONSTRAINT positive_amount CHECK (amount > 0);
ALTER TABLE transfers ADD CONSTRAINT valid_status
  CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'cancelled'));
```

### Foreign Key Best Practices

```sql
-- Always specify ON DELETE behavior
ALTER TABLE transfers
ADD CONSTRAINT fk_sender
FOREIGN KEY (sender_id) REFERENCES users(id) ON DELETE RESTRICT;

-- Use RESTRICT for financial data (prevent accidental deletion)
-- Use CASCADE only for dependent data (e.g., notifications)
```

---

## 5. Concurrency & Locking (MEDIUM-HIGH)

### Avoid Long Transactions

```sql
-- BAD: Long transaction holding locks
BEGIN;
  UPDATE wallets SET balance = balance - 100 WHERE user_id = $1;
  -- ... long processing ...
  UPDATE wallets SET balance = balance + 100 WHERE user_id = $2;
COMMIT;

-- GOOD: Use advisory locks for complex operations
SELECT pg_advisory_xact_lock(hashtext($1::text));
UPDATE wallets SET balance = balance - 100 WHERE user_id = $1;
UPDATE wallets SET balance = balance + 100 WHERE user_id = $2;
```

### Use Row-Level Locking

```sql
-- GOOD: Lock only the rows needed
SELECT * FROM wallets WHERE user_id = $1 FOR UPDATE NOWAIT;
```
