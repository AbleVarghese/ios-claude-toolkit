# Trigger Patterns

Reference guide for PostgreSQL trigger design, debugging, and anti-patterns.

---

## The #1 Trigger Bug: Unqualified Column References

This is the most common trigger bug in PostgreSQL. It causes silent data corruption or unexpected behavior.

```sql
-- WRONG: Bare column name -- resolves to what? The table column? A variable? Nothing?
CREATE OR REPLACE FUNCTION update_status()
RETURNS TRIGGER AS $$
BEGIN
    IF status = 'active' THEN        -- BUG: Which "status"? PostgreSQL guesses.
        updated_at := now();          -- BUG: Assigns to a local variable, not the row.
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- RIGHT: Every column reference uses NEW. or OLD.
CREATE OR REPLACE FUNCTION update_status()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'active' THEN
        NEW.updated_at := now();
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Rule:** In a trigger function, EVERY reference to a table column MUST be prefixed with `NEW.` or `OLD.`. No exceptions.

- `NEW` = the row after the operation (available in INSERT, UPDATE triggers)
- `OLD` = the row before the operation (available in UPDATE, DELETE triggers)
- INSERT triggers: only `NEW` is available
- UPDATE triggers: both `NEW` and `OLD` are available
- DELETE triggers: only `OLD` is available

---

## Common Trigger Patterns

### 1. Auto-update `updated_at`

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to any table:
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON public.items
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 2. Audit Log (track all changes)

```sql
CREATE OR REPLACE FUNCTION audit_log_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public.audit_log (
        table_name,
        record_id,
        action,
        old_data,
        new_data,
        changed_by,
        changed_at
    ) VALUES (
        TG_TABLE_NAME,
        COALESCE(NEW.id, OLD.id),
        TG_OP,  -- 'INSERT', 'UPDATE', or 'DELETE'
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END,
        auth.uid(),
        now()
    );

    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Apply as AFTER trigger (so it runs after the main operation succeeds)
CREATE TRIGGER audit_items
    AFTER INSERT OR UPDATE OR DELETE ON public.items
    FOR EACH ROW
    EXECUTE FUNCTION audit_log_trigger();
