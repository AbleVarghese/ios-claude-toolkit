# Schema Design Patterns

Reference guide for PostgreSQL schema design in Supabase-backed iOS applications.

---

## Entity Design Template

Every table in the application follows this base structure:

```sql
CREATE TABLE public.items (
    -- Primary key: UUID v7 (time-ordered for index locality)
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Business columns
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'archived')),

    -- Audit columns (every table, no exceptions)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,  -- NULL = active, non-NULL = soft-deleted

    -- Extensibility
    metadata JSONB DEFAULT '{}'::jsonb
);

-- Soft-delete filter: most queries should exclude deleted rows
CREATE INDEX idx_items_active ON public.items (user_id)
    WHERE deleted_at IS NULL;

-- Auto-update updated_at via trigger
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON public.items
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## UUID v7 Strategy

**Why UUID v7 over UUID v4:**
- UUID v4 is fully random, causing B-tree index fragmentation (inserts scatter across the tree)
- UUID v7 is time-ordered (first 48 bits are millisecond timestamp), so new rows insert at the end of the index
- Result: 2-5x better insert performance on tables with millions of rows
- Bonus: UUIDs are roughly sortable by creation time without needing `created_at`

**Implementation in PostgreSQL:**
```sql
-- If your PostgreSQL version supports gen_random_uuid() but not UUIDv7 natively,
-- use a custom function or the pgcrypto extension.
-- Supabase provides gen_random_uuid() which generates v4.
-- For v7, use a custom function:

CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS UUID AS $$
DECLARE
    unix_ts_ms BIGINT;
    uuid_bytes BYTEA;
BEGIN
    unix_ts_ms = (EXTRACT(EPOCH FROM clock_timestamp()) * 1000)::BIGINT;
    uuid_bytes = substring(int8send(unix_ts_ms) from 3);
    uuid_bytes = uuid_bytes || gen_random_bytes(10);
    -- Set version 7
    uuid_bytes = set_byte(uuid_bytes, 6, (get_byte(uuid_bytes, 6) & 15) | 112);
    -- Set variant 2
    uuid_bytes = set_byte(uuid_bytes, 8, (get_byte(uuid_bytes, 8) & 63) | 128);
    RETURN encode(uuid_bytes, 'hex')::UUID;
END;
$$ LANGUAGE plpgsql VOLATILE;
```

**Why not auto-incrementing integers:**
- Enumeration attacks: attacker can guess `/api/users/1`, `/api/users/2`, etc.
- ID guessing: sequential IDs leak information about system size and growth rate
- Merge conflicts: impossible to merge data from two databases with integer PKs
- UUIDs are globally unique without coordination between services

---

## Audit Columns

Every table must have these three columns:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- Immutable after INSERT
updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- Auto-updated on every UPDATE
deleted_at TIMESTAMPTZ,                         -- NULL = active, set = soft-deleted
```

**Auto-updating `updated_at`:**
```sql
-- Create the function once, use it on every table
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to each table
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON public.items
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

**Timezone handling:**
- Store as `TIMESTAMPTZ` (timestamp with time zone), always in UTC
- Display in user's local timezone at the UI layer (iOS handles this automatically with `Date`)
- Never use `TIMESTAMP` without time zone -- it's ambiguous and causes timezone bugs

---

## JSONB Usage: When Columns vs Metadata

**Use a proper column when:**
- The field is queried in WHERE clauses
- The field is used in JOINs
- The field needs a foreign key constraint
- The field needs a CHECK constraint
- The field is used 3+ times in application queries
- The field is part of a unique constraint

**Use the `metadata` JSONB column when:**
- The field is experimental or temporary
- Different rows have different optional fields (polymorphic data)
- The field is integration-specific (Stripe metadata, webhook payloads)
- The field is read-only (displayed but never queried)
- You need schema flexibility during rapid development

**Promotion rule:** When a JSONB field appears in 3+ queries, promote it to a proper column with a migration. The migration should:
1. ADD the new column with a DEFAULT
2. Backfill from JSONB: `UPDATE items SET new_col = (metadata->>'field')::type`
3. Remove the field from JSONB: `UPDATE items SET metadata = metadata - 'field'`
4. Add appropriate constraints and indexes on the new column

```sql
-- Example promotion migration
ALTER TABLE public.items ADD COLUMN priority INTEGER DEFAULT 0;
UPDATE public.items SET priority = (metadata->>'priority')::INTEGER WHERE metadata ? 'priority';
UPDATE public.items SET metadata = metadata - 'priority' WHERE metadata ? 'priority';
ALTER TABLE public.items ADD CONSTRAINT check_priority CHECK (priority BETWEEN 0 AND 5);
CREATE INDEX idx_items_priority ON public.items (priority) WHERE deleted_at IS NULL;
```

---

## Index Selection Guide

### B-tree (default)
Best for: equality, range queries, sorting
```sql
CREATE INDEX idx_items_user ON public.items (user_id);
CREATE INDEX idx_items_created ON public.items (created_at DESC);
```

### GIN (Generalized Inverted Index)
Best for: JSONB queries, array contains, full-text search
```sql
-- JSONB field queries
CREATE INDEX idx_items_metadata ON public.items USING GIN (metadata);
-- Supports: metadata @> '{"key": "value"}', metadata ? 'key'

