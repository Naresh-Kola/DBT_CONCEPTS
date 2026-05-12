# dbt INCREMENTAL MODELS: BEGINNER TO PRODUCTION — COMPLETE GUIDE

Everything about incremental models: what, why, when, how, strategies, edge cases, production patterns, and every interview question.

---

# PART 1: BEGINNER LEVEL

## 1. What is an Incremental Model?

A normal dbt model (`materialized='table'`) **DROPS and RECREATES** the entire table on every run. 100 million rows? Rebuild all 100 million.

An incremental model only processes **NEW or CHANGED rows** since the last run. 100 million rows total, 50K new today? Process only 50K.

**Analogy:**
- TABLE = Rewriting your entire diary from scratch every night.
- INCREMENTAL = Adding only today's entry to your existing diary.

```
TABLE (full refresh):              INCREMENTAL:

Run 1: Process 100M rows          Run 1: Process 100M rows (first run)
Run 2: Process 100M rows          Run 2: Process 50K rows (only new)
Run 3: Process 100M rows          Run 3: Process 48K rows (only new)
Run 4: Process 100M rows          Run 4: Process 52K rows (only new)

Time:  40 min every run           Time: 40 min first, then 30 sec each
Cost:  $$$$ every run             Cost: $$$$ once, then ¢¢ each
```

---

## 2. Why Do We Use Incremental Models?

1. **SPEED:** 40 minutes → 30 seconds (process only new data)
2. **COST:** Scan 500GB → 2GB (less warehouse compute = fewer credits)
3. **SLAs:** Data must be ready in 5 minutes, not 1 hour
4. **SCALE:** Billions of rows can't be rebuilt every run
5. **FREQUENCY:** Hourly/every-5-min runs need to be fast

**Real Numbers (2 billion row table):**

| Approach | Duration | Credits | Annual Cost |
|----------|----------|---------|-------------|
| Full refresh | 45 min | 12 credits | $13,140/year |
| Incremental | 90 sec | 0.5 credits | $548/year |
| **Savings** | **96% faster** | | **$12,592 saved** |

---

## 3. Your First Incremental Model (Simplest Possible)

**Steps:**
1. Tell dbt this model is incremental
2. Write a normal SELECT
3. Add a WHERE clause that only picks new rows

```sql
-- FILE: models/marts/fct_orders.sql

{{
  config(
    materialized='incremental'
  )
}}

SELECT
    order_id,
    customer_id,
    order_total,
    order_status,
    created_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

### How It Works (Step by Step):

**FIRST RUN** (table doesn't exist yet):
1. `is_incremental()` = FALSE (no table to compare against)
2. WHERE clause is SKIPPED
3. ALL rows from stg_orders are selected
4. dbt creates the table with all data
5. Result: Full table created (same as `materialized='table'`)

**SECOND RUN** (table already exists):
1. `is_incremental()` = TRUE (table exists from Run 1)
2. WHERE clause is ACTIVE
3. `{{ this }}` refers to the EXISTING table (fct_orders)
4. `MAX(created_at)` = '2024-06-15 10:30:00' (latest row from Run 1)
5. Only rows WHERE created_at > '2024-06-15 10:30:00' are selected
6. New rows are INSERTED into the existing table
7. Result: Table grows by only the new rows

---

## 4. Key Concepts for Beginners

### `{{ this }}`
Refers to the CURRENT table being built.
In fct_orders.sql, `{{ this }}` = `ANALYTICS_PROD.FINANCE.FCT_ORDERS`
Used to query the existing table to find "what's already there."

### `is_incremental()`
Returns **TRUE** when:
1. The model is configured as `materialized='incremental'`
2. The target table ALREADY EXISTS in the database
3. The `--full-refresh` flag was NOT passed

Returns **FALSE** when:
1. First run (table doesn't exist yet)
2. `--full-refresh` flag was used

### `--full-refresh`
Forces an incremental model to behave like a table model.
Drops and rebuilds from scratch. Use when:
- Data is corrupted
- You changed the model logic and need to reprocess everything
- Weekly maintenance rebuild

```bash
dbt run --select fct_orders --full-refresh
```

---

# PART 2: INTERMEDIATE LEVEL

## 5. unique_key (Handling Updates, Not Just Inserts)

**WITHOUT unique_key:** New rows are INSERTED. If a row is updated in the source, you get DUPLICATES (old version + new version both exist).

**WITH unique_key:** dbt uses MERGE. If a row with that key already exists, it UPDATES it. If it's new, it INSERTS it.

### ❌ WITHOUT unique_key (duplicates on updates):
```
Source on Monday:    order_id=1, status='pending'
Source on Tuesday:   order_id=1, status='shipped'  (status changed!)
Your table after 2 runs:
  order_id=1, status='pending'    ← old version still here
  order_id=1, status='shipped'    ← new version also here
  DUPLICATE!
