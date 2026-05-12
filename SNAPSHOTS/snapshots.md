# DBT SNAPSHOTS: COMPLETE GUIDE

What are snapshots, why we use them, when to use them, how they work internally, and examples of all strategies and scenarios.

---

## 1. WHAT ARE DBT SNAPSHOTS?

Snapshots = dbt's way of implementing **SCD TYPE 2** (Slowly Changing Dimensions).

They CAPTURE THE STATE of a source table at a point in time, so you can track HOW a record changed over its lifetime.

**SIMPLE EXPLANATION:**
Your source table only shows the CURRENT state of a row. But business needs to know: "What was this customer's status LAST MONTH?" Snapshots keep a HISTORY of every change automatically.

**ANALOGY:**
- Source table = Live CCTV feed (shows only NOW)
- Snapshot table = Recorded footage (shows entire history)

**WHAT dbt DOES:**
1. First run: Takes a "photo" of the source table (all rows)
2. Subsequent runs: Compares current source with previous snapshot
3. If a row CHANGED → closes old record (sets dbt_valid_to) + inserts new record
4. If a row is NEW → inserts it
5. If a row is DELETED → optionally marks it as invalidated

---

## 2. WHY DO WE NEED SNAPSHOTS?

**PROBLEM:** Source systems OVERWRITE data (no history preserved)

**Example: Customer table in source**
- Day 1: `{ id: 101, name: "Rahul", plan: "Basic", city: "Mumbai" }`
- Day 5: `{ id: 101, name: "Rahul", plan: "Premium", city: "Mumbai" }`
- Day 10: `{ id: 101, name: "Rahul", plan: "Premium", city: "Delhi" }`

If you query the source TODAY, you only see the latest:
`{ id: 101, name: "Rahul", plan: "Premium", city: "Delhi" }`

**Questions you CANNOT answer without history:**
- When did Rahul upgrade from Basic to Premium?
- How long was he on Basic plan?
- Did he change city before or after upgrading?
- What was his plan on Day 3?

**SNAPSHOTS solve this by recording EVERY state change.**

---

## 3. WHEN TO USE SNAPSHOTS

- ✓ Source data is MUTABLE (rows get updated, not appended)
- ✓ Business needs to track HISTORY of changes
- ✓ You need SCD Type 2 behavior (multiple rows per entity, valid_from/to)
- ✓ Compliance/audit requirements (know state at any point in time)
- ✓ Dimension tables that change slowly (customer, product, employee)
- ✓ Status tracking (order status history, ticket lifecycle)
- ✓ Contract/pricing changes (what was the price on date X?)

### WHEN NOT TO USE SNAPSHOTS:

- ✗ Source data is IMMUTABLE (event/fact tables — already append-only)
- ✗ Source already provides history (has valid_from/valid_to columns)
- ✗ High-frequency changes (millions of updates/day → snapshot table explodes)
- ✗ Data you don't need history for (staging tables, temp data)
- ✗ Large fact tables (snapshots are for dimensions, not billion-row facts)

---

## 4. HOW SNAPSHOTS WORK (Internal Mechanics)

dbt adds 4 META COLUMNS to every snapshot table:

| META COLUMN | PURPOSE |
|-------------|---------|
| `dbt_scd_id` | Unique hash for each snapshot record (surrogate key) |
| `dbt_updated_at` | When this snapshot record was created |
| `dbt_valid_from` | When this version of the record became active |
| `dbt_valid_to` | When this version was superseded (NULL = current) |

**RULES:**
- `dbt_valid_to = NULL` → This is the CURRENT (active) version
- `dbt_valid_to = timestamp` → This version was replaced at that time

**LIFECYCLE:**

**RUN 1 (Initial):**
All source rows inserted. `dbt_valid_from = now`, `dbt_valid_to = NULL`

**RUN 2 (Detect changes):**
For each source row:
- IF unchanged → do nothing
- IF changed → close old (set `dbt_valid_to = now`) + insert new (`dbt_valid_to = NULL`)
- IF new row → insert (`dbt_valid_from = now`, `dbt_valid_to = NULL`)
- IF deleted → optionally close (set `dbt_valid_to = now`)

---

## 5. SNAPSHOT STRATEGIES

dbt offers 2 strategies to DETECT whether a row has changed:

### STRATEGY 1: TIMESTAMP
> "Has the updated_at column changed since last snapshot?"

- **Requires:** A reliable `updated_at` / `modified_at` column in source
- **How:** Compares `source.updated_at > snapshot.dbt_updated_at`
- **Best for:** Sources with trustworthy timestamp columns

### STRATEGY 2: CHECK
> "Have any of these specific columns changed since last snapshot?"

- **Requires:** List of columns to monitor (or `'all'`)
- **How:** Compares current values vs previous snapshot values
- **Best for:** Sources WITHOUT a reliable `updated_at` column

---

## 6. EXAMPLE 1: TIMESTAMP STRATEGY (Most Common)

SOURCE TABLE: `RAW.CUSTOMERS` (updates the `updated_at` column on every change)

**File:** `/snapshots/snap_customers.sql`
```sql
{% snapshot snap_customers %}

{{
    config(
        target_database='DBT_DEV_DB',
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at'
    )
}}

SELECT
    customer_id,
    first_name,
    last_name,
    email,
    plan_type,
    city,
    is_active,
    updated_at
FROM {{ source('raw', 'customers') }}

{% endsnapshot %}
```

### WALKTHROUGH: What happens over 3 runs

**SOURCE DATA ON DAY 1:**

| customer_id | first_name | plan_type | city | updated_at |
|-------------|------------|-----------|------|------------|
| 101 | Rahul | Basic | Mumbai | 2024-06-01 10:00:00 |
| 102 | Priya | Premium | Delhi | 2024-06-01 10:00:00 |

**AFTER `dbt snapshot` (Run 1):**

| customer_id | plan_type | city | dbt_valid_from | dbt_valid_to | dbt_updated_at |
|-------------|-----------|------|----------------|--------------|----------------|
| 101 | Basic | Mumbai | 2024-06-01 10:00:00 | NULL | 2024-06-01 10:00:00 |
| 102 | Premium | Delhi | 2024-06-01 10:00:00 | NULL | 2024-06-01 10:00:00 |

**SOURCE DATA ON DAY 5 (Rahul upgraded):**

