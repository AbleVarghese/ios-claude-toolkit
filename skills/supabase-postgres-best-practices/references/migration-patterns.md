# Migration Best Practices

Reference patterns for safe Supabase Postgres migrations.

## 6. Migration Best Practices

### Safe Migration Patterns

```sql
-- GOOD: Add column with default (non-blocking in PG 11+)
ALTER TABLE transfers ADD COLUMN fee_amount numeric(12,2) DEFAULT 0;

-- BAD: Adding NOT NULL without default on existing table (locks table)
ALTER TABLE transfers ADD COLUMN fee_amount numeric(12,2) NOT NULL;

-- GOOD: Create index concurrently (non-blocking)
CREATE INDEX CONCURRENTLY idx_transfers_status ON transfers(status);

-- BAD: Regular index creation (locks table for writes)
CREATE INDEX idx_transfers_status ON transfers(status);
```

### Migration Naming Convention

```
YYYYMMDDHHMMSS_descriptive_name.sql
-- Example: 20260201120000_add_transfer_fee_columns.sql
```

### Always Include Rollback

```sql
-- Up migration
ALTER TABLE transfers ADD COLUMN fee_breakdown jsonb;

-- Down migration (in separate file or comment)
-- ALTER TABLE transfers DROP COLUMN fee_breakdown;
```

---

## Zero-Downtime Migration Patterns

### Pattern 1: Add Column (Safe)

Adding a column with a default value does NOT lock the table in Postgres 11+. The default is stored in the catalog, not written to every row.

```sql
-- Safe: No table lock, no rewrite
ALTER TABLE transfers ADD COLUMN fee_type text DEFAULT 'standard';

-- Later, if you need NOT NULL:
-- Step 1: Add column with default (instant)
ALTER TABLE transfers ADD COLUMN new_col text DEFAULT 'value';
-- Step 2: Backfill in batches (no lock)
UPDATE transfers SET new_col = 'computed_value' WHERE new_col = 'value' AND id < $batch_end;
-- Step 3: Add NOT NULL constraint (validates existing rows, brief lock)
ALTER TABLE transfers ALTER COLUMN new_col SET NOT NULL;
```

### Pattern 2: Rename Column (DANGEROUS)

Renaming a column instantly breaks all queries referencing the old name. Use the expand/contract pattern instead.

```sql
-- NEVER in production:
ALTER TABLE transfers RENAME COLUMN old_name TO new_name;

-- SAFE: Expand/Contract pattern
-- Step 1: Add new column
ALTER TABLE transfers ADD COLUMN new_name text;

-- Step 2: Backfill new column
UPDATE transfers SET new_name = old_name WHERE new_name IS NULL;
-- (Do in batches for large tables)

-- Step 3: Add trigger to keep both in sync during transition
CREATE OR REPLACE FUNCTION sync_old_to_new()
RETURNS trigger AS $$
BEGIN
  NEW.new_name = COALESCE(NEW.new_name, NEW.old_name);
  NEW.old_name = NEW.new_name;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_columns
BEFORE INSERT OR UPDATE ON transfers
FOR EACH ROW EXECUTE FUNCTION sync_old_to_new();

-- Step 4: Deploy code that reads from new_name
-- Step 5: Deploy code that writes to new_name
-- Step 6: Drop trigger and old column (after confirming no reads)
DROP TRIGGER sync_columns ON transfers;
ALTER TABLE transfers DROP COLUMN old_name;
```

### Pattern 3: Change Column Type (DANGEROUS)

Changing a column type rewrites the entire table, holding an exclusive lock.

```sql
-- NEVER for large tables:
ALTER TABLE transfers ALTER COLUMN amount TYPE numeric(16,4);

-- SAFE: Add new column, migrate, swap
-- Step 1: Add new column with desired type
ALTER TABLE transfers ADD COLUMN amount_v2 numeric(16,4);

-- Step 2: Backfill in batches
DO $$
DECLARE
  batch_size int := 10000;
  last_id uuid := '00000000-0000-0000-0000-000000000000';
BEGIN
  LOOP
    UPDATE transfers
    SET amount_v2 = amount::numeric(16,4)
    WHERE id > last_id
      AND amount_v2 IS NULL
    ORDER BY id
    LIMIT batch_size
    RETURNING id INTO last_id;

    EXIT WHEN NOT FOUND;
    COMMIT;  -- Release locks between batches
  END LOOP;
END $$;

-- Step 3: Add trigger to keep in sync
-- Step 4: Switch application to read amount_v2
-- Step 5: Switch application to write amount_v2
-- Step 6: Drop old column
```