```

### ✅ WITH unique_key (updates handled correctly):
```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id'
  )
}}
```
```
Source on Monday:    order_id=1, status='pending'   → INSERTED
Source on Tuesday:   order_id=1, status='shipped'   → UPDATED (replaces pending)
Your table after 2 runs:
  order_id=1, status='shipped'    ← only latest version, no duplicate
```

### Compound unique key (multiple columns):
```sql
{{
  config(
    materialized='incremental',
    unique_key=['order_id', 'product_id']
  )
}}
```

---

## 6. The Four Incremental Strategies

The strategy determines HOW new data is merged into the existing table.

| Strategy | SQL Generated | Use Case |
|----------|--------------|----------|
| **append** | `INSERT INTO target SELECT FROM new_data` | Events/logs (never change) |
| **merge** (default) | `MERGE INTO target USING new_data ON key WHEN MATCHED → UPDATE WHEN NOT MATCHED → INSERT` | Rows that can be updated (orders, users) |
| **delete+insert** | `DELETE FROM target WHERE key IN (new); INSERT INTO target SELECT FROM new` | Replace date partitions, daily aggregations |
| **microbatch** (dbt 1.9+) | Process in time batches, delete+insert per batch | Very large event tables, backfills |

---

## 7. Strategy 1: APPEND (Fastest, Simplest)

Just INSERTs new rows. No deduplication, no merge.
**Use for:** Immutable events that NEVER change (clicks, page views, logs).

```sql
{{
  config(
    materialized='incremental',
    incremental_strategy='append'
  )
}}

SELECT
    event_id,
    user_id,
    event_type,
    page_url,
    event_timestamp
FROM {{ ref('stg_snowplow__events') }}

{% if is_incremental() %}
    WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

**What dbt generates:**
```sql
INSERT INTO analytics.fct_events (event_id, user_id, ...)
SELECT event_id, user_id, ...
FROM staging.stg_snowplow__events
WHERE event_timestamp > '2024-06-15 10:30:00';
```

- **PROS:** Fastest (no merge overhead, no target scan)
- **CONS:** If you re-run or have overlapping data → DUPLICATES
- **WHEN:** Events that are write-once (clicks, logs, sensor readings)

---

## 8. Strategy 2: MERGE (Default, Most Common)

Uses MERGE (upsert). Matches on unique_key. If key exists → UPDATE. If new → INSERT.
**Use for:** Rows that can be updated (orders changing status).

```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
  )
}}

SELECT
    order_id,
    customer_id,
    order_total,
    order_status,
    updated_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**What dbt generates:**
```sql
MERGE INTO analytics.fct_orders AS target
USING (
    SELECT order_id, customer_id, order_total, order_status, updated_at
    FROM staging.stg_orders
    WHERE updated_at > '2024-06-15 10:30:00'
) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET
    target.customer_id = source.customer_id,
    target.order_total = source.order_total,
    target.order_status = source.order_status,
    target.updated_at = source.updated_at
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, order_total, order_status, updated_at)
    VALUES (source.order_id, source.customer_id, source.order_total, source.order_status, source.updated_at);
```

- **PROS:** Handles inserts AND updates correctly
- **CONS:** Slower than append (must scan target for matching keys)
- **WHEN:** Orders, users, subscriptions — anything that changes

---

## 9. Strategy 3: DELETE+INSERT (Replace Partitions)

Deletes matching rows from target, then inserts fresh rows.
**Use for:** Recalculating time-based partitions (daily aggregations).

```sql
{{
  config(
    materialized='incremental',
    unique_key='metric_date',
    incremental_strategy='delete+insert'
  )
}}

SELECT
    event_date AS metric_date,
    COUNT(DISTINCT user_id) AS daily_active_users,
    COUNT(*) AS total_events,
    SUM(revenue) AS daily_revenue
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
    WHERE event_date >= (SELECT MAX(metric_date) - INTERVAL '3 days' FROM {{ this }})
{% endif %}