| customer_id | first_name | plan_type | city | updated_at |
|-------------|------------|-----------|------|------------|
| 101 | Rahul | **Premium** | Mumbai | 2024-06-05 14:00:00 | ← CHANGED |
| 102 | Priya | Premium | Delhi | 2024-06-01 10:00:00 | ← unchanged |

**AFTER `dbt snapshot` (Run 2):**

| customer_id | plan_type | city | dbt_valid_from | dbt_valid_to |
|-------------|-----------|------|----------------|--------------|
| 101 | Basic | Mumbai | 2024-06-01 10:00:00 | 2024-06-05 14:00:00 | ← CLOSED |
| 101 | Premium | Mumbai | 2024-06-05 14:00:00 | NULL | ← NEW (current) |
| 102 | Premium | Delhi | 2024-06-01 10:00:00 | NULL | ← unchanged |

**SOURCE DATA ON DAY 10 (Rahul moved city):**

| customer_id | first_name | plan_type | city | updated_at |
|-------------|------------|-----------|------|------------|
| 101 | Rahul | Premium | **Delhi** | 2024-06-10 09:00:00 | ← CHANGED |
| 102 | Priya | Premium | Delhi | 2024-06-01 10:00:00 | ← unchanged |

**AFTER `dbt snapshot` (Run 3):**

| customer_id | plan_type | city | dbt_valid_from | dbt_valid_to |
|-------------|-----------|------|----------------|--------------|
| 101 | Basic | Mumbai | 2024-06-01 10:00:00 | 2024-06-05 14:00:00 | ← historical |
| 101 | Premium | Mumbai | 2024-06-05 14:00:00 | 2024-06-10 09:00:00 | ← historical (CLOSED) |
| 101 | Premium | Delhi | 2024-06-10 09:00:00 | NULL | ← CURRENT |
| 102 | Premium | Delhi | 2024-06-01 10:00:00 | NULL | ← unchanged |

**NOW YOU CAN ANSWER:** "What was Rahul's plan on June 3?"
```sql
SELECT * FROM snap_customers
WHERE customer_id = 101
AND '2024-06-03' BETWEEN dbt_valid_from AND COALESCE(dbt_valid_to, '9999-12-31')
-- Result: plan_type = 'Basic', city = 'Mumbai'
```

---

## 6.1 UNDER THE HOOD: WHAT SQL DOES dbt ACTUALLY EXECUTE?

When you run `dbt snapshot`, dbt does NOT just INSERT rows. It generates a **MERGE statement** behind the scenes.

### STEP 1: FIRST RUN (Table doesn't exist yet)

On the very first run, dbt simply creates the snapshot table and inserts all rows from source with the meta columns:

```sql
CREATE TABLE DBT_DEV_DB.snapshots.snap_customers AS (
    SELECT
        customer_id,
        first_name,
        last_name,
        email,
        plan_type,
        city,
        is_active,
        updated_at,
        MD5(COALESCE(CAST(customer_id AS VARCHAR), ''))
            AS dbt_scd_id,
        updated_at AS dbt_updated_at,
        updated_at AS dbt_valid_from,
        NULL::TIMESTAMP AS dbt_valid_to
    FROM raw.public.customers
);
```

That's it. Simple SELECT INTO with 4 extra columns added.

### STEP 2: SUBSEQUENT RUNS (The MERGE statement)

On every run after the first, dbt generates a MERGE statement:

```sql
MERGE INTO DBT_DEV_DB.snapshots.snap_customers AS target
USING (

    -- Part A: Detect CHANGED rows that need to be CLOSED
    --         (set dbt_valid_to on the old version)
    SELECT
        snap.dbt_scd_id,                         -- identifies the old row
        source.updated_at AS dbt_valid_to,       -- close it with new timestamp
        source.updated_at AS dbt_updated_at,
        'update' AS dbt_change_type
    FROM (
        SELECT * FROM raw.public.customers       -- current source data
    ) AS source
    INNER JOIN DBT_DEV_DB.snapshots.snap_customers AS snap
        ON source.customer_id = snap.customer_id
        AND snap.dbt_valid_to IS NULL             -- only match CURRENT rows
    WHERE source.updated_at > snap.dbt_updated_at -- timestamp is newer = change

    UNION ALL

    -- Part B: INSERT the new version of changed rows
    SELECT
        MD5(COALESCE(CAST(source.customer_id AS VARCHAR), '')
            || '|' || COALESCE(CAST(source.updated_at AS VARCHAR), ''))
            AS dbt_scd_id,
        NULL AS dbt_valid_to,                     -- new version is current
        source.updated_at AS dbt_updated_at,
        'insert' AS dbt_change_type
    FROM (
        SELECT * FROM raw.public.customers
    ) AS source
    INNER JOIN DBT_DEV_DB.snapshots.snap_customers AS snap
        ON source.customer_id = snap.customer_id
        AND snap.dbt_valid_to IS NULL
    WHERE source.updated_at > snap.dbt_updated_at

    UNION ALL

    -- Part C: INSERT brand new rows (not in snapshot at all)
    SELECT
        MD5(COALESCE(CAST(source.customer_id AS VARCHAR), ''))
            AS dbt_scd_id,
        NULL AS dbt_valid_to,
        source.updated_at AS dbt_updated_at,
        'insert' AS dbt_change_type
    FROM (
        SELECT * FROM raw.public.customers
    ) AS source
    LEFT JOIN DBT_DEV_DB.snapshots.snap_customers AS snap
        ON source.customer_id = snap.customer_id
        AND snap.dbt_valid_to IS NULL
    WHERE snap.customer_id IS NULL                -- no match = new row

) AS staged_changes
ON target.dbt_scd_id = staged_changes.dbt_scd_id

-- Action 1: If matched and it's an UPDATE → close the old row
WHEN MATCHED
    AND staged_changes.dbt_change_type = 'update'
THEN UPDATE SET
    target.dbt_valid_to = staged_changes.dbt_valid_to

-- Action 2: If NOT matched and it's an INSERT → add the new row
WHEN NOT MATCHED
    AND staged_changes.dbt_change_type = 'insert'
THEN INSERT (
    customer_id, first_name, last_name, email,
    plan_type, city, is_active, updated_at,
    dbt_scd_id, dbt_updated_at, dbt_valid_from, dbt_valid_to
)
VALUES (
    staged_changes.customer_id, staged_changes.first_name, ...
    staged_changes.dbt_scd_id, staged_changes.dbt_updated_at,
    staged_changes.dbt_updated_at,  -- dbt_valid_from = updated_at
    NULL                            -- dbt_valid_to = NULL (current)
);
```