### Pattern 4: Add Index Concurrently

```sql
-- ALWAYS use CONCURRENTLY for indexes on tables with traffic
CREATE INDEX CONCURRENTLY idx_transfers_recipient
ON transfers(recipient_id);

-- IMPORTANT: CONCURRENTLY cannot run inside a transaction
-- Supabase migrations run in transactions by default
-- You may need to run this manually or configure the migration tool

-- If CONCURRENTLY fails partway through, the index is left INVALID:
-- Check for invalid indexes:
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
JOIN pg_index ON pg_stat_user_indexes.indexrelid = pg_index.indexrelid
WHERE NOT pg_index.indisvalid;

-- Fix: Drop the invalid index and retry
DROP INDEX CONCURRENTLY idx_transfers_recipient;
CREATE INDEX CONCURRENTLY idx_transfers_recipient ON transfers(recipient_id);
```

### Pattern 5: Add NOT NULL Constraint

Adding NOT NULL validates all existing rows, which acquires a brief ACCESS EXCLUSIVE lock.

```sql
-- For small tables (< 100K rows): direct ALTER is fine
ALTER TABLE transfers ALTER COLUMN sender_id SET NOT NULL;

-- For large tables: use CHECK constraint first (validates without long lock)
-- Step 1: Add CHECK constraint NOT VALID (instant, no validation)
ALTER TABLE transfers ADD CONSTRAINT sender_id_not_null
  CHECK (sender_id IS NOT NULL) NOT VALID;

-- Step 2: Validate the constraint (reads all rows but doesn't lock writes)
ALTER TABLE transfers VALIDATE CONSTRAINT sender_id_not_null;

-- Step 3: Convert to real NOT NULL (Postgres recognizes the CHECK)
ALTER TABLE transfers ALTER COLUMN sender_id SET NOT NULL;

-- Step 4: Drop the now-redundant CHECK constraint
ALTER TABLE transfers DROP CONSTRAINT sender_id_not_null;
```

### Pattern 6: Drop Column (Safe with Caution)

```sql
-- Dropping a column is instant in Postgres (marks as invisible, doesn't rewrite)
-- But it acquires a brief ACCESS EXCLUSIVE lock

-- Safe approach:
-- Step 1: Stop writing to the column (deploy code change)
-- Step 2: Drop the column
ALTER TABLE transfers DROP COLUMN IF EXISTS legacy_field;

-- For very high-traffic tables, use a maintenance window
```

### Pattern 7: Add Unique Constraint

```sql
-- BAD: Direct unique constraint (locks table for validation)
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

-- GOOD: Create unique index concurrently first
CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_unique ON users(email);

-- Then add the constraint using the existing index (instant)
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE USING INDEX idx_users_email_unique;
```

---

## Migration Safety Checklist

Before applying any migration to production:

```
[ ] Migration tested against a copy of production data
[ ] Rollback script written and tested
[ ] No ALTER TABLE that rewrites the table (type change, ADD COLUMN NOT NULL without default)
[ ] All CREATE INDEX uses CONCURRENTLY
[ ] Large data updates done in batches (not single UPDATE)
[ ] Migration time estimated (test on production-size dataset)
[ ] Maintenance window scheduled if migration takes > 5 seconds
[ ] Application code compatible with both old and new schema (for expand/contract)
[ ] No DROP COLUMN until all code references removed and deployed
[ ] Foreign key constraints use NOT VALID + VALIDATE pattern for large tables
```

---

## Supabase-Specific Migration Notes

### Using Supabase CLI

```bash
# Create a new migration
supabase migration new add_transfer_fees

# Apply migrations locally
supabase db reset  # Resets and replays all migrations

# Push to remote
supabase db push

# Pull remote schema changes (dashboard edits)
supabase db pull
```

### Migration Transaction Behavior

By default, Supabase migrations run inside a transaction. This means:
- `CREATE INDEX CONCURRENTLY` will fail (cannot run inside a transaction)
- If any statement fails, the entire migration is rolled back

For migrations that need CONCURRENTLY:
1. Split into a separate migration file
2. Add `-- migrate:no-transaction` comment (if supported by your migration tool)
3. Or run the index creation manually via the Supabase SQL Editor

### Seeding vs Migration

- **Migrations**: Schema changes (DDL). Always reversible. Run once per environment.
- **Seeds**: Initial data. Idempotent (use UPSERT). May differ per environment.
- Don't put data changes in migrations unless they're required for schema integrity (e.g., backfilling a new NOT NULL column).