GROUP BY event_date
```

- **PROS:** Clean replacement (no partial updates), good for aggregations
- **CONS:** Brief moment between DELETE and INSERT where data is missing
- **WHEN:** Daily rollups, weekly summaries — replace entire date partitions

### DELETE+INSERT Internal Working Flow

**Existing target table (before run):**
```
metric_date  | daily_active_users | total_events | daily_revenue
2024-06-01   | 5000               | 80000        | 12000.00
2024-06-02   | 5200               | 85000        | 13500.00
2024-06-03   | 4800               | 78000        | 11000.00
2024-06-04   | 5100               | 82000        | 12800.00
2024-06-05   | 5300               | 86000        | 14000.00  ← MAX date
```

**Step 1:** dbt computes new data (SELECT runs with lookback):
- `MAX(metric_date) - 3 days` = 2024-06-02
- Queries source for `event_date >= 2024-06-02`

**Step 2:** dbt runs DELETE:
```sql
DELETE FROM analytics.fct_daily_metrics
WHERE metric_date IN ('2024-06-02','2024-06-03','2024-06-04','2024-06-05','2024-06-06');
```

**Step 3:** dbt runs INSERT:
```sql
INSERT INTO analytics.fct_daily_metrics
SELECT * FROM __dbt_tmp;
```

**Result:** Last 3 days replaced with fresh calculations + new day added.

**How this handles deletes:** If an order was deleted from source on June 4, the recalculated June 4 row won't include it. The old row is DELETEd, and the new INSERT reflects the deletion.

**Limitation:** Only catches deletes within the lookback window (3 days). Rows deleted from older dates need `--full-refresh`.

---

## 10. Strategy 4: MICROBATCH (dbt 1.9+)

Instead of processing ALL new data in one go, microbatch splits the data into time-based chunks (e.g., one chunk per day) and processes each chunk separately. If Monday's chunk fails, only Monday retries — Tuesday and Wednesday are unaffected.

```sql
{{
  config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_timestamp',
    begin='2020-01-01',
    batch_size='day',
    lookback=3
  )
}}

SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp
FROM {{ ref('stg_events') }}
```

**`event_time`** tells dbt which column to use for splitting data into batches.

- **First run:** Processes from `begin='2020-01-01'` to today, one day at a time.
- **Next runs:** Processes only NEW days + re-processes last 3 days (lookback=3).
- **If a batch fails:** Only that batch retries, not the whole model.

---

## 11. Choosing the Right Strategy (Decision Tree)

```
Does your source data CHANGE after it's written?
  NO → append (events, logs, clicks)
  YES ↓

Is the model an AGGREGATION (GROUP BY date)?
  YES → delete+insert (recalculate date partitions)
  NO ↓

Is the table VERY large with time-series data?
  YES → microbatch (automatic time partitioning)
  NO ↓

Default → merge (handles inserts + updates via unique_key)
```

---

# PART 3: ADVANCED LEVEL

## 12. Late-Arriving Data (The Lookback Window)

**Problem:** An event HAPPENED at 10:00 AM but ARRIVED at 10:30 AM. Your incremental ran at 10:15 AM → `MAX(timestamp)` = 10:15. Next run: `WHERE timestamp > 10:15` → the 10:00 event is MISSED forever.

**Solution:** Lookback window — re-process the last N hours.

```sql
-- ❌ NAIVE (misses late data):
WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})

-- ✅ WITH LOOKBACK (catches late data):
WHERE created_at >= (
    SELECT DATEADD('hour', -6, MAX(created_at)) FROM {{ this }}
)
```

Combined with merge strategy, duplicates are handled by unique_key.

**How to choose lookback size:**
| Data Source | Recommended Lookback |
|-------------|---------------------|
| Mobile app events | 6-12 hours |
| Ad platforms (Google/Facebook) | 24-48 hours |
| CRM data (Fivetran) | 2-4 hours |
| Streaming events | 1-2 hours |

---

## 13. Incremental Predicates (Speed Up Large Merges)

**Problem:** Merge must scan the TARGET table to find matching rows. On a 1TB table → full scan takes 5 minutes.

**Solution:** `incremental_predicates` limit which part of target is scanned.

```sql
{{
  config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    incremental_predicates=[
      "DBT_INTERNAL_DEST.event_date >= DATEADD('day', -3, CURRENT_DATE())"
    ]
  )
}}
```

**Explanation:**
- `DBT_INTERNAL_DEST` → dbt's internal alias for the TARGET table
- `.event_date` → column to filter on
- `>= DATEADD('day', -3, CURRENT_DATE())` → only scan last 3 days

**What dbt generates:**
```sql
MERGE INTO analytics.fct_events AS DBT_INTERNAL_DEST
USING (SELECT ... new data ...) AS DBT_INTERNAL_SOURCE
ON DBT_INTERNAL_DEST.event_id = DBT_INTERNAL_SOURCE.event_id
   AND DBT_INTERNAL_DEST.event_date >= DATEADD('day', -3, CURRENT_DATE())