---

## SOFT DELETES: TWO SCENARIOS EXPLAINED

### SCENARIO 1: SOURCE HAS is_deleted / is_active COLUMN

**Source table: RAW.CUSTOMERS**

| customer_id | name | plan_type | is_active | updated_at |
|-------------|------|-----------|-----------|------------|
| 101 | Rahul | Premium | TRUE | 2024-06-01 10:00:00 |
| 102 | Priya | Gold | TRUE | 2024-06-01 10:00:00 |

**Run 1:** dbt snapshot captures both rows.

**Snapshot table AFTER Run 1:**

| customer_id | name | is_active | dbt_valid_from | dbt_valid_to |
|-------------|------|-----------|----------------|--------------|
| 101 | Rahul | TRUE | 2024-06-01 10:00:00 | NULL |
| 102 | Priya | TRUE | 2024-06-01 10:00:00 | NULL |

**DAY 5:** Business soft-deletes Rahul (sets is_active=FALSE, updates timestamp)

| customer_id | name | plan_type | is_active | updated_at |
|-------------|------|-----------|-----------|------------|
| 101 | Rahul | Premium | **FALSE** | 2024-06-05 14:00:00 | ← SOFT DELETED |
| 102 | Priya | Gold | TRUE | 2024-06-01 10:00:00 |

**Run 2:** dbt snapshot detects the change automatically.

**HOW?** The MERGE's Part A detects:
`source.updated_at (2024-06-05) > snap.dbt_updated_at (2024-06-01)` → Customer 101 HAS CHANGED

dbt does NOT know or care that it's a "soft delete". It treats `is_active` changing from `TRUE→FALSE` exactly the same as `plan_type` changing from `Basic→Premium`. It's just a column value change.

**Snapshot table AFTER Run 2:**

| customer_id | name | is_active | dbt_valid_from | dbt_valid_to |
|-------------|------|-----------|----------------|--------------|
| 101 | Rahul | TRUE | 2024-06-01 10:00:00 | 2024-06-05 14:00:00 | ← CLOSED |
| 101 | Rahul | FALSE | 2024-06-05 14:00:00 | NULL | ← CURRENT (soft deleted) |
| 102 | Priya | TRUE | 2024-06-01 10:00:00 | NULL | ← UNTOUCHED |

**KEY POINT:** The row is NOT removed. dbt captures the soft delete as a NEW VERSION with `is_active=FALSE`. You have full history:
- "Rahul was active from June 1 to June 5"
- "Rahul was soft-deleted on June 5"

To query only active customers from snapshot:
```sql
SELECT * FROM snap_customers
WHERE dbt_valid_to IS NULL        -- current version
  AND is_active = TRUE            -- not soft-deleted
```

### SCENARIO 2: SOURCE DOES NOT HAVE is_deleted / is_active COLUMN

**Source table: RAW.PRODUCTS** (no is_deleted, no is_active column)

| product_id | name | price | category | updated_at |
|------------|------|-------|----------|------------|
| 201 | Laptop | 50000 | Electronics | 2024-06-01 10:00:00 |
| 202 | Headphones | 2000 | Electronics | 2024-06-01 10:00:00 |

**DAY 5:** Business decides to discontinue "Laptop". But source has NO is_active column. Two things can happen:

#### CASE A: Source PHYSICALLY DELETES the row (HARD DELETE)

Source table now (product_id 201 is GONE):

| product_id | name | price | category | updated_at |
|------------|------|-------|----------|------------|
| 202 | Headphones | 2000 | Electronics | 2024-06-01 10:00:00 |

**WITHOUT `invalidate_hard_deletes`:**
The MERGE's Part A looks for `source.updated_at > snap.dbt_updated_at`. But product 201 is NOT in source anymore. The INNER JOIN produces NO match. dbt has NO IDEA the row was deleted.

> **BUG:** Laptop looks like it still exists! Reports show it as active product.

**WITH `invalidate_hard_deletes=True`:**
dbt adds Part D (LEFT JOIN to detect missing rows). This finds product_id=201 and closes it.

> You only know it disappeared. You do NOT know WHY. There is no `is_active=FALSE` row — just the closure.

#### CASE B: Source KEEPS the row but changes some column

| product_id | name | price | category | updated_at |
|------------|------|-------|----------|------------|
| 201 | Laptop | 50000 | **DISCONTINUED** | 2024-06-05 14:00:00 | ← category changed |

dbt detects `updated_at` changed. Works exactly like Scenario 1. Closes old row, inserts new row with `category='DISCONTINUED'`. No special config needed.

#### CASE C: Source KEEPS the row but DOES NOT change any column

The business says "Laptop is discontinued" but the source system makes NO changes. No flag, no timestamp update.

- Timestamp strategy: `updated_at` hasn't changed → NO detection
- Check strategy: column values haven't changed → NO detection
- Hard deletes: Row still exists → NO detection

> **RESULT:** dbt CANNOT detect this. The source MUST signal the change somehow.

---

## FULL MERGE STATEMENT WITH invalidate_hard_deletes = True

```sql
{% snapshot snap_customers %}
{{ config(
    target_database='DBT_DEV_DB',
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=True       -- THIS IS THE ONLY ADDITION
) }}
SELECT * FROM {{ source('raw', 'customers') }}
{% endsnapshot %}
```

The MERGE statement now has **4 parts instead of 3**:

```sql
MERGE INTO DBT_DEV_DB.snapshots.snap_customers AS target
USING (

    -- Part A: Detect CHANGED rows → CLOSE old version
    SELECT
        snap.dbt_scd_id,
        source.updated_at AS dbt_valid_to,
        source.updated_at AS dbt_updated_at,
        'update' AS dbt_change_type
    FROM (SELECT * FROM raw.public.customers) AS source
    INNER JOIN DBT_DEV_DB.snapshots.snap_customers AS snap
        ON source.customer_id = snap.customer_id
        AND snap.dbt_valid_to IS NULL
    WHERE source.updated_at > snap.dbt_updated_at

    UNION ALL

    -- Part B: INSERT new version of changed rows
    SELECT
        MD5(COALESCE(CAST(source.customer_id AS VARCHAR), '')
            || '|' || COALESCE(CAST(source.updated_at AS VARCHAR), ''))
            AS dbt_scd_id,
        NULL AS dbt_valid_to,
        source.updated_at AS dbt_updated_at,
        'insert' AS dbt_change_type
    FROM (SELECT * FROM raw.public.customers) AS source
    INNER JOIN DBT_DEV_DB.snapshots.snap_customers AS snap
        ON source.customer_id = snap.customer_id
        AND snap.dbt_valid_to IS NULL
    WHERE source.updated_at > snap.dbt_updated_at

    UNION ALL

    -- Part C: INSERT brand new rows
    SELECT
        MD5(COALESCE(CAST(source.customer_id AS VARCHAR), ''))
            AS dbt_scd_id,
        NULL AS dbt_valid_to,
        source.updated_at AS dbt_updated_at,
        'insert' AS dbt_change_type
    FROM (SELECT * FROM raw.public.customers) AS source
    LEFT JOIN DBT_DEV_DB.snapshots.snap_customers AS snap
        ON source.customer_id = snap.customer_id
        AND snap.dbt_valid_to IS NULL
    WHERE snap.customer_id IS NULL

    UNION ALL

    -- ================================================================
    -- Part D: HARD DELETES — Close rows DELETED from source
    -- ================================================================
    -- THIS PART ONLY EXISTS WHEN invalidate_hard_deletes = True
    SELECT
        snap.dbt_scd_id,
        CURRENT_TIMESTAMP() AS dbt_valid_to,      -- close with current time
        CURRENT_TIMESTAMP() AS dbt_updated_at,
        'delete' AS dbt_change_type                -- new change type!
    FROM DBT_DEV_DB.snapshots.snap_customers AS snap
    LEFT JOIN (SELECT * FROM raw.public.customers) AS source
        ON snap.customer_id = source.customer_id
    WHERE snap.dbt_valid_to IS NULL               -- only check CURRENT rows
      AND source.customer_id IS NULL              -- source row is GONE

) AS staged_changes
ON target.dbt_scd_id = staged_changes.dbt_scd_id

-- Action 1: CLOSE changed rows (Part A)
WHEN MATCHED AND staged_changes.dbt_change_type = 'update'
THEN UPDATE SET target.dbt_valid_to = staged_changes.dbt_valid_to

-- Action 2: CLOSE hard-deleted rows (Part D) — NEW ACTION
WHEN MATCHED AND staged_changes.dbt_change_type = 'delete'
THEN UPDATE SET target.dbt_valid_to = staged_changes.dbt_valid_to

-- Action 3: INSERT new versions and new rows (Parts B & C)
WHEN NOT MATCHED AND staged_changes.dbt_change_type = 'insert'
THEN INSERT (...)
VALUES (...);
```

### WALKTHROUGH: HARD DELETE DETECTION

**Snapshot table BEFORE run:**

| dbt_scd_id | customer_id | name | plan_type | dbt_valid_from | dbt_valid_to |
|------------|-------------|------|-----------|----------------|--------------|
| abc123 | 101 | Rahul | Premium | 2024-06-01 10:00:00 | NULL |
| def456 | 102 | Priya | Gold | 2024-06-01 10:00:00 | NULL |

**Source table (customer 101 has been PHYSICALLY DELETED):**

| customer_id | name | plan_type | updated_at |
|-------------|------|-----------|------------|
| 102 | Priya | Gold | 2024-06-01 10:00:00 |

**Part D's LEFT JOIN produces:**

| snap.dbt_scd_id | snap.customer_id | source.customer_id | dbt_change_type |
|-----------------|------------------|--------------------|-----------------|
| abc123 | 101 | NULL | delete | ← DETECTED! |
| def456 | 102 | 102 | (filtered out) | ← EXISTS, skip |

**Snapshot table AFTER run:**

| dbt_scd_id | customer_id | name | plan_type | dbt_valid_from | dbt_valid_to |
|------------|-------------|------|-----------|----------------|--------------|
| abc123 | 101 | Rahul | Premium | 2024-06-01 10:00:00 | 2024-06-10 08:00:00 | ← CLOSED |
| def456 | 102 | Priya | Gold | 2024-06-01 10:00:00 | NULL | ← STILL CURRENT |

> **NOTE:** Unlike soft deletes, there is NO new row inserted for hard deletes. The old row is simply CLOSED.

### KEY DIFFERENCES: WITH vs WITHOUT invalidate_hard_deletes

| ASPECT | WITHOUT (default) | WITH invalidate_hard_deletes=True |
|--------|-------------------|-----------------------------------|
| MERGE parts | 3 (A, B, C) | 4 (A, B, C, D) |
| Deleted source rows | Stay as dbt_valid_to = NULL (looks current) | Closed with dbt_valid_to = CURRENT_TIMESTAMP() |
| New row inserted for hard delete? | N/A (not detected) | NO (only closure, no new version row) |
| Performance impact | Faster (3 UNION ALL) | Slightly slower (extra LEFT JOIN) |
| Risk if source is temporarily empty | None | ALL current rows get CLOSED! (dangerous!) |
| When to use | Source never hard-deletes rows | Source does hard-delete rows |

> **WARNING:** If your source table is TEMPORARILY EMPTY (e.g., during an ETL refresh or pipeline failure), Part D will see ALL snapshot rows as "deleted" and CLOSE every single row. This is catastrophic. ALWAYS ensure your source is fully loaded before running snapshots.

---

## SUMMARY: SOFT vs HARD DELETE HANDLING

| DELETE TYPE | HOW dbt DETECTS IT | CONFIG NEEDED |
|-------------|-------------------|---------------|
| Soft delete with flag (is_active, is_deleted) | is_active changes value → detected as normal column change | NONE (automatic) |
| Soft delete via status (status='DISCONTINUED') | status/category changes → detected as normal column change | NONE (automatic) |
| Hard delete (row removed) | Row missing from source → LEFT JOIN detects gap | `invalidate_hard_deletes=True` |
| Silent delete (no change) | UNDETECTABLE — dbt cannot see invisible changes | Fix source system |

---

## WALKTHROUGH: MERGE FOR RUN 2 (Rahul upgraded plan on Day 5)

**Snapshot table BEFORE Run 2:**

| dbt_scd_id | customer_id | plan_type | dbt_valid_from | dbt_valid_to |
|------------|-------------|-----------|----------------|--------------|
| abc123 | 101 | Basic | 2024-06-01 10:00:00 | NULL |
| def456 | 102 | Premium | 2024-06-01 10:00:00 | NULL |

**Source data on Day 5:**