-- Array contains
CREATE INDEX idx_items_tags ON public.items USING GIN (tags);
-- Supports: tags @> ARRAY['swift']

-- Full-text search
CREATE INDEX idx_items_search ON public.items USING GIN (to_tsvector('english', name || ' ' || description));
```

### Partial Indexes
Best for: queries that always filter by a condition (most common: soft-delete filter)
```sql
-- Only index active rows -- smaller index, faster queries
CREATE INDEX idx_items_active_user ON public.items (user_id)
    WHERE deleted_at IS NULL;

-- Only index specific statuses
CREATE INDEX idx_items_pending ON public.items (created_at)
    WHERE status = 'pending';
```

### Covering Indexes (INCLUDE)
Best for: index-only scans (query satisfied entirely from the index, no table access)
```sql
-- Include columns needed by the query so PostgreSQL never touches the table
CREATE INDEX idx_items_user_covering ON public.items (user_id)
    INCLUDE (name, status, created_at)
    WHERE deleted_at IS NULL;

-- Query satisfied entirely from index:
-- SELECT name, status, created_at FROM items WHERE user_id = $1 AND deleted_at IS NULL
```

### Composite Indexes
```sql
-- Order matters: put equality columns first, range columns last
CREATE INDEX idx_items_user_status_created ON public.items (user_id, status, created_at DESC)
    WHERE deleted_at IS NULL;

-- This index supports:
-- WHERE user_id = $1                              (uses first column)
-- WHERE user_id = $1 AND status = $2              (uses first two)
-- WHERE user_id = $1 AND status = $2 ORDER BY created_at DESC  (uses all three)
-- But NOT: WHERE status = $2 (skips first column, can't use index efficiently)
```

---

## Enumeration Patterns

### Option 1: CHECK Constraint (simplest, recommended for <10 values)
```sql
status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'archived'))
```
- Pro: Simple, self-documenting, easy to extend
- Con: Changing values requires ALTER TABLE

### Option 2: PostgreSQL ENUM (type-safe, but rigid)
```sql
CREATE TYPE item_status AS ENUM ('active', 'inactive', 'archived');
ALTER TABLE public.items ADD COLUMN status item_status NOT NULL DEFAULT 'active';
```
- Pro: Type-safe, smaller storage than text
- Con: Cannot remove values, adding values requires `ALTER TYPE ... ADD VALUE` (not reversible)

### Option 3: Lookup Table (most flexible, recommended for >10 values or user-configurable)
```sql
CREATE TABLE public.item_statuses (
    id TEXT PRIMARY KEY,  -- 'active', 'inactive', etc.
    display_name TEXT NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true
);

ALTER TABLE public.items
    ADD CONSTRAINT fk_items_status
    FOREIGN KEY (status) REFERENCES public.item_statuses(id);
```
- Pro: Application can manage values, supports display names and ordering
- Con: Extra JOIN for queries, more complex schema

**Decision rule:**
- 2-5 fixed values that rarely change: CHECK constraint
- 5-10 values that may grow: PostgreSQL ENUM (if you can live with append-only)
- 10+ values or user-configurable: lookup table

---

## Common Entity Patterns

### Users (extends Supabase auth.users)
```sql
CREATE TABLE public.profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE RESTRICT,
    display_name TEXT,
    avatar_url TEXT,
    timezone TEXT DEFAULT 'UTC',
    notification_preferences JSONB DEFAULT '{"push": true, "email": true}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,
    metadata JSONB DEFAULT '{}'::jsonb
);
```

### Orders / Transactions (immutable history)
```sql
CREATE TABLE public.orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
    total_cents BIGINT NOT NULL CHECK (total_cents >= 0),
    currency TEXT NOT NULL DEFAULT 'USD' CHECK (length(currency) = 3),
    -- Immutable: corrections create new records referencing the original
    original_order_id UUID REFERENCES public.orders(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,
    metadata JSONB DEFAULT '{}'::jsonb
);
```

### Notifications / Events (append-only audit log)
```sql
CREATE TABLE public.notification_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
    type TEXT NOT NULL,
    title TEXT NOT NULL,
    body TEXT,
    sent_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at TIMESTAMPTZ,
    -- No updated_at: this is append-only
    -- No deleted_at: never delete notification history
    metadata JSONB DEFAULT '{}'::jsonb
);

-- Index for user's unread notifications
CREATE INDEX idx_notifications_unread ON public.notification_log (user_id, sent_at DESC)
    WHERE read_at IS NULL;
```
