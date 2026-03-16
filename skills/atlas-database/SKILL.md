---
name: atlas-database
description: "Add a table, change the schema, database migration, Supabase schema. PostgreSQL/Supabase migration operations — creating, repairing, and pushing migrations, designing schemas, writing RLS policies, trigger functions. Trigger on: 'add a column', 'new database table', 'write a migration', 'fix migration history', 'RLS policy', 'enable row level security', 'trigger not working', or any hands-on schema change. For query optimization, indexing, and connection pooling, use supabase-postgres-best-practices instead."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: true
---

# ATLAS - Database Migration & Schema

PostgreSQL/Supabase database skill covering migration creation and repair, schema design, Row Level Security policies, and trigger function development with proper column qualification.

> **Iron Law**: "Never run a migration in production without testing it against a copy of production data first."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). In particular, locate the Supabase project directory from the project structure (commonly `backend/supabase/`, `supabase/`, or `Backend/supabase/`). This directory is referred to as `{SUPABASE_DIR}` throughout this skill.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|---|---|---|
| "I'll just add the column directly in production" | Direct DDL changes bypass migration history, making environments impossible to reconcile. The next `db push` will fail or overwrite. | Always create a migration file, even for a single column addition. |
| "This migration is simple enough to skip testing" | Simple migrations cause the worst outages because nobody expects them to fail. An `ALTER COLUMN` on a 10M row table locks the table for minutes. | Run against a copy of production data. Measure lock duration. |
| "We can always rollback" | Not all migrations are reversible. Data type changes, column drops, and data backfills destroy data on rollback. | Write explicit rollback SQL. Test the rollback. Document what data is lost if rolled back. |
| "RLS is too complex, we'll handle auth in the app" | Application-level auth is one bug away from exposing all data. RLS is the last line of defense — not the first. It catches the bugs your app layer misses. | Implement RLS policies for every table with user data. Test with both authenticated and anon roles. |
| "Triggers are fine without NEW/OLD qualification" | Unqualified column references in triggers silently resolve to the wrong value or fail entirely. This is the #1 trigger bug in PostgreSQL. | Always use `NEW.column_name` or `OLD.column_name`. Never bare `column_name`. |
| "I'll just use a text column for the status" | Text columns accept any value. Typos, casing differences, and invalid states propagate silently. Debugging "actve" vs "active" at 2am is not fun. | Use CHECK constraints, PostgreSQL ENUMs, or lookup tables with foreign keys. |
| "We don't need indexes yet, the table is small" | Tables grow. By the time you notice the slow query, it's affecting users. Adding indexes to large tables locks them. | Add indexes at table creation time for columns used in WHERE, JOIN, and ORDER BY. |
| "I'll store this as JSON, it's more flexible" | JSONB is untyped, unvalidated, and invisible to the schema. You lose constraint enforcement, foreign keys, and query planning. | Use JSONB only for genuinely unstructured data. If you query a field 3+ times, promote it to a column. |

---

## Red Flags -- STOP

- **ALTER COLUMN on a production table without testing lock duration** -- `ALTER TYPE`, `ALTER ... SET NOT NULL`, and `ALTER ... SET DEFAULT` can lock large tables for minutes. Test against production-size data first.
- **UPDATE/DELETE without a WHERE clause** -- This modifies or destroys every row. Always include WHERE. Always run SELECT with the same WHERE first to verify scope.
- **Trigger function without NEW/OLD qualification on every column reference** -- Bare column names cause silent resolution failures. Every. Single. Reference. Must. Be. Qualified.
- **RLS policy with a correlated subquery per row** -- This executes N subqueries for N rows. Use JOINs or `EXISTS` instead. If performance is unacceptable, consider a materialized security lookup table.
- **Migration that drops a column without a prior data migration** -- Column drops are irreversible. First deploy code that stops reading the column. Then backfill data elsewhere. Then drop.
- **Deploying migrations and application code simultaneously** -- If the migration fails, the new app code talks to the old schema. Deploy migrations separately, verify, then deploy the app.

---

## Behavioral Enforcement

### Phase Gate: MIGRATION SAFETY
**Cannot push migration to production until:**
- [ ] Migration tested against copy of production data (not just empty schema)
- [ ] Rollback procedure documented and tested
- [ ] RLS policies tested with BOTH authenticated and service_role
- [ ] All triggers verified with proper NEW/OLD qualification

