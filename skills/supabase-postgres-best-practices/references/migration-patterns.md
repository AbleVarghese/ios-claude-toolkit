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