-- ↑ Predicate added to ON clause — Snowflake prunes old partitions
```

- **Without predicates:** Scans ALL 1TB → 5+ minutes
- **With predicates:** Scans only 3 days (~10GB) → 3 seconds
- **Rule:** Predicate window > lookback window (always wider for safety)

---

## 14. on_schema_change (Handling New Columns)

```sql
{{
  config(
    materialized='incremental',
    on_schema_change='sync_all_columns'
  )
}}
```

| Option | Behavior |
|--------|----------|
| `'ignore'` (default) | New columns silently dropped. Target unchanged. |
| `'fail'` | Build FAILS. Forces you to review the change. |
| `'append_new_columns'` | Adds new columns. Old rows get NULL. |
| `'sync_all_columns'` | Adds new AND removes dropped columns. |

**Recommendation:** Staging: `'append_new_columns'` | Marts: `'fail'`

---

## 15. loaded_at vs event_timestamp

| Approach | Filter On | Pros | Cons |
|----------|-----------|------|------|
| event_timestamp + lookback | When event happened | Intuitive | Very late data still missed |
| loaded_at | When row arrived | Never misses data | Needs loaded_at column |
| Both (belt and suspenders) | Either | Most complete | Slightly more complex |

```sql
-- OPTION C: Belt and suspenders
{% if is_incremental() %}
    WHERE _fivetran_synced > (SELECT DATEADD('hour', -1, MAX(_fivetran_synced)) FROM {{ this }})
       OR event_timestamp >= (SELECT DATEADD('hour', -6, MAX(event_timestamp)) FROM {{ this }})
{% endif %}
```

---

## 15B. Handling Deletes in Source

Incremental models **keep deleted rows** by default. Here's how each strategy handles deletes:

| Strategy | Handles Deletes? | How |
|----------|-----------------|-----|
| append | ❌ NO | Only inserts |
| merge | ❌ NO (by default) | Only matches new/updated rows |
| delete+insert | ✅ YES (partial) | Replaces partition — deleted rows disappear |
| microbatch | ✅ YES (partial) | Same as delete+insert per batch |
| --full-refresh | ✅ YES (complete) | Rebuilds from scratch |

### Solution 1: Soft Deletes (Best for merge)

```sql
{{
  config(materialized='incremental', unique_key='order_id', incremental_strategy='merge')
}}

SELECT
    order_id, customer_id, order_total,
    _fivetran_deleted AS is_deleted,
    updated_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

Downstream: `SELECT * FROM fct_orders WHERE is_deleted = FALSE`

### Solution 2: delete+insert (Partition-level)

Replaces entire time partitions. Deleted rows within the lookback window disappear.

### Solution 3: post_hook DELETE (Small tables only)

```sql
{{
  config(
    materialized='incremental', unique_key='order_id',
    post_hook=["DELETE FROM {{ this }} WHERE order_id NOT IN (SELECT order_id FROM {{ ref('stg_orders') }})"]
  )
}}
```

### Solution 4: Weekly --full-refresh (Simplest)

```bash
# Weekdays (fast): 
dbt run --select fct_orders
# Sunday (thorough):
dbt run --select fct_orders --full-refresh
```

### Solution 5: Microbatch with lookback

Each batch reprocesses its time window — deleted rows are not re-inserted.

---

# PART 4: PRODUCTION LEVEL

## 16. Production-Ready Incremental Model