| customer_id | plan_type | updated_at |
|-------------|-----------|------------|
| 101 | Premium | 2024-06-05 14:00:00 | ← newer timestamp |
| 102 | Premium | 2024-06-01 10:00:00 | ← same timestamp |

**The MERGE USING subquery produces these staged_changes:**

**Part A** (update - close old row):

| dbt_scd_id | dbt_valid_to | dbt_change_type |
|------------|--------------|-----------------|
| abc123 | 2024-06-05 14:00:00 | update |

(Only customer 101 matched because its updated_at > dbt_updated_at. Customer 102 NOT matched because timestamps are equal.)

**Part B** (insert - new version):

| dbt_scd_id | dbt_valid_to | dbt_change_type |
|------------|--------------|-----------------|
| xyz789 | NULL | insert |

**Part C** (insert - brand new rows): empty — both customers already exist.

**The MERGE then:**
1. MATCHES dbt_scd_id = abc123 → UPDATE: sets dbt_valid_to = 2024-06-05
2. Does NOT MATCH dbt_scd_id = xyz789 → INSERT: new row with Premium plan

**Snapshot table AFTER Run 2:**

| dbt_scd_id | customer_id | plan_type | dbt_valid_from | dbt_valid_to |
|------------|-------------|-----------|----------------|--------------|
| abc123 | 101 | Basic | 2024-06-01 10:00:00 | 2024-06-05 14:00:00 | ← CLOSED |
| xyz789 | 101 | Premium | 2024-06-05 14:00:00 | NULL | ← NEW |
| def456 | 102 | Premium | 2024-06-01 10:00:00 | NULL | ← UNTOUCHED |

### KEY INSIGHTS:

1. dbt uses MERGE (not DELETE+INSERT) → atomic, safe, transactional
2. Old rows are NEVER deleted → only `dbt_valid_to` gets updated
3. `dbt_scd_id` is the join key for the MERGE (MD5 hash of unique_key + timestamp)
4. Unchanged rows are SKIPPED entirely (no wasted compute)
5. For CHECK strategy: instead of comparing timestamps, dbt compares actual column VALUES using MD5 hash
6. The UNION ALL in USING has 3 parts:
   - Part A = close old versions (UPDATE)
   - Part B = insert new versions of changed rows (INSERT)
   - Part C = insert brand new rows never seen before (INSERT)

### CHECK STRATEGY MERGE DIFFERENCE:

For CHECK strategy, the WHERE clause changes from:
```sql
WHERE source.updated_at > snap.dbt_updated_at
```
to:
```sql
WHERE MD5(source.price || source.category) != MD5(snap.price || snap.category)
```
It compares HASHES of the checked columns instead of timestamps.

### HARD DELETES MERGE ADDITION:

When `invalidate_hard_deletes=True`, dbt adds a FOURTH part to the UNION ALL:
```sql
UNION ALL

-- Part D: Close rows that no longer exist in source
SELECT
    snap.dbt_scd_id,
    CURRENT_TIMESTAMP() AS dbt_valid_to,
    'delete' AS dbt_change_type
FROM DBT_DEV_DB.snapshots.snap_customers AS snap
LEFT JOIN raw.public.customers AS source
    ON snap.customer_id = source.customer_id
WHERE snap.dbt_valid_to IS NULL           -- only current rows
  AND source.customer_id IS NULL          -- source row is GONE
```

---

## 7. EXAMPLE 2: CHECK STRATEGY (No updated_at column)

USE WHEN: Source table has NO reliable `updated_at` column. dbt will compare specific column VALUES to detect changes.

**File:** `/snapshots/snap_products.sql`
```sql
{% snapshot snap_products %}

{{
    config(
        target_database='DBT_DEV_DB',
        target_schema='snapshots',
        unique_key='product_id',
        strategy='check',
        check_cols=['price', 'category', 'is_available']
    )
}}

SELECT
    product_id,
    product_name,
    price,
    category,
    is_available
FROM {{ source('raw', 'products') }}

{% endsnapshot %}
```

### HOW CHECK STRATEGY WORKS:

- **Run 1:** Source has `{ product_id: 1, price: 100, category: 'Electronics' }` → Inserts into snapshot with `dbt_valid_to = NULL`
- **Run 2:** Source has `{ product_id: 1, price: 120, category: 'Electronics' }` → dbt compares: price changed (100 → 120)! → Closes old, inserts new
- **Run 3:** Source has `{ product_id: 1, price: 120, category: 'Electronics' }` → Nothing changed → Does nothing

**CHECK ALL COLUMNS:**
```sql
{{ config(strategy='check', check_cols='all') }}
```
> **WARNING:** `check_cols='all'` is expensive (compares every column every run)

---

## 8. EXAMPLE 3: HANDLING HARD DELETES

**PROBLEM:** If a row is DELETED from source, the snapshot won't know. The old record stays with `dbt_valid_to = NULL` forever.

**SOLUTION:** `invalidate_hard_deletes = True`

```sql
{% snapshot snap_employees %}

{{
    config(
        target_database='DBT_DEV_DB',
        target_schema='snapshots',
        unique_key='emp_id',
        strategy='timestamp',
        updated_at='updated_at',
        invalidate_hard_deletes=True
    )
}}

SELECT
    emp_id, emp_name, department, salary, is_active, updated_at
FROM {{ source('raw', 'employees') }}

{% endsnapshot %}
```

**What happens:**
- **Run 1:** emp_id=103 (Amit) captured in snapshot
- **Run 2:** emp_id=103 DELETED from source
  - **WITHOUT flag:** Amit stays with `dbt_valid_to = NULL` (looks active!) — BUG
  - **WITH flag:** dbt notices 103 is missing → sets `dbt_valid_to = CURRENT_TIMESTAMP()` — CORRECT

---

## 9. EXAMPLE 4: SNAPSHOT WITH SELECT TRANSFORMATION

You can apply transformations in the SELECT (filter, rename, compute):

```sql
{% snapshot snap_orders_active %}

{{
    config(
        target_database='DBT_DEV_DB',
        target_schema='snapshots',
        unique_key='order_id',
        strategy='check',
        check_cols=['status', 'amount']
    )
}}

SELECT
    order_id,
    customer_id,
    status,
    amount,
    CASE
        WHEN status IN ('pending', 'processing') THEN 'OPEN'
        WHEN status = 'shipped' THEN 'IN_TRANSIT'
        WHEN status IN ('delivered', 'completed') THEN 'CLOSED'
        ELSE 'UNKNOWN'
    END AS status_category,
    order_date,
    updated_at
FROM {{ source('raw', 'orders') }}
WHERE status != 'cancelled'

{% endsnapshot %}
```

