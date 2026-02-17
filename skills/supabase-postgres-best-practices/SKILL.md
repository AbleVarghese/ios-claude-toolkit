---
name: supabase-postgres-best-practices
description: "This skill should be used when the user asks to 'optimize a query', 'design a schema', 'add an index', 'write an RLS policy', 'create a migration', 'connection pooling', or mentions Postgres performance. Provides Supabase Postgres best practices covering query optimization, RLS policies, connection pooling, indexing, migrations, and Edge Functions patterns."
user-invocable: false
license: MIT
metadata:
  author: supabase
  version: "1.1.0"
  organization: Supabase
  date: January 2026
  abstract: >
    Comprehensive Postgres performance optimization guide for developers using Supabase
    and Postgres. Contains performance rules across 8 categories, prioritized by impact
    from critical (query performance, connection management) to incremental (advanced
    features). Each rule includes detailed explanations, incorrect vs. correct SQL
    examples, query plan analysis, and specific performance metrics.
allowed-tools:
  - Read
  - Edit
  - Grep
  - Glob
  - Bash
---

# Supabase Postgres Best Practices

Comprehensive performance optimization guide for Postgres, maintained by Supabase. Contains rules across 8 categories, prioritized by impact to guide automated query optimization and schema design.

Source: [github.com/supabase/agent-skills](https://github.com/supabase/agent-skills)

## When to Apply

Reference these guidelines when:
- Writing SQL queries or designing schemas
- Implementing indexes or query optimization
- Reviewing database performance issues
- Configuring connection pooling or scaling
- Optimizing for Postgres-specific features
- Working with Row-Level Security (RLS)
- Creating Supabase migrations
- Writing Edge Functions

## Rule Categories by Priority

| Priority | Category | Impact | Prefix | Reference |
|----------|----------|--------|--------|-----------|
| 1 | Query Performance | CRITICAL | `query-` | `references/query-patterns.md` |
| 2 | Connection Management | CRITICAL | `conn-` | `references/query-patterns.md` |
| 3 | Security & RLS | CRITICAL | `security-` | Below |
| 4 | Schema Design | HIGH | `schema-` | `references/schema-patterns.md` |
| 5 | Concurrency & Locking | MEDIUM-HIGH | `lock-` | `references/schema-patterns.md` |
| 6 | Migration Best Practices | MEDIUM | `data-` | `references/migration-patterns.md` |
| 7 | Monitoring & Diagnostics | LOW-MEDIUM | `monitor-` | Below |
| 8 | Edge Functions Patterns | LOW | `advanced-` | Below |

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

---

## 7. Monitoring & Diagnostics (LOW-MEDIUM)

### Key Queries to Monitor

```sql
-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Index usage
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Cache hit ratio (should be > 99%)
SELECT
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit) as heap_hit,
  sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) as ratio
FROM pg_statio_user_tables;
```

---

## 8. Edge Functions Patterns

### Database Access from Edge Functions

```typescript
// GOOD: Use Supabase client with proper error handling
const { data, error } = await supabase
  .from('transfers')
  .select('id, amount, status')
  .eq('sender_id', userId)
  .order('created_at', { ascending: false })
  .limit(20);

if (error) throw new Error(`Database error: ${error.message}`);
```

### Webhook Processing

```typescript
// GOOD: Verify webhook signatures, process idempotently
const signature = req.headers.get('stripe-signature');
const event = await stripe.webhooks.constructEventAsync(
  body, signature, webhookSecret
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

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [Supabase Documentation](https://supabase.com/docs)
- [PostgreSQL Performance Wiki](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Supabase Database Guide](https://supabase.com/docs/guides/database/overview)
- [Supabase RLS Guide](https://supabase.com/docs/guides/auth/row-level-security)
- `references/query-patterns.md` -- Query Performance + Connection Management
- `references/schema-patterns.md` -- Schema Design + Concurrency
- `references/migration-patterns.md` -- Migration Best Practices