```

### 3. Computed Column (denormalized data)

```sql
-- Keep a full_name column in sync with first_name + last_name
CREATE OR REPLACE FUNCTION compute_full_name()
RETURNS TRIGGER AS $$
BEGIN
    NEW.full_name = TRIM(COALESCE(NEW.first_name, '') || ' ' || COALESCE(NEW.last_name, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_full_name
    BEFORE INSERT OR UPDATE OF first_name, last_name ON public.profiles
    FOR EACH ROW
    EXECUTE FUNCTION compute_full_name();
```

### 4. Cascade Soft Delete

```sql
-- When a parent is soft-deleted, soft-delete its children
CREATE OR REPLACE FUNCTION cascade_soft_delete()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.deleted_at IS NOT NULL AND OLD.deleted_at IS NULL THEN
        -- Parent was just soft-deleted
        UPDATE public.child_items
        SET deleted_at = NEW.deleted_at
        WHERE parent_id = NEW.id
        AND deleted_at IS NULL;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cascade_parent_soft_delete
    AFTER UPDATE OF deleted_at ON public.parent_items
    FOR EACH ROW
    WHEN (NEW.deleted_at IS NOT NULL AND OLD.deleted_at IS NULL)
    EXECUTE FUNCTION cascade_soft_delete();
```

### 5. Validate Business Rules

```sql
-- Prevent changing status backwards (state machine enforcement)
CREATE OR REPLACE FUNCTION enforce_status_transitions()
RETURNS TRIGGER AS $$
DECLARE
    valid_transitions JSONB := '{
        "pending": ["confirmed", "cancelled"],
        "confirmed": ["shipped", "cancelled"],
        "shipped": ["delivered"],
        "delivered": [],
        "cancelled": []
    }'::JSONB;
    allowed_next JSONB;
BEGIN
    IF OLD.status = NEW.status THEN
        RETURN NEW;  -- No status change, allow
    END IF;

    allowed_next := valid_transitions -> OLD.status;

    IF NOT (allowed_next ? NEW.status) THEN
        RAISE EXCEPTION 'Invalid status transition: % -> %', OLD.status, NEW.status;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_status_transition
    BEFORE UPDATE OF status ON public.orders
    FOR EACH ROW
    EXECUTE FUNCTION enforce_status_transitions();
```

---

## Trigger Execution Order

| Timing | Level | When It Fires | Can Modify Row? |
|--------|-------|---------------|-----------------|
| BEFORE | Row | Before each row is modified | Yes (`RETURN NEW` with changes) |
| AFTER | Row | After each row is modified | No (row is already committed) |
| BEFORE | Statement | Before the entire statement | N/A |
| AFTER | Statement | After the entire statement | N/A |

**Decision guide:**
- Need to modify the row being inserted/updated? --> BEFORE ROW
- Need to log/audit after the fact? --> AFTER ROW
- Need to validate across multiple rows? --> AFTER STATEMENT (or BEFORE with a deferred constraint)

---

## Debugging Triggers

```sql
-- List all triggers on a table
SELECT
    tgname AS trigger_name,
    tgenabled AS enabled,
    pg_get_triggerdef(oid) AS definition
FROM pg_trigger
WHERE tgrelid = 'public.items'::regclass
AND NOT tgisinternal;

-- Read the trigger function source
SELECT
    proname AS function_name,
    prosrc AS source_code
FROM pg_proc
WHERE proname = 'my_trigger_function';

-- Test trigger in a transaction (rollback to keep data clean)
BEGIN;
    UPDATE public.items SET status = 'active' WHERE id = 'test-uuid';
    SELECT * FROM public.items WHERE id = 'test-uuid';
    SELECT * FROM public.audit_log ORDER BY changed_at DESC LIMIT 5;
ROLLBACK;

-- Temporarily disable a trigger for debugging
ALTER TABLE public.items DISABLE TRIGGER my_trigger_name;
-- ... debug ...
ALTER TABLE public.items ENABLE TRIGGER my_trigger_name;
```

---

## Anti-Patterns

### 1. Recursive Triggers
A trigger that modifies its own table fires itself again, causing infinite recursion.

```sql
-- DANGEROUS: This trigger updates the same table it fires on
CREATE TRIGGER bad_trigger
    AFTER UPDATE ON public.items
    FOR EACH ROW
    EXECUTE FUNCTION update_related_items();
-- If update_related_items() does UPDATE public.items ... it fires itself

-- SAFE: Use pg_trigger_depth() to prevent recursion
CREATE OR REPLACE FUNCTION update_related_items()
RETURNS TRIGGER AS $$
BEGIN
    IF pg_trigger_depth() > 1 THEN
        RETURN NEW;  -- Already in a trigger, don't recurse
    END IF;
    -- ... safe to proceed ...
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 2. Heavy Computation in Triggers
Triggers execute synchronously in the transaction. Slow triggers make every INSERT/UPDATE slow.

```sql
-- BAD: HTTP call, external API, heavy aggregation in a trigger
-- The INSERT doesn't complete until this finishes

-- GOOD: Triggers should be fast (<10ms). For heavy work:
-- 1. Use LISTEN/NOTIFY to send events to a worker
-- 2. Use pg_cron for scheduled aggregation
-- 3. Use Supabase Edge Functions triggered by database webhooks
```

### 3. Triggers as Business Logic
Triggers are invisible to application developers. Business logic in triggers is hard to test, hard to debug, and impossible to trace in application code.

```sql
-- BAD: Complex pricing calculation in a trigger
-- Nobody reading the application code will know this happens

-- GOOD: Keep triggers for infrastructure concerns:
-- updated_at, audit logging, computed columns, constraint enforcement
-- Keep business logic in the application layer where it's visible and testable
```

### 4. SECURITY DEFINER Without Necessity
`SECURITY DEFINER` functions run with the privileges of the function owner (usually the superuser). This bypasses RLS.

```sql
-- Only use SECURITY DEFINER when the function NEEDS elevated privileges
-- Example: audit log trigger that writes to a table the user can't directly access
-- Always set search_path to prevent injection:
CREATE FUNCTION my_func() RETURNS TRIGGER AS $$
BEGIN
    -- ...
END;
$$ LANGUAGE plpgsql SECURITY DEFINER SET search_path = public;
```