> **NOTE:** Be careful with WHERE clauses. If a row moves from included → excluded (e.g., status becomes 'cancelled'), it looks like a DELETE. Consider including all rows and filtering in downstream models instead.

---

## 10. EXAMPLE 5: SNAPSHOT ON A MODEL (Not just sources)

You can snapshot any model, not just raw sources. Useful for tracking changes in transformed/aggregated data.

```sql
{% snapshot snap_customer_metrics %}

{{
    config(
        target_database='DBT_DEV_DB',
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='check',
        check_cols=['total_orders', 'total_spent', 'customer_tier']
    )
}}

SELECT
    customer_id,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_spent,
    CASE
        WHEN SUM(amount) > 100000 THEN 'Platinum'
        WHEN SUM(amount) > 50000 THEN 'Gold'
        WHEN SUM(amount) > 10000 THEN 'Silver'
        ELSE 'Bronze'
    END AS customer_tier
FROM {{ ref('stg_orders') }}
GROUP BY customer_id

{% endsnapshot %}
```

This tracks when a customer's tier CHANGES over time! "When did Rahul become a Gold customer?"

---

## 10.1 CHECK STRATEGY: HOW INSERTS, UPDATES & DELETES WORK UNDER THE HOOD

### CHECK strategy config:
```sql
{% snapshot snap_products %}
{{ config(
    target_database='DBT_DEV_DB',
    target_schema='snapshots',
    unique_key='product_id',
    strategy='check',
    check_cols=['product_name', 'price', 'category', 'is_active']
) }}
SELECT * FROM {{ source('raw', 'products') }}
{% endsnapshot %}
```

### HOW CHECK STRATEGY DIFFERS FROM TIMESTAMP

**TIMESTAMP** strategy detects changes by:
```sql
WHERE source.updated_at > snap.dbt_updated_at
-- (compares a SINGLE timestamp column)
```

**CHECK** strategy detects changes by:
```sql
WHERE MD5(source.product_name || source.price || source.category || source.is_active)
   != MD5(snap.product_name || snap.price || snap.category || snap.is_active)
-- (compares a HASH of ALL checked columns)
```

CHECK strategy does NOT need an `updated_at` column. It computes `dbt_updated_at = CURRENT_TIMESTAMP()` at the time of the run.

### STEP 1: FIRST RUN

```sql
CREATE TABLE DBT_DEV_DB.snapshots.snap_products AS (
    SELECT
        product_id, product_name, price, category, is_active,
        MD5(COALESCE(CAST(product_id AS VARCHAR), '')) AS dbt_scd_id,
        CURRENT_TIMESTAMP() AS dbt_updated_at,     -- NOTE: current time, not source column
        CURRENT_TIMESTAMP() AS dbt_valid_from,      -- NOTE: current time, not source column
        NULL::TIMESTAMP AS dbt_valid_to
    FROM raw.public.products
);
```

### STEP 2: THE MERGE (CHECK STRATEGY)

The key difference is in the WHERE clause — MD5 hash comparison instead of timestamp:

```sql
MERGE INTO DBT_DEV_DB.snapshots.snap_products AS target
USING (

    -- Part A: Detect CHANGED rows → CLOSE old version
    SELECT
        snap.dbt_scd_id,
        CURRENT_TIMESTAMP() AS dbt_valid_to,           -- uses CURRENT_TIMESTAMP
        CURRENT_TIMESTAMP() AS dbt_updated_at,
        'update' AS dbt_change_type
    FROM (SELECT * FROM raw.public.products) AS source
    INNER JOIN DBT_DEV_DB.snapshots.snap_products AS snap
        ON source.product_id = snap.product_id
        AND snap.dbt_valid_to IS NULL
    WHERE
        -- *** KEY DIFFERENCE FROM TIMESTAMP STRATEGY ***
        MD5(COALESCE(CAST(source.product_name AS VARCHAR), '')
            || '|' || COALESCE(CAST(source.price AS VARCHAR), '')
            || '|' || COALESCE(CAST(source.category AS VARCHAR), '')
            || '|' || COALESCE(CAST(source.is_active AS VARCHAR), ''))
        !=
        MD5(COALESCE(CAST(snap.product_name AS VARCHAR), '')
            || '|' || COALESCE(CAST(snap.price AS VARCHAR), '')
            || '|' || COALESCE(CAST(snap.category AS VARCHAR), '')
            || '|' || COALESCE(CAST(snap.is_active AS VARCHAR), ''))

    UNION ALL

    -- Part B: INSERT new version of changed rows (same hash WHERE clause)
    ...

    UNION ALL

    -- Part C: INSERT brand new rows (same LEFT JOIN logic)
    ...

) AS staged_changes
ON target.dbt_scd_id = staged_changes.dbt_scd_id

WHEN MATCHED AND staged_changes.dbt_change_type = 'update'
THEN UPDATE SET target.dbt_valid_to = staged_changes.dbt_valid_to

WHEN NOT MATCHED AND staged_changes.dbt_change_type = 'insert'
THEN INSERT (...) VALUES (...);
```

### WALKTHROUGH: INSERT (New product added to source)

**Snapshot BEFORE:**

| product_id | product_name | price | is_active | dbt_valid_from | dbt_valid_to |
|------------|-------------|-------|-----------|----------------|--------------|
| 201 | Laptop | 50000 | TRUE | 2024-06-01 08:00:00 | NULL |
| 202 | Headphones | 2000 | TRUE | 2024-06-01 08:00:00 | NULL |

**Source on Day 5 (new product 203 added):**

| product_id | product_name | price | category | is_active |
|------------|-------------|-------|----------|-----------|
| 201 | Laptop | 50000 | Electronics | TRUE |
| 202 | Headphones | 2000 | Electronics | TRUE |
| **203** | **Mouse** | **500** | **Electronics** | **TRUE** | ← NEW |

- Part A: Hash comparison for 201 → same → SKIP. Same for 202. 203 not in snapshot → no INNER JOIN match → SKIP.
- Part B: Same as Part A → SKIP
- Part C: LEFT JOIN for 203 → `snap.product_id IS NULL` → MATCH! → INSERT

**Snapshot AFTER:**