**Hard Stop**: Running untested migrations in production risks data loss. There is no undo for a bad migration that corrupts data. Test first. Always.

### Phase Gate: SCHEMA CHANGE
**Cannot create migration until:**
- [ ] Existing schema reviewed (don't duplicate tables/columns that exist)
- [ ] Impact on existing RLS policies assessed
- [ ] Impact on existing triggers assessed

### Fix Attempt Tracking
- Migration fails #1: Read the error. Fix the SQL.
- Migration fails #2: Check constraints, triggers, RLS. Something is blocking.
- Migration fails #3: STOP. `supabase db reset` locally, review the full migration chain. The issue is cumulative.

### Self-Audit
1. Did I test this migration on a copy before pushing to production?
2. Can I rollback this migration if it breaks something?
3. Did I verify RLS policies still work after the schema change?
4. Are all trigger column references qualified with NEW/OLD?

### Required Output Artifact
Every migration operation must produce:
- The migration SQL file with header comment (purpose, date, rollback procedure)
- Verification output from `supabase db reset` (local) or `supabase db push --dry-run`
- RLS policy test results (authenticated role, anon role, service_role)

---

## When NOT to Use This Skill

1. **Query optimization and performance tuning** -- Use the `supabase-postgres-best-practices` skill. This skill creates schemas; that skill optimizes them.
2. **Application-level data access patterns** -- This skill handles database DDL and policies. For repository patterns, data access layers, and ORM configuration, work in the application codebase directly.
3. **One-off data analysis or reporting queries** -- If you're exploring data (not changing schema), use the database client directly. Migrations are for schema changes, not ad-hoc queries.
4. **Supabase Edge Functions or API configuration** -- Those are application-layer concerns, not database schema concerns.

---

## Decision Framework

```
Need to change the database?
|
+-- Adding new functionality?
|   |
|   +-- New table needed? --> SOP-1 (Create Migration)
|   |   Include: table, indexes, RLS, grants
|   |
|   +-- New column on existing table? --> SOP-1
|   |   Use: ADD COLUMN with DEFAULT (never ALTER existing)
|   |
|   +-- Need computed/derived data? --> Consider trigger (SOP-3)
|       vs. application-level computation
|       Rule: If 3+ consumers need it, use a trigger
|
+-- Fixing a problem?
|   |
|   +-- Migration history mismatch? --> SOP-2 (Repair)
|   |
|   +-- Trigger not firing/wrong values? --> SOP-3 (Debug)
|   |   First check: NEW/OLD qualification
|   |
|   +-- RLS blocking legitimate access? --> Check policy WITH CHECK
|       vs. USING clauses. Test with the correct role.
|
+-- Security change?
    |
    +-- Table has user data? --> Write RLS policy
    |   Pattern: ownership-based, role-based, or attribute-based
    |
    +-- Need row-level audit? --> Add trigger for audit log
        Pattern: AFTER INSERT/UPDATE/DELETE --> audit table
```

---

## Standard Operating Procedures

### SOP-1: Create Migration

```bash
# 1. Generate migration file
TIMESTAMP=$(date +%Y%m%d%H%M%S)
MIGRATION_FILE="{SUPABASE_DIR}/migrations/${TIMESTAMP}_description.sql"

# 2. Write migration SQL with proper structure
cat > $MIGRATION_FILE << 'EOF'
-- ============================================================================
-- MIGRATION: [Description]
-- Purpose: [Why this migration exists]
-- Date: [YYYY-MM-DD]
-- Rollback: [How to reverse this migration]
-- ============================================================================

-- PART 1: Schema Changes
-- Always ADD columns, never ALTER existing ones in production
-- Use defaults so existing rows are valid

-- PART 2: Indexes
-- Add indexes for columns used in WHERE, JOIN, ORDER BY
-- Use partial indexes where appropriate: CREATE INDEX ... WHERE deleted_at IS NULL

-- PART 3: Constraints
-- CHECK constraints for value validation
-- UNIQUE constraints (consider partial unique for soft-delete)

-- PART 4: RLS Policies
-- Every table with user data gets RLS
-- Test with: SET ROLE authenticated; SET request.jwt.claims = '{"sub":"user-uuid"}';

-- PART 5: Triggers/Functions
-- Always qualify with NEW. or OLD.
-- Use SECURITY DEFINER only when the function needs elevated privileges

-- PART 6: Grants
-- GRANT SELECT, INSERT, UPDATE on tables to authenticated
-- GRANT USAGE on sequences
-- Never GRANT DELETE (use soft delete)

-- PART 7: Verification
-- SELECT to confirm schema looks right
-- Run a test INSERT/UPDATE to verify constraints and triggers
EOF

# 3. Test against local database first
cd {SUPABASE_DIR} && supabase db reset

# 4. Verify migration succeeded
supabase db push --dry-run

# 5. Push migration
supabase db push
```

**Why each part matters:**
- Schema changes without indexes = slow queries as data grows
- Missing RLS = data exposure vulnerability
- Missing grants = application cannot access the data
- Missing verification = silent failures discovered in production

### SOP-2: Repair Migration History

```bash
# DIAGNOSIS: First, understand the mismatch
cd {SUPABASE_DIR}

# List local migrations
ls -la migrations/

# List remote migration history
supabase migration list

# SCENARIO A: Remote has migrations not in local
# This happens when someone applied migrations directly or from another branch
supabase migration repair --status reverted [migration_ids...]
supabase db pull  # Sync local with remote state

# SCENARIO B: Local has migrations not applied to remote
# This happens when migrations were created but never pushed
supabase migration repair --status applied [migration_ids...]
supabase db push

# SCENARIO C: Complete desync -- nuclear option
# WARNING: This resets migration tracking, not the database itself
supabase db pull --schema public  # Pull current remote schema
# Review the pulled migration, then:
supabase db push
```

**Why repair is needed:**
Migration history is a linear sequence. When local and remote diverge (branch merges, manual changes, failed pushes), the history must be reconciled before any new migrations can be applied. Skipping repair leads to "migration already applied" errors or missed migrations.

### SOP-3: Debug Trigger Issues

```sql
-- STEP 1: Verify the trigger exists and is enabled
SELECT
    tgname AS trigger_name,
    tgrelid::regclass AS table_name,
    CASE tgenabled
        WHEN 'O' THEN 'enabled (origin)'
        WHEN 'D' THEN 'DISABLED'
        WHEN 'R' THEN 'enabled (replica)'
        WHEN 'A' THEN 'enabled (always)'
    END AS status,
    tgtype::int & 2 > 0 AS is_before,
    tgtype::int & 4 > 0 AS is_insert,
    tgtype::int & 8 > 0 AS is_update,
    tgtype::int & 16 > 0 AS is_delete,
    tgtype::int & 1 > 0 AS is_row_level
FROM pg_trigger
WHERE NOT tgisinternal
ORDER BY tgrelid::regclass, tgname;

-- STEP 2: Read the function source code
SELECT pg_get_functiondef(oid)
FROM pg_proc
WHERE proname = 'function_name';

-- STEP 3: Check for the #1 bug -- unqualified column references
-- In the function body, EVERY column reference must be:
--   NEW.column_name  (for the new row in INSERT/UPDATE triggers)
--   OLD.column_name  (for the old row in UPDATE/DELETE triggers)
--
-- WRONG: IF status = 'active' THEN
-- RIGHT: IF NEW.status = 'active' THEN
--
-- WRONG: updated_at := now();
-- RIGHT: NEW.updated_at := now();

-- STEP 4: Test the trigger in isolation
BEGIN;
  -- Insert/update a test row
  INSERT INTO table_name (columns...) VALUES (values...);
  -- Check the result
  SELECT * FROM table_name WHERE id = 'test-id';
  -- Check audit log or side effects
  SELECT * FROM audit_log ORDER BY created_at DESC LIMIT 5;
ROLLBACK;  -- Don't persist test data
```

### SOP-4: Design RLS Policies

```sql
-- Enable RLS on the table (required before any policies take effect)
ALTER TABLE public.items ENABLE ROW LEVEL SECURITY;

-- Force RLS for table owners too (prevents bypassing)
ALTER TABLE public.items FORCE ROW LEVEL SECURITY;

-- PATTERN 1: Ownership-based access
-- Users can only see and modify their own rows
CREATE POLICY "Users can view own items"
    ON public.items FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own items"
    ON public.items FOR INSERT
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own items"
    ON public.items FOR UPDATE
    USING (auth.uid() = user_id)        -- Which rows can be selected for update
    WITH CHECK (auth.uid() = user_id);  -- What the row must look like after update

-- PATTERN 2: Role-based access
-- Different access levels based on user role from JWT
CREATE POLICY "Admins can view all items"
    ON public.items FOR SELECT
    USING (
        (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
    );

-- PATTERN 3: Attribute-based access (shared resources)
-- Access based on membership in a group/team
CREATE POLICY "Team members can view team items"
    ON public.items FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM public.team_members tm
            WHERE tm.team_id = items.team_id
            AND tm.user_id = auth.uid()
            AND tm.deleted_at IS NULL
        )
    );

-- PATTERN 4: Service role bypass
-- service_role key bypasses RLS by default. To enforce:
-- GRANT on the table controls service_role access

-- TESTING: Always verify with the actual role
SET ROLE authenticated;
SET request.jwt.claims = '{"sub":"test-user-uuid","role":"authenticated"}';
SELECT * FROM public.items;  -- Should only show this user's items
RESET ROLE;
```

**Why USING vs WITH CHECK matters:**
- `USING` controls which existing rows are visible (SELECT, UPDATE, DELETE)
- `WITH CHECK` controls what new/modified rows must look like (INSERT, UPDATE)
- For UPDATE, both are needed: USING filters which rows to update, WITH CHECK validates the result
- Missing WITH CHECK on UPDATE = users can reassign their rows to other users

### SOP-5: Multi-Environment Migration Pipeline

```
LOCAL (development)
  |
  v  supabase db reset (applies all migrations from scratch)
  |  Verify: schema correct, triggers fire, RLS blocks/allows correctly
  |
STAGING (linked Supabase project)
  |
  v  supabase db push --linked (applies only new migrations)
  |  Verify: migration applies cleanly to existing data
  |  Test: application works against staging database
  |
PRODUCTION
  |
  v  supabase db push --linked (with production project linked)
     Verify: migration completes without errors
     Monitor: application logs for query errors or permission denials
```

**Why staging matters:**
Local `db reset` applies migrations to an empty database. Production has existing data, partial indexes, cached query plans, and live connections. Staging catches issues that only appear with existing data: constraint violations on existing rows, lock contention on large tables, trigger failures on legacy data formats.

---

## Schema Design Patterns

See `references/schema-design.md` for comprehensive patterns including:
- Entity design with UUID v7, audit columns, and metadata
- When to use JSONB vs proper columns
- Index selection guide (B-tree, GIN, GiST, partial, covering)
- Enumeration strategies (ENUMs vs CHECK vs lookup tables)

## Trigger Patterns

See `references/trigger-patterns.md` for:
- Common patterns (audit logging, computed columns, cascade updates)
- NEW/OLD qualification rules with examples
- Trigger execution order and debugging
- Anti-patterns to avoid

---

## Quality Gates (Before Marking Complete)

- [ ] Migration file follows naming convention: `{TIMESTAMP}_{description}.sql`
- [ ] Migration includes rollback procedure (documented in header comment or separate down migration)
- [ ] SQL syntax validated by running `supabase db reset` locally
- [ ] RLS policies tested with both `authenticated` and `anon` roles
- [ ] RLS policies tested with `service_role` to verify bypass behavior
- [ ] Triggers use proper `NEW.`/`OLD.` qualification on every column reference
- [ ] All new tables have: `id` (UUID v7), `created_at`, `updated_at`, `deleted_at`, `metadata` (JSONB)
- [ ] Indexes added for columns in WHERE, JOIN, and ORDER BY clauses
- [ ] CHECK constraints added for value-bounded columns (status, rating, etc.)
- [ ] Migration tested against staging with existing data before production push
- [ ] Grants explicitly set for `authenticated`, `anon`, and `service_role` as needed

---

## Cross-Skill References

- **velocity-fastlane** -- If database changes require app updates, coordinate the deploy: migration first, then app build.
- **pipeline-cicd** -- Add migration verification to CI pipeline. Run `supabase db reset` in CI to catch migration errors before merge.
- **loki-logs** -- After deploying migrations, monitor application logs for query errors or RLS permission denials that indicate policy misconfiguration.