```sql
{% set lookback_hours = var('activity_lookback_hours', 8) %}

{{
  config(
    materialized='incremental',
    unique_key='activity_id',
    incremental_strategy='merge',
    cluster_by=['activity_date', 'user_segment'],
    snowflake_warehouse='TRANSFORM_WH_HEAVY',
    incremental_predicates=[
      "DBT_INTERNAL_DEST.activity_date >= DATEADD('day', -5, CURRENT_DATE())"
    ],
    on_schema_change='sync_all_columns',
    tags=['product', 'tier_1', 'hourly']
  )
}}

WITH events AS (
    SELECT
        event_id AS activity_id,
        user_id,
        event_type AS activity_type,
        event_timestamp AS activity_timestamp,
        event_timestamp::DATE AS activity_date,
        event_properties,
        loaded_at
    FROM {{ ref('stg_snowplow__events') }}

    {% if is_incremental() %}
        WHERE event_timestamp >= (
            SELECT DATEADD('hour', -{{ lookback_hours }}, MAX(activity_timestamp))
            FROM {{ this }}
        )
    {% endif %}
),

enriched AS (
    SELECT
        e.activity_id, e.user_id, u.user_segment,
        e.activity_type, e.activity_timestamp, e.activity_date,
        e.event_properties:page_url::STRING AS page_url,
        e.event_properties:device_type::STRING AS device_type,
        e.loaded_at
    FROM events e
    LEFT JOIN {{ ref('dim_users') }} u ON e.user_id = u.user_id
)

SELECT * FROM enriched
```

