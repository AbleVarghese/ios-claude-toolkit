---
name: atlas-database
description: "This skill should be used when the user asks to 'create a migration', 'design database schema', 'add RLS policy', 'write a trigger function', or 'repair migration history'. Provides PostgreSQL/Supabase migration management, schema design, RLS policies, and trigger function development."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: true
---

# ATLAS - Database Migration & Schema

PostgreSQL/Supabase database skill covering migration creation and repair, schema design, Row Level Security policies, and trigger function development with proper column qualification.

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). In particular, locate the Supabase project directory from the project structure (commonly `backend/supabase/`, `supabase/`, or `Backend/supabase/`). This directory is referred to as `{SUPABASE_DIR}` throughout this skill.

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
-- ============================================================================

-- PART 1: Schema Changes
-- PART 2: RLS Policies
-- PART 3: Triggers/Functions
-- PART 4: Grants
-- PART 5: Verification
EOF

# 3. Push migration
cd {SUPABASE_DIR} && supabase db push
```

### SOP-2: Repair Migration History
```bash
# When remote has migrations not in local
supabase migration repair --status reverted [migration_ids...]
supabase db pull  # Sync local with remote

# When local has migrations not applied
supabase migration repair --status applied [migration_ids...]
supabase db push
```

### SOP-3: Debug Trigger Issues
```sql
-- Check existing triggers
SELECT tgname, tgrelid::regclass, tgtype
FROM pg_trigger
WHERE NOT tgisinternal;

-- Check function definition
SELECT pg_get_functiondef(oid)
FROM pg_proc
WHERE proname = 'function_name';

-- CRITICAL: Always qualify column references in triggers
-- BAD:  IF reference IS NULL THEN
-- GOOD: IF NEW.reference IS NULL THEN
```

## Quality Gates

Before marking any database task complete:
- [ ] Migration file follows naming convention
- [ ] SQL syntax validated
- [ ] RLS policies tested with both authenticated and service_role
- [ ] Triggers use proper NEW/OLD qualification
- [ ] Migration pushed and verified in remote
- [ ] Rollback procedure documented