| product_id | product_name | price | is_active | dbt_valid_from | dbt_valid_to |
|------------|-------------|-------|-----------|----------------|--------------|
| 201 | Laptop | 50000 | TRUE | 2024-06-01 08:00:00 | NULL |
| 202 | Headphones | 2000 | TRUE | 2024-06-01 08:00:00 | NULL |
| **203** | **Mouse** | **500** | **TRUE** | **2024-06-05 08:00:00** | **NULL** | ← NEW |

### WALKTHROUGH: UPDATE (Price changed, no updated_at column needed!)

**Source on Day 10 (Laptop price changed 50000→45000):**

- Part A: Hash for 201: `MD5('Laptop|45000|Electronics|TRUE') != MD5('Laptop|50000|Electronics|TRUE')` → **CHANGE DETECTED!**
- Part B: Same change → INSERT new version
- MERGE: Close old 201, insert new 201 with price=45000

**Snapshot AFTER:**

| product_id | product_name | price | is_active | dbt_valid_from | dbt_valid_to |
|------------|-------------|-------|-----------|----------------|--------------|
| 201 | Laptop | 50000 | TRUE | 2024-06-01 08:00:00 | 2024-06-10 08:00:00 | ← CLOSED |
| 201 | Laptop | **45000** | TRUE | 2024-06-10 08:00:00 | NULL | ← NEW CURRENT |

> **NOTICE:** No `updated_at` column needed! The hash caught the price change.

### WALKTHROUGH: SOFT DELETE — SOURCE HAS is_active FLAG

**Source on Day 15 (Headphones discontinued, is_active set to FALSE):**

- Part A: `MD5('Headphones|2000|Electronics|FALSE') != MD5('Headphones|2000|Electronics|TRUE')` → **CHANGE DETECTED!**
- dbt doesn't know or care that it's a "soft delete flag". It just sees a column value changed.
- MERGE: Close old 202, insert new 202 with `is_active=FALSE`

**Snapshot AFTER:**

| product_id | product_name | price | is_active | dbt_valid_from | dbt_valid_to |
|------------|-------------|-------|-----------|----------------|--------------|
| 202 | Headphones | 2000 | TRUE | 2024-06-01 08:00:00 | 2024-06-15 08:00:00 | ← CLOSED |
| 202 | Headphones | 2000 | **FALSE** | 2024-06-15 08:00:00 | NULL | ← CURRENT (soft deleted) |

### WALKTHROUGH: SOFT DELETE — SOURCE DOES NOT HAVE is_active FLAG

Three cases (identical behavior to timestamp strategy):

**CASE A: Source PHYSICALLY DELETES the row** → Only detectable with `invalidate_hard_deletes=True`

**CASE B: Source changes some column (e.g., category='DISCONTINUED')** → Hash changes → detected automatically

**CASE C: Source keeps the row, changes NOTHING** → UNDETECTABLE. Fix source system.

### SUMMARY: CHECK STRATEGY — INSERT, UPDATE & DELETE HANDLING

| OPERATION | HOW CHECK STRATEGY DETECTS | WHAT MERGE DOES |
|-----------|---------------------------|-----------------|
| INSERT (new row) | LEFT JOIN: source row not in snapshot | Part C: INSERT new row |
| UPDATE (column change) | MD5 hash mismatch (no updated_at needed!) | Part A: close old + Part B: insert new |
| SOFT DELETE (has flag) | is_active hash changes | Same as UPDATE: close old, insert new |
| SOFT DELETE (via column) | category/status hash changes | Same as UPDATE: close old, insert new |
| HARD DELETE (row removed) | LEFT JOIN detects gap (needs `invalidate_hard_deletes=True`) | Part D: close old (no new row) |
| SILENT DELETE (no change) | UNDETECTABLE | NOTHING — fix source system |

### CHECK vs TIMESTAMP: SIDE-BY-SIDE COMPARISON