**What's included:**
- ✅ unique_key for deduplication
- ✅ merge strategy for updates
- ✅ Lookback window (configurable via var)
- ✅ incremental_predicates (fast merge on large tables)
- ✅ cluster_by (optimized downstream queries)
- ✅ Dedicated warehouse (won't block small models)
- ✅ on_schema_change (handles new columns)
- ✅ Tags (for selective scheduling)

---

## 17. Reusable Lookback Macro

```sql
-- FILE: macros/incremental_lookback.sql
{% macro incremental_lookback(column_name, lookback_hours=6) %}
    {% if is_incremental() %}
        WHERE {{ column_name }} >= (
            SELECT DATEADD('hour', -{{ lookback_hours }}, MAX({{ column_name }}))
            FROM {{ this }}
        )
    {% endif %}
{% endmacro %}
```

**Usage:**
```sql
SELECT * FROM {{ ref('stg_events') }}
{{ incremental_lookback('event_timestamp', 8) }}
```

---

## 18. Scheduling Pattern

| Schedule | Command |
|----------|---------|
| Hourly (Tier 1) | `dbt run --select tag:tier_1` |
| Every 4hr (Tier 2) | `dbt run --select tag:tier_2` |
| Daily at 6 AM | `dbt build --target prod` |
| Weekly Sunday 2 AM | `dbt run --select tag:incremental --full-refresh` |
| After outage | `dbt run --select model --vars '{"lookback": 72}'` |

---

## 19. Common Mistakes and Fixes

| Mistake | Fix |
|---------|-----|
| No lookback window (misses late data) | Add `DATEADD(-N hours)` to WHERE clause |
| Using append when data updates (duplicates) | Switch to merge with unique_key |
| Forgot unique_key with merge | Add unique_key to config |
| Window function over all data (RANK, NTILE) | Use TABLE materialization instead |
| Slow merge on huge target | Add incremental_predicates |
| Making everything incremental | Only if > 10M rows and build > 5 min |
| Never running full-refresh | Schedule weekly --full-refresh |
| No on_schema_change config | Add `on_schema_change='fail'` for marts |

---

# PART 5: INTERVIEW QUESTIONS & ANSWERS

## Q1: "What is an incremental model in dbt?"
An incremental model only processes NEW or CHANGED rows since the last run, instead of rebuilding the entire table. It uses `is_incremental()` to conditionally add a WHERE clause that filters to only recent data. First run loads everything; subsequent runs load only the delta.

## Q2: "Why would you use incremental instead of table?"
Speed and cost. A 2-billion-row table that takes 45 minutes as a full rebuild takes 90 seconds as incremental. This saves 96% in compute costs and enables hourly refreshes with strict SLAs. Use incremental when: table > 10M rows, data has a reliable timestamp, and full rebuild takes > 5 minutes.

## Q3: "What are the incremental strategies in Snowflake?"
Four strategies: **append** (INSERT only, fastest, for immutable events), **merge** (upsert, default, for rows that update), **delete+insert** (replace partitions, for daily aggregations), **microbatch** (automatic time batching, for very large event tables).

## Q4: "What is unique_key and why is it important?"
unique_key tells dbt which column(s) identify a row. With merge strategy, if a row with that key exists → UPDATE it. If new → INSERT. Without unique_key, updated rows create duplicates.

## Q5: "How do you handle late-arriving data?"
Use a lookback window. Instead of `WHERE timestamp > MAX(timestamp)`, use `WHERE timestamp > MAX(timestamp) - INTERVAL '6 hours'`. This re-processes the last 6 hours on every run, catching late arrivals. Merge deduplicates via unique_key. Size window at 2-3x the measured p99 latency.

## Q6: "What is is_incremental() and when is it TRUE?"
Returns TRUE when: (1) model is `materialized='incremental'`, (2) target table already exists, (3) `--full-refresh` flag was NOT passed. On first run or with --full-refresh, it returns FALSE.

## Q7: "What are incremental_predicates?"
They limit which part of the TARGET table Snowflake scans during a merge. Without predicates, a merge on a 1TB table scans everything. With predicates like `event_date >= DATEADD(-3 days)`, only the last 3 days are scanned. 100x faster merges.

## Q8: "When would you NOT use incremental?"
Five scenarios: (1) Small tables < 10M rows, (2) No reliable timestamp, (3) Window functions over all data (RANK, NTILE), (4) Source does hard deletes, (5) Full rebuild < 5 minutes.

## Q9: "How do you recover a corrupted incremental model?"
`dbt run --select fct_orders --full-refresh` — drops and recreates from scratch. For targeted recovery: `dbt run --select fct_orders --vars '{"lookback": 72}'` to extend the lookback.

## Q10: "What is on_schema_change?"
Controls what happens when source columns change: `'ignore'` (dropped), `'fail'` (build fails), `'append_new_columns'` (adds with NULL), `'sync_all_columns'` (adds new, removes dropped). Use `'fail'` for marts, `'append_new_columns'` for staging.

## Q11: "What is {{ this }}?"
A reference to the current model's already-existing table. Resolves to the full path like `ANALYTICS_PROD.FINANCE.FCT_ORDERS`. Used in WHERE clause to query the existing table for the watermark.

## Q12: "How do you handle deletes in source?"
Three solutions: (1) Soft deletes via `_fivetran_deleted` flag, (2) Periodic `--full-refresh` weekly, (3) delete+insert strategy replaces partitions.

## Q13: "Difference between append and merge?"
**append** = INSERT only, no dedup, no target scan. Fastest but creates duplicates on re-runs. **merge** = MERGE (upsert), matches on unique_key, updates existing, inserts new. Slower but handles updates correctly.

## Q14: "How do you test incremental models?"
Extra tests needed: (1) unique on PK (catch duplicates), (2) row count should increase, (3) freshness test, (4) no date gaps, (5) compare totals vs source.

## Q15: "Walk me through designing an incremental model."
1. ANALYZE: What updates? What timestamp? What volume?
2. CHOOSE STRATEGY: merge (if updates), append (if immutable)
3. CHOOSE KEY: unique_key on business identifier
4. CHOOSE LOOKBACK: 2-3x the p99 latency
5. ADD PREDICATES: Limit target scan to N days
6. CONFIGURE: cluster_by, warehouse, on_schema_change, tags
7. TEST: unique, not_null, freshness
8. SCHEDULE: Hourly + weekly full-refresh
9. MONITOR: Duration alerts, row count anomalies

---

## Quick Reference Card

| Config | What It Does |
|--------|-------------|
| `materialized` | `'incremental'` |
| `unique_key` | Column(s) for dedup: `'id'` or `['id','date']` |
| `incremental_strategy` | `'append'`, `'merge'`, `'delete+insert'`, `'microbatch'` |
| `incremental_predicates` | Limit target scan during merge |
| `on_schema_change` | `'ignore'`, `'fail'`, `'append_new_columns'`, `'sync_all_columns'` |
| `cluster_by` | Optimize downstream queries |
| `snowflake_warehouse` | Route to specific warehouse |
| `--full-refresh` | CLI flag to rebuild from scratch |

**Commands:**
```bash
dbt run --select fct_orders                             # incremental run
dbt run --select fct_orders --full-refresh              # rebuild from scratch
dbt run --select tag:tier_1                             # run all tier 1 models
dbt run --select fct_orders --vars '{"lookback": 48}'   # extended lookback
```