| ASPECT | TIMESTAMP STRATEGY | CHECK STRATEGY |
|--------|-------------------|----------------|
| Change detection | `source.updated_at > snap.dbt_updated_at` | `MD5(source.check_cols) != MD5(snap.check_cols)` |
| Requires updated_at? | YES (mandatory) | NO (that's the whole point) |
| dbt_valid_from value | `source.updated_at` (from source column) | `CURRENT_TIMESTAMP()` (time of dbt run) |
| Performance | Fast (compare 1 timestamp) | Slower (compute MD5 hash of multiple columns) |
| Accuracy of dbt_valid_from | Exact (source tells you when change happened) | Approximate (you only know when dbt RUN detected it) |
| Missed changes? | If source updates data but forgets timestamp → MISSED! | Never misses if data changed (compares actual values) |
| Best for | Sources with reliable updated_at | Sources without updated_at or unreliable timestamps |

---

## 11. QUERYING SNAPSHOT DATA

```sql
-- Get CURRENT state of all customers (same as source):
SELECT * FROM {{ ref('snap_customers') }}
WHERE dbt_valid_to IS NULL;

-- Get state at a specific point in time:
SELECT * FROM {{ ref('snap_customers') }}
WHERE dbt_valid_from <= '2024-06-03'
  AND (dbt_valid_to > '2024-06-03' OR dbt_valid_to IS NULL);

-- Get full history for a specific customer:
SELECT * FROM {{ ref('snap_customers') }}
WHERE customer_id = 101
ORDER BY dbt_valid_from;

-- Count how many times each customer changed:
SELECT customer_id, COUNT(*) - 1 AS change_count
FROM {{ ref('snap_customers') }}
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY change_count DESC;

-- Duration of each plan:
SELECT
    customer_id,
    plan_type,
    dbt_valid_from,
    COALESCE(dbt_valid_to, CURRENT_TIMESTAMP()) AS valid_until,
    DATEDIFF('day', dbt_valid_from, COALESCE(dbt_valid_to, CURRENT_TIMESTAMP())) AS days_on_plan
FROM {{ ref('snap_customers') }}
ORDER BY customer_id, dbt_valid_from;
```

---

## 12. RUNNING SNAPSHOTS

```bash
# Run ALL snapshots:
dbt snapshot

# Run specific snapshot:
dbt snapshot --select snap_customers

# Run with full refresh (DANGEROUS - rebuilds from scratch, loses history):
dbt snapshot --select snap_customers --full-refresh
# ⚠️ WARNING: This DELETES all historical data and starts fresh!
# Only use if snapshot is corrupted or schema changed.

# Snapshots in dbt build:
dbt build  # runs snapshots + models + tests in dependency order
```

---

## 13. SNAPSHOT CONFIGURATION OPTIONS

**REQUIRED:**

| Option | Description |
|--------|-------------|
| `unique_key` | Column(s) that identify a row (PK) |
| `strategy` | `'timestamp'` or `'check'` |
| `updated_at` | Column name (only for timestamp strategy) |
| `check_cols` | List of columns (only for check strategy) |

**OPTIONAL:**

| Option | Description |
|--------|-------------|
| `target_database` | Database for snapshot table |
| `target_schema` | Schema for snapshot table |
| `invalidate_hard_deletes` | Track deleted rows (default: false) |

**UNIQUE KEY WITH MULTIPLE COLUMNS:**
```sql
{{ config(unique_key="order_id || '-' || line_item_id", ...) }}
```

---

## 14. TIMESTAMP vs CHECK: WHICH TO USE?

| CRITERIA | TIMESTAMP | CHECK |
|----------|-----------|-------|
| Requires | Reliable updated_at col | Nothing special |
| Speed | Fast (only checks 1 col) | Slower (compares N cols) |
| Accuracy | Depends on source updating timestamp | Catches ALL value changes |
| Risk | Misses change if timestamp not updated | Expensive for wide tables |
| Best for | Sources you trust (ERP, CRM) | Sources without reliable timestamps |
| Performance | O(n) - one column compare | O(n×m) - m column compare |

**RECOMMENDATION:**
- Use TIMESTAMP if source has updated_at AND you trust it
- Use CHECK if source has no updated_at OR you don't trust it
- Use CHECK with specific columns (not 'all') for performance

---

## 15. COMMON ISSUES AND FIXES

**ISSUE 1: Snapshot table keeps growing (too many versions)**
- Cause: Source updates updated_at even when no real change
- Fix: Switch to CHECK strategy on specific columns

**ISSUE 2: Snapshot misses changes**
- Cause: Using TIMESTAMP but source doesn't update updated_at
- Fix: Switch to CHECK strategy, or fix source

**ISSUE 3: dbt_valid_from has wrong timestamp**
- Cause: Timezone issues
- Fix: Ensure source timestamps are timezone-aware

**ISSUE 4: Snapshot full-refresh lost all history**
- Cause: Someone ran `dbt snapshot --full-refresh`
- Fix: Restore from Time Travel. NEVER full-refresh in production.

**ISSUE 5: Composite key not working**
- Cause: unique_key needs concatenation
- Fix: `unique_key="col1 || '-' || col2"` (single expression, not a list)

**ISSUE 6: Deleted rows still show as current**
- Cause: `invalidate_hard_deletes` not enabled
- Fix: Add `invalidate_hard_deletes=True`

---

## 16. BEST PRACTICES

- ✓ Run snapshots BEFORE models (snapshot → then run models that ref it)
- ✓ Use timestamp strategy when possible (faster)
- ✓ Always specify `target_schema = 'snapshots'` (separate from models)
- ✓ NEVER run `--full-refresh` on production snapshots (loses history)
- ✓ Monitor snapshot table growth (set alerts if rows grow too fast)
- ✓ Add `invalidate_hard_deletes` for entities that can be deleted
- ✓ Keep snapshot SELECT simple (transform in downstream models)
- ✓ Schedule `dbt snapshot` at regular intervals (daily at minimum)
- ✓ Test snapshots (unique on dbt_scd_id, not_null on dbt_valid_from)
- ✓ Document: "This snapshot tracks customer plan changes for billing audit"

---

## 17. INTERVIEW QUESTIONS ON SNAPSHOTS

**Q1: What are dbt snapshots?**
> dbt's implementation of SCD Type 2. They track historical changes to source data by recording every version of a row with valid_from/valid_to.

**Q2: What are the two snapshot strategies?**
> TIMESTAMP (uses an updated_at column) and CHECK (compares specific column values).

**Q3: When would you use CHECK over TIMESTAMP?**
> When the source has no reliable updated_at column, or when the source updates the timestamp even without real data changes.

**Q4: What are the meta columns dbt adds to snapshot tables?**
> `dbt_scd_id` (surrogate key), `dbt_updated_at` (when captured), `dbt_valid_from` (when active), `dbt_valid_to` (when superseded, NULL = current).

**Q5: How do you identify the current version of a record?**
> `WHERE dbt_valid_to IS NULL`

**Q6: What happens if a row is deleted from source?**
> Without `invalidate_hard_deletes`: stays with `dbt_valid_to = NULL` forever. With it: dbt sets `dbt_valid_to = now`.

**Q7: Can you snapshot a dbt model (not just a source)?**
> Yes. The SELECT can reference `{{ ref('model_name') }}`. Useful for tracking changes in aggregated data.

**Q8: What happens when you run `dbt snapshot --full-refresh`?**
> It DROPS the snapshot table and recreates from scratch. ALL HISTORICAL DATA IS LOST.

**Q9: How do you query a snapshot "as of" a specific date?**
> ```sql
> SELECT * FROM snap_table
> WHERE dbt_valid_from <= '2024-06-03'
> AND (dbt_valid_to > '2024-06-03' OR dbt_valid_to IS NULL)
> ```

**Q10: How often should snapshots run?**
> Depends on change frequency. Daily for slowly changing dimensions, hourly for faster data. Best: every run as part of `dbt build`.

**Q11: What's the difference between snapshots and incremental models?**
> Snapshots track HISTORY (SCD Type 2, multiple rows per entity over time). Incremental models process NEW DATA efficiently. Snapshot = "how did this row change?" vs Incremental = "what's new?"

**Q12: Can snapshots handle composite keys?**
> Yes. Use concatenation: `unique_key="col1 || '-' || col2"`

**Q13: How do you handle snapshot table growth in production?**
> 1. Check if source triggers false changes
> 2. Switch from `check_cols='all'` to specific columns
> 3. Archive old records
> 4. Consider if you really need history for that entity

**Q14: In your migration project, did you use snapshots?**
> "Yes. After migrating dimension tables, we set up snapshots on the 15 key dimension tables. This gave us SCD Type 2 history going forward. We ran snapshots daily via dbt Cloud scheduled jobs."

**Q15: What's the execution order in a dbt pipeline?**
> `dbt seed` → `dbt snapshot` → `dbt run` → `dbt test`. Or just `dbt build` (handles all in dependency order). Snapshots run BEFORE models so models can `ref()` the snapshot.

