# DBT MATERIALIZATIONS: VIEWS, TABLES & EPHEMERAL - COMPLETE GUIDE
## From Scratch to Production Level

---

## SECTION 1: WHAT ARE DBT MATERIALIZATIONS?

In dbt, a **"materialization"** defines HOW your model's SQL gets deployed to the data warehouse. It determines whether the result is stored physically or computed on-the-fly.

### The 4 core materializations in dbt:

| # | Materialization | Description |
|---|----------------|-------------|
| 1 | **VIEW** | Creates a database view (virtual table) |
| 2 | **TABLE** | Creates a physical table (full data copy) |
| 3 | **EPHEMERAL** | Creates nothing in the database (inline CTE) |
| 4 | **INCREMENTAL** | Creates a table that appends/merges new data only |

> This guide focuses on **VIEW**, **TABLE**, and **EPHEMERAL**.

---

## SECTION 2: VIEW MATERIALIZATION

### What is it?

A view is a saved SQL query. It does **NOT** store data physically. Every time you query the view, the underlying SQL re-executes.

### In dbt:

dbt translates your model's SELECT statement into:
```sql
CREATE OR REPLACE VIEW <schema>.<model_name> AS (
  <your SELECT statement>
);
```

### Configuration:

In `dbt_project.yml`:
```yaml
models:
  my_project:
    staging:
      +materialized: view
```

Or in the model file itself:
```sql
{{ config(materialized='view') }}
```

### Example: Staging model as a VIEW

**File:** `models/staging/stg_customers.sql`
```sql
{{ config(materialized='view') }}

SELECT
    id AS customer_id,
    first_name,
    last_name,
    email,
    created_at
FROM {{ source('raw', 'customers') }}
WHERE _deleted = FALSE
```

**What dbt generates in Snowflake:**
```sql
CREATE OR REPLACE VIEW analytics.staging.stg_customers AS (
    SELECT
        id AS customer_id,
        first_name,
        last_name,
        email,
        created_at
    FROM raw.public.customers
    WHERE _deleted = FALSE
);
```

### When to use views:

1. Staging/source layer models (lightweight transformations)
2. When data must always be real-time/current
3. Simple renaming, casting, or filtering of source columns
4. When storage cost is a concern
5. Models that are queried infrequently

### Problems views solve:

1. No storage duplication (saves Snowflake credits on storage)
2. Always returns latest data from source
3. Fast to build during `dbt run` (just DDL, no data movement)
4. Great for staging layer where you just clean/rename columns

### Drawbacks:

1. Slow query performance on complex logic (recomputes every time)
2. Downstream queries pay the compute cost repeatedly
3. Can cause warehouse timeouts if underlying query is heavy
4. No micro-partition pruning benefits (Snowflake-specific)

---

## SECTION 3: TABLE MATERIALIZATION

### What is it?

A table materialization creates a **PHYSICAL table** in the warehouse. Data is stored on disk. Queries read pre-computed results.

### In dbt:

```sql
CREATE OR REPLACE TABLE <schema>.<model_name> AS (
  <your SELECT statement>
);
```

On every `dbt run`, the table is **DROPPED and RECREATED** from scratch.

### Configuration:
```sql
{{ config(materialized='table') }}
```

### Example: Mart/fact model as a TABLE

**File:** `models/marts/fct_orders.sql`
```sql
{{ config(
    materialized='table',
    cluster_by=['order_date'],
    tags=['daily']
) }}

SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    SUM(p.amount) AS total_amount,
    COUNT(p.payment_id) AS payment_count
FROM {{ ref('stg_orders') }} o
LEFT JOIN {{ ref('stg_payments') }} p
    ON o.order_id = p.order_id
GROUP BY 1, 2, 3, 4
```

**What dbt generates in Snowflake:**
```sql
CREATE OR REPLACE TABLE analytics.marts.fct_orders
    CLUSTER BY (order_date)
AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        SUM(p.amount) AS total_amount,
        COUNT(p.payment_id) AS payment_count
    FROM analytics.staging.stg_orders o
    LEFT JOIN analytics.staging.stg_payments p
        ON o.order_id = p.order_id
    GROUP BY 1, 2, 3, 4
);
```

### When to use tables:

1. Final mart/reporting layer models queried by BI tools
2. Complex aggregations, joins, window functions
3. Models queried frequently by many users
4. When query performance matters more than build time
5. Small-to-medium datasets that can be fully rebuilt quickly
6. Models with heavy transformations (multiple joins, CTEs)

### Problems tables solve:

1. Fast query performance (pre-computed, stored on disk)
2. Micro-partition pruning in Snowflake (especially with cluster keys)
3. Consistent query times regardless of source complexity
4. BI tools (Tableau, Looker, Power BI) perform well on tables
5. Result caching in Snowflake (identical queries return instantly)

### Drawbacks:

1. Full rebuild every run (expensive for large datasets)
2. Storage costs (duplicate data)
3. Data staleness between dbt runs
4. Longer dbt run times for large tables
5. Transient period during rebuild where table is being replaced

---

## SECTION 4: EPHEMERAL MATERIALIZATION

### What is it?

An ephemeral model creates **NOTHING** in the database. It is injected as a Common Table Expression (CTE) into downstream models.

### In dbt:

dbt does **NOT** run any DDL/DML for ephemeral models. Instead, it inlines the SQL as a CTE wherever the model is referenced.

### Configuration:
```sql
{{ config(materialized='ephemeral') }}
```

### Example: Ephemeral helper model

**File:** `models/staging/stg_payment_methods.sql`
```sql
{{ config(materialized='ephemeral') }}

SELECT
    payment_id,
    order_id,
    CASE
        WHEN payment_method = 'cc' THEN 'credit_card'
        WHEN payment_method = 'bnk' THEN 'bank_transfer'
        WHEN payment_method = 'gc' THEN 'gift_card'
        ELSE payment_method
    END AS payment_method_name,
    amount
FROM {{ source('raw', 'payments') }}
```

### When a downstream model references it:

**File:** `models/marts/fct_revenue.sql`
```sql
{{ config(materialized='table') }}

SELECT
    payment_method_name,
    SUM(amount) AS total_revenue
FROM {{ ref('stg_payment_methods') }}   -- ephemeral model
GROUP BY 1
```

### What dbt ACTUALLY generates (ephemeral is inlined as CTE):
```sql
CREATE OR REPLACE TABLE analytics.marts.fct_revenue AS (
    WITH __dbt__cte__stg_payment_methods AS (
        SELECT
            payment_id,
            order_id,
            CASE
                WHEN payment_method = 'cc' THEN 'credit_card'
                WHEN payment_method = 'bnk' THEN 'bank_transfer'
                WHEN payment_method = 'gc' THEN 'gift_card'
                ELSE payment_method
            END AS payment_method_name,
            amount
        FROM raw.public.payments
    )
    SELECT
        payment_method_name,
        SUM(amount) AS total_revenue
    FROM __dbt__cte__stg_payment_methods
    GROUP BY 1
);
```

### When to use ephemeral:

1. Small utility/helper transformations
2. Intermediate mapping logic (code lookups, enums)
3. Models referenced by only 1-2 downstream models
4. When you want to keep your warehouse schema clean
5. Lightweight transformations that don't need independent testing

### Problems ephemeral solves:

1. Reduces schema clutter (no objects created in warehouse)
2. Zero storage cost
3. Zero compute for creation (no DDL executed)
4. Keeps intermediate logic modular without polluting the database
5. Great for DRY (Don't Repeat Yourself) code reuse

### Drawbacks:

1. Cannot be queried directly (no object exists in warehouse)
2. Cannot be tested with dbt tests (no table/view to test against)
3. Harder to debug (must look at compiled SQL)
4. If referenced by many models, the CTE is duplicated in each
5. Can create very large compiled SQL (performance risk)
6. Does NOT appear in dbt docs lineage graph as a queryable node
7. Cannot use dbt freshness checks on ephemeral models

---

## SECTION 5: COMPARISON TABLE

| Feature | VIEW | TABLE | EPHEMERAL |
|---------|------|-------|-----------|
| Stored in DB? | Yes | Yes | No |
| Data stored? | No | Yes | No |
| Query speed | Slow | Fast | N/A |
| Build speed | Fast | Slow | Instant |
| Storage cost | None | High | None |
| Always fresh? | Yes | No | Yes |
| Testable? | Yes | Yes | No |
| Queryable? | Yes | Yes | No |
| Cluster keys? | No | Yes | No |

---

## SECTION 6: SNOWFLAKE-SPECIFIC VIEW TYPES

Beyond dbt's view materialization, Snowflake has multiple view types:

### 6.1 STANDARD VIEW
```sql
CREATE OR REPLACE VIEW analytics.reporting.v_active_customers AS
    SELECT customer_id, first_name, last_name, email
    FROM analytics.marts.dim_customers
    WHERE is_active = TRUE;
```

### 6.2 SECURE VIEW (hides definition from non-owners)
```sql
CREATE OR REPLACE SECURE VIEW analytics.reporting.v_customer_pii AS
    SELECT customer_id, first_name, last_name, email, phone
    FROM analytics.marts.dim_customers;
```
> **USE CASE:** Data sharing, multi-tenant environments, security compliance. The query definition is hidden from SHOW VIEWS and GET_DDL for non-owners.

### 6.3 MATERIALIZED VIEW (Snowflake auto-maintains, not a dbt concept)
```sql
CREATE OR REPLACE MATERIALIZED VIEW analytics.reporting.mv_daily_revenue AS
    SELECT
        order_date,
        SUM(total_amount) AS daily_revenue,
        COUNT(DISTINCT customer_id) AS unique_customers
    FROM analytics.marts.fct_orders
    GROUP BY order_date;
```
> **USE CASE:** Frequently queried aggregations on large tables. Snowflake automatically refreshes when base table changes. Enterprise Edition required.

---

## SECTION 7: SNOWFLAKE TABLE TYPES

### 7.1 PERMANENT TABLE (default)
```sql
CREATE OR REPLACE TABLE analytics.marts.dim_products (
    product_id INT,
    product_name VARCHAR,
    category VARCHAR
);
```
- Full Time Travel (up to 90 days)
- Fail-safe (7 days additional recovery)
- Highest storage cost

### 7.2 TRANSIENT TABLE
```sql
CREATE OR REPLACE TRANSIENT TABLE analytics.staging.stg_events (
    event_id INT,
    event_type VARCHAR,
    created_at TIMESTAMP
);
```
- Time Travel up to 1 day only
- No Fail-safe period
- Lower storage cost
- **USE CASE:** Staging tables, intermediate transformations, ETL landing

### 7.3 TEMPORARY TABLE
```sql
CREATE OR REPLACE TEMPORARY TABLE session_temp_results (
    metric_name VARCHAR,
    metric_value FLOAT
);
```
- Exists only for the session duration
- No Time Travel, no Fail-safe
- Lowest cost (auto-dropped when session ends)
- **USE CASE:** Session-specific computations, ad-hoc analysis

### 7.4 DYNAMIC TABLE (Snowflake's declarative pipeline)
```sql
CREATE OR REPLACE DYNAMIC TABLE analytics.pipeline.dt_daily_summary
    TARGET_LAG = '1 hour'
    WAREHOUSE = compute_wh
AS
    SELECT
        DATE_TRUNC('day', event_time) AS event_date,
        COUNT(*) AS event_count
    FROM raw.events.page_views
    GROUP BY 1;
```
> **USE CASE:** Replaces streams+tasks for simple-to-moderate pipelines. Snowflake auto-refreshes based on TARGET_LAG.

### 7.5 EXTERNAL TABLE (data lives outside Snowflake)
```sql
CREATE OR REPLACE EXTERNAL TABLE analytics.external.ext_s3_logs
    WITH LOCATION = @my_s3_stage
    FILE_FORMAT = (TYPE = 'PARQUET');
```
> **USE CASE:** Query data in S3/Azure/GCS without loading into Snowflake.

### 7.6 ICEBERG TABLE (open table format)
```sql
CREATE OR REPLACE ICEBERG TABLE analytics.lakehouse.orders
    EXTERNAL_VOLUME = 'my_ext_vol'
    CATALOG = 'snowflake'
    BASE_LOCATION = 'orders/';
```
> **USE CASE:** Multi-engine interoperability (Spark, Flink, Snowflake).

---

## SECTION 7.7: WHICH TABLE TYPES DOES DBT SUPPORT ON SNOWFLAKE?

dbt can create the following table types via configuration options:

| Table Type | dbt Configuration |
|------------|-------------------|
| **Transient Table (DEFAULT)** | `{{ config(materialized='table') }}` — dbt-snowflake creates TRANSIENT tables by default! No fail-safe = lower storage cost. |
| **Permanent Table** | `{{ config(materialized='table', transient=false) }}` — Explicitly set transient=false for full Time Travel + Fail-safe. |
| **Temporary Table** | NOT supported by dbt (session-scoped, would be dropped before downstream models could use them) |
| **Dynamic Table** | `{{ config(materialized='dynamic_table', target_lag='1 hour', snowflake_warehouse='compute_wh') }}` (dbt-snowflake adapter 1.6.0+) |
| **External Table** | NOT natively supported by dbt core. Use `dbt-external-tables` package. |
| **Iceberg Table** | `{{ config(materialized='table', table_format='iceberg', external_volume='my_vol', base_location_subpath='path/') }}` (dbt-snowflake 1.7.0+) |
| **Materialized View** | `{{ config(materialized='materialized_view') }}` (dbt-snowflake 1.6.0+). Limited SQL support (single table, no joins). |

### 7.7.1 TRANSIENT TABLE (DEFAULT in dbt-snowflake)

**File:** `models/staging/stg_raw_events.sql`
```sql
{{ config(
    materialized='table'
) }}

-- This creates a TRANSIENT TABLE by default in dbt-snowflake!
-- No explicit transient=true needed.

SELECT
    event_id,
    event_type,
    payload,
    loaded_at
FROM {{ source('raw', 'events') }}
WHERE loaded_at >= DATEADD('day', -90, CURRENT_DATE())
```

**Why is transient the default?**
- dbt tables are rebuilt from source on every run.
- Data is reproducible, so Fail-safe is unnecessary.
- Saves storage costs (no 7-day Fail-safe period).
- Time Travel is limited to 1 day (usually sufficient for dbt).

### 7.7.2 PERMANENT TABLE (explicitly opt-in)

**File:** `models/marts/fct_sales.sql`
```sql
{{ config(
    materialized='table',
    transient=false,
    cluster_by=['sale_date', 'region']
) }}

SELECT
    sale_id,
    customer_id,
    sale_date,
    region,
    amount
FROM {{ ref('stg_sales') }}
```

**Why permanent (transient=false)?**
- Critical business data that needs full Time Travel (up to 90 days).
- Fail-safe provides additional 7-day disaster recovery.
- Use for final mart tables where data loss is unacceptable.

### 7.7.3 DYNAMIC TABLE (Snowflake auto-refreshes)

**File:** `models/pipeline/dt_hourly_metrics.sql`
```sql
{{ config(
    materialized='dynamic_table',
    target_lag='1 hour',
    snowflake_warehouse='transform_wh'
) }}

SELECT
    DATE_TRUNC('hour', event_time) AS hour,
    event_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT user_id) AS unique_users
FROM {{ ref('stg_events') }}
GROUP BY 1, 2
```

**Why dynamic table?**
- Snowflake manages the refresh schedule automatically.
- No need for dbt to run on a schedule for this model.
- Data is always within 1 hour of real-time.
- Ideal for operational dashboards that need near-real-time data.

### 7.7.4 ICEBERG TABLE (open format, multi-engine access)

**File:** `models/lakehouse/fct_orders_iceberg.sql`
```sql
{{ config(
    materialized='table',
    table_format='iceberg',
    external_volume='s3_lakehouse_vol',
    base_location_subpath='orders/'
) }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount,
    status
FROM {{ ref('stg_orders') }}
```

**Why iceberg?**
- Data is stored in open Parquet format on S3.
- Spark, Flink, Trino can also read this table directly.
- No vendor lock-in to Snowflake for downstream consumers.

### 7.7.5 MATERIALIZED VIEW (Snowflake auto-refreshes, limited SQL)

**File:** `models/reporting/mv_product_summary.sql`
```sql
{{ config(
    materialized='materialized_view'
) }}

SELECT
    product_id,
    COUNT(*) AS order_count,
    SUM(quantity) AS total_quantity,
    AVG(price) AS avg_price
FROM {{ ref('fct_order_items') }}
GROUP BY product_id
```

**Why materialized view?**
- Always up-to-date (Snowflake refreshes automatically).
- Query optimizer can auto-rewrite base table queries to use this.
- No dbt run needed to keep it fresh.
- **LIMITATIONS:** Single table only, no joins, no window functions, no UDFs, limited aggregate functions.

### 7.7.6 INCREMENTAL with TRANSIENT (large fact tables, low storage)

**File:** `models/marts/fct_page_views.sql`
```sql
{{ config(
    materialized='incremental',
    unique_key='page_view_id',
    incremental_strategy='merge',
    transient=true,
    cluster_by=['view_date']
) }}

SELECT
    page_view_id,
    user_id,
    page_url,
    view_date,
    duration_seconds
FROM {{ ref('stg_page_views') }}

{% if is_incremental() %}
WHERE view_date > (SELECT MAX(view_date) FROM {{ this }})
{% endif %}
```

**Why incremental + transient?**
- Billions of rows: full rebuild would take hours.
- Append-only event data: only new records need processing.
- Transient: source is reproducible, no fail-safe needed.

### Summary: dbt Materializations vs Snowflake Object Types

| dbt Materialization | Snowflake Object Created |
|---------------------|--------------------------|
| `view` | VIEW (or SECURE VIEW with `secure=true`) |
| `table` | TRANSIENT TABLE (default) or PERMANENT TABLE |
| `table` + `table_format` | ICEBERG TABLE |
| `incremental` | TABLE (with merge/append/delete logic) |
| `ephemeral` | NOTHING (inlined as CTE) |
| `dynamic_table` | DYNAMIC TABLE |
| `materialized_view` | MATERIALIZED VIEW |

**What dbt CANNOT create:**
- Temporary tables (session-scoped, incompatible with DAG execution)
- External tables (use `dbt-external-tables` package as workaround)
- Hybrid tables (Snowflake HTAP tables, no dbt adapter support yet)

---

## SECTION 8: REAL-WORLD APPLICATION ARCHITECTURE

### Production dbt Project Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│  SOURCE (raw database)                                   │
│  - Raw tables loaded by Fivetran/Airbyte/Snowpipe       │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  STAGING LAYER (materialized='view')                     │
│  - 1:1 with source tables                               │
│  - Rename, cast, filter deleted records                 │
│  - stg_customers, stg_orders, stg_payments              │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  INTERMEDIATE LAYER (materialized='ephemeral' or 'view') │
│  - Business logic helpers                               │
│  - int_orders_pivoted, int_payment_type_mapping         │
│  - Not exposed to end users                             │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  MARTS LAYER (materialized='table' or 'incremental')     │
│  - Final business entities                              │
│  - fct_orders, dim_customers, dim_products              │
│  - Consumed by BI tools, analysts, data scientists      │
└─────────────────────────────────────────────────────────┘
```

### dbt_project.yml configuration:
```yaml
models:
  my_project:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral
    marts:
      +materialized: table
      +schema: marts
```

---

## SECTION 9: PRODUCTION-LEVEL DECISION FRAMEWORK

### Decision Tree: Which materialization to use?

```
START
  │
  ├── Is this a source/staging model?
  │     └── YES → USE VIEW
  │
  ├── Is this a small helper used by 1-2 models only?
  │     └── YES → USE EPHEMERAL
  │
  ├── Is this queried by BI tools or end users?
  │     └── YES → Is the dataset large (>100M rows)?
  │                  ├── YES → USE INCREMENTAL
  │                  └── NO  → USE TABLE
  │
  ├── Does it need to always show real-time data?
  │     └── YES → USE VIEW (accept slower queries)
  │
  └── Default → USE TABLE
```

### Production Considerations:

**1. COST OPTIMIZATION**
- Views: Free storage, but repeated compute on every query
- Tables: Pay storage, but save compute on repeated queries
- Rule: If queried > 3x per refresh cycle → TABLE wins

**2. FRESHNESS REQUIREMENTS**
- Real-time needed? → VIEW or DYNAMIC TABLE
- Hourly refresh OK? → TABLE with scheduled dbt run
- Daily refresh OK? → TABLE with nightly dbt run

**3. WAREHOUSE SIZING**
- Views with complex logic → larger warehouse needed at query time
- Tables → larger warehouse needed at build time only
- Ephemeral in many models → can cause query compilation overhead

**4. DEBUGGING**
- Views: Easy to debug (run SELECT on the view)
- Tables: Easy to debug (query the table directly)
- Ephemeral: Hard to debug (check `target/compiled/` folder)

**5. CI/CD IMPACT**
- Views: Fast CI builds (just DDL)
- Tables: Slow CI builds (full data load)
- Ephemeral: No CI build impact (nothing to create)

---

## SECTION 10: COMMON PRODUCTION PATTERNS

### Pattern 1: Staging as VIEW with downstream TABLE
```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}
SELECT * FROM {{ source('raw', 'orders') }}

-- models/marts/fct_orders.sql
{{ config(materialized='table') }}
SELECT ... FROM {{ ref('stg_orders') }}
```

### Pattern 2: Ephemeral for code mapping
```sql
-- models/intermediate/int_status_mapping.sql
{{ config(materialized='ephemeral') }}
SELECT
    CASE status_code
        WHEN 1 THEN 'pending'
        WHEN 2 THEN 'shipped'
        WHEN 3 THEN 'delivered'
        WHEN 4 THEN 'returned'
    END AS status_name,
    status_code
FROM {{ ref('stg_orders') }}
```

### Pattern 3: Secure view for data sharing
```sql
-- models/sharing/shared_customer_summary.sql
{{ config(
    materialized='view',
    secure=true
) }}
SELECT
    region,
    COUNT(*) AS customer_count,
    AVG(lifetime_value) AS avg_ltv
FROM {{ ref('dim_customers') }}
GROUP BY 1
```

### Pattern 4: Transient table for staging in dbt
```sql
-- models/staging/stg_large_events.sql
{{ config(
    materialized='table',
    transient=true
) }}
SELECT * FROM {{ source('raw', 'events') }}
WHERE event_date >= DATEADD('day', -90, CURRENT_DATE())
```

---

## SECTION 11: PROBLEMS EACH MATERIALIZATION SOLVES IN PRODUCTION

| Problem | Solution | Why |
|---------|----------|-----|
| "Our BI dashboards are slow" | Materialize as TABLE with cluster keys | Pre-computed data, Snowflake caches results |
| "Our dbt runs take 4 hours" | Use VIEW for staging, INCREMENTAL for large facts | Reduce unnecessary full table rebuilds |
| "We have 500 objects in our analytics schema" | Use EPHEMERAL for intermediate helpers | Reduce schema clutter, keep only final marts visible |
| "Analysts need real-time source data" | Use VIEW for staging layer | Every query hits live source data |
| "Our Snowflake bill is too high on storage" | Convert infrequently-queried tables to views | Eliminate redundant data copies |
| "We need to share data securely with partners" | Use SECURE VIEW on top of mart tables | Hide logic, expose only approved columns |
| "Our compiled SQL is 5000 lines because of ephemeral" | Convert heavily-referenced ephemeral to VIEW | Each downstream model queries the view instead of inlining |
| "We need different freshness for different consumers" | Mix materializations | VIEW for real-time, TABLE for batch, DYNAMIC TABLE for near-real-time |

---

## SECTION 12: INTERVIEW QUESTIONS - BEGINNER TO PRODUCTION LEVEL

---

### BEGINNER LEVEL (0-1 years experience)

**Q1: What are the different materializations available in dbt?**
> view, table, ephemeral, and incremental.

**Q2: What is the default materialization in dbt?**
> VIEW is the default materialization.

**Q3: What SQL does dbt generate for a view materialization?**
> `CREATE OR REPLACE VIEW <name> AS (<your SELECT>);`

**Q4: What is the difference between a view and a table in dbt?**
> A view stores only the query definition (no data), re-executes on every query. A table stores data physically, pre-computed results.

**Q5: What is an ephemeral model in dbt?**
> An ephemeral model creates nothing in the database. Its SQL is injected as a CTE into any downstream model that references it.

**Q6: Can you run dbt tests on ephemeral models?**
> No, because no database object exists to test against.

**Q7: How do you configure a model's materialization?**
> Either in `dbt_project.yml` under models config, or in the model file using `{{ config(materialized='table') }}`.

**Q8: What is a secure view in Snowflake?**
> A view where the definition (SQL) is hidden from non-owner roles. Used for data sharing and security compliance.

**Q9: What is a transient table?**
> A table with max 1-day Time Travel and no Fail-safe period. Lower storage cost than permanent tables.

**Q10: When would you use a view vs a table?**
> View for staging/lightweight transforms, Table for complex aggregations queried by BI tools.

---

### INTERMEDIATE LEVEL (1-3 years experience)

**Q11: Why is the staging layer typically materialized as views?**
> Staging models do minimal transformation (rename, cast, filter). Views are fast to build, always fresh, and avoid data duplication since marts will materialize the final aggregated result.

**Q12: What happens when an ephemeral model is referenced by 5 downstream models?**
> The ephemeral SQL is duplicated as a CTE in all 5 compiled queries. This can lead to large compiled SQL and repeated computation. Consider converting to a view if referenced by many models.

**Q13: How does Snowflake's result cache interact with tables vs views?**
> For tables: identical queries return cached results instantly (24hr). For views: cache only works if underlying data hasn't changed since last query, which is less likely for views on frequently updated sources.

**Q14: What is the difference between a Snowflake Materialized View and a dbt table materialization?**
> Snowflake MV is auto-refreshed by Snowflake when base table changes (serverless compute cost). dbt table is rebuilt only during `dbt run`. MV has query restrictions (single table, limited functions). dbt table has no SQL restrictions.

**Q15: How do you make a dbt model create a transient table in Snowflake?**
> `{{ config(materialized='table', transient=true) }}` — though in dbt-snowflake, table is already transient by default.

**Q16: What is a Dynamic Table and how does it compare to dbt incremental?**
> Dynamic Table is Snowflake-native declarative pipeline with TARGET_LAG. It auto-refreshes without dbt. dbt incremental requires scheduled dbt runs and explicit merge/append logic.

**Q17: Can you add a cluster key to a dbt table model?**
> Yes. `{{ config(materialized='table', cluster_by=['col1','col2']) }}`

**Q18: What are the trade-offs of using ephemeral vs view for intermediate models?**
> Ephemeral: No DB object (clean schema), but not testable/debuggable. View: Creates DB object (schema clutter), but testable and queryable.

**Q19: How do you handle a model that needs real-time data for some consumers but batch data for others?**
> Create a VIEW for real-time consumers and a TABLE (downstream of the view) for batch/BI consumers. Both can coexist.

**Q20: What is the impact of views on Snowflake warehouse auto-suspend?**
> Views force compute on every query, which keeps warehouses active longer. Tables serve from cache/storage, allowing faster auto-suspend.

---

### ADVANCED LEVEL (3-5 years experience)

**Q21: You have a dbt project with 200 staging views and nightly runs take 45 minutes. The staging views build in 2 minutes. Where is the bottleneck and how do you optimize?**
> The bottleneck is the downstream tables (43 min). Strategies:
> - Convert large fact tables to incremental
> - Use dbt's `--select` flag to run only changed models
> - Increase warehouse size for table builds
> - Use dbt's defer + `state:modified` for CI

**Q22: Explain how ephemeral models affect query compilation time in Snowflake.**
> Heavily nested ephemeral models create very large compiled SQL. Snowflake's query compiler must parse/optimize the entire SQL. This increases compilation time (appears as "queued" in query history). Can hit Snowflake's 1MB query text limit in extreme cases.

**Q23: When would you choose a Snowflake Materialized View over a dbt table for a reporting model?**
> When: (1) the query is simple (single table, basic aggs), (2) you need always-current data without waiting for dbt runs, (3) the base table changes infrequently. Avoid when: complex joins, UDFs, or window functions are needed.

**Q24: How do you implement a "late-binding" view pattern in dbt for Snowflake?**
> Use view materialization with dynamic source references. If source tables are recreated (schema evolution), views automatically pick up new columns. Unlike tables which freeze the schema at build time.

**Q25: Describe a scenario where converting a view to a table actually INCREASES total cost.**
> If the source data changes every minute and the table is rebuilt hourly, but the table is only queried once a day. The rebuild compute + storage cost exceeds the single query compute on a view.

**Q26: How do you handle the "swap" problem when dbt rebuilds tables?**
> dbt uses CREATE OR REPLACE (atomic swap in Snowflake). During the rebuild, the old table remains queryable until the new one is ready. For very large tables, consider incremental to avoid full rebuilds.

**Q27: What is the relationship between dbt materializations and Snowflake's zero-copy clone?**
> dbt doesn't use cloning natively. But you can use pre-hook/post-hook to clone tables for blue-green deployments. Views don't benefit from cloning (no data to clone). Tables can be cloned for testing without additional storage cost.

**Q28: How do ephemeral models interact with dbt's ref() graph resolution?**
> Ephemeral models appear in the DAG but don't create run steps. dbt resolves their SQL at compile time and injects CTEs. They still participate in `--select` and dependency tracking but have no "execute" node in the run plan.

**Q29: Design a materialization strategy for a multi-tenant SaaS analytics platform.**
> - Shared staging: VIEWS (all tenants read same source)
> - Tenant-specific intermediate: EPHEMERAL (no cross-tenant visibility)
> - Tenant marts: TABLES with row-level security via SECURE VIEWS
> - Shared reporting: SECURE VIEWS on top of mart tables
> - Use dbt vars for tenant isolation in SQL logic

**Q30: How do you monitor and alert on view query performance degradation in production?**
> - Query `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` for views
> - Track execution_time trends per view
> - Set up alerts when P95 latency exceeds threshold
> - Consider converting to table if degradation is consistent
> - Use Snowflake Resource Monitors for cost guardrails

---

### PRODUCTION/PRINCIPAL LEVEL (5+ years)

**Q31: You're designing a data platform serving 50 analysts and 200 dashboards. How do you decide materialization strategy at scale?**
> Framework:
> 1. Profile query patterns (frequency, complexity, latency SLA)
> 2. Classify models by access pattern:
>    - Hot path (>10 queries/hr) → TABLE with cluster keys
>    - Warm path (1-10 queries/hr) → TABLE or Materialized View
>    - Cold path (<1 query/hr) → VIEW (save storage)
> 3. Implement tiered refresh:
>    - Critical dashboards → incremental every 15 min
>    - Standard reports → table rebuild every 4 hours
>    - Ad-hoc exploration → views (always fresh)
> 4. Monitor with QUERY_HISTORY and auto-promote views to tables when access frequency crosses threshold.

**Q32: How do you handle zero-downtime deployments with table materializations in dbt?**
> Strategies:
> 1. Blue-green schemas: Build into _staging schema, then swap
> 2. dbt's native CREATE OR REPLACE is atomic in Snowflake
> 3. For very large tables: use incremental + full_refresh=false
> 4. Custom macro: build in temp schema, validate, then rename
> 5. Use dbt defer to only rebuild changed models in CI
> 6. Leverage Snowflake's zero-copy clone for instant rollback

**Q33: Explain the interaction between Snowflake's query optimizer, result cache, and each materialization type.**
> - **TABLE:** Full optimizer (predicate pushdown, partition pruning, join reordering). Result cache valid for 24h if data unchanged.
> - **VIEW:** Optimizer works on expanded query (can be suboptimal for complex nested views). Result cache invalidated on any base table change.
> - **MATERIALIZED VIEW:** Optimizer can auto-rewrite queries against base table to use MV. Own partition pruning. Auto-refresh invalidates old cache but new cache is quickly established.
> - **EPHEMERAL (CTE):** Compiled into final SQL, optimizer treats as single query. Can benefit from CTE result reuse within query.

**Q34: A team reports that their 200-model dbt project takes 2 hours. 60% of models are tables. Diagnose and prescribe a fix.**
> Diagnosis:
> 1. Check DAG for serial bottlenecks (models with many dependents)
> 2. Profile: Are all tables necessary? Check query frequency.
> 3. Identify candidates for conversion:
>    - Infrequently queried tables → convert to views
>    - Large tables with append-only sources → convert to incremental
>    - Small reference tables → keep as table
> 4. Optimize execution:
>    - Increase threads (parallelism) in profiles.yml
>    - Use dbt's slim CI (`state:modified+`)
>    - Multi-warehouse strategy (different sizes per layer)
> 5. Expected improvement: 60-70% reduction in runtime

**Q35: How would you implement a cost-aware auto-materialization recommender for a dbt project?**
> Design:
> 1. Collect metrics:
>    - Per-model build cost (QUERY_HISTORY during dbt run)
>    - Per-model query frequency (ACCESS_HISTORY)
>    - Per-model storage size (TABLE_STORAGE_METRICS)
> 2. Cost formula:
>    - Table cost = build_credits/run + storage_credits/day
>    - View cost = query_credits * queries_per_day
> 3. Decision rule:
>    - If view_cost > table_cost → recommend TABLE
>    - If table_cost > view_cost → recommend VIEW
>    - If referenced <3 times and by <2 models → recommend EPHEMERAL
> 4. Implement as dbt macro or external Python script that reads Snowflake metadata and outputs recommendations.
> 5. Run weekly as part of platform health checks.

**Q36: Describe how you would implement materialization-aware CI/CD testing in dbt.**
> Strategy:
> - **PR builds (fast):** Only build modified models + 1 downstream
>   - Views: Always build (instant)
>   - Tables: Build with LIMIT 1000 (validate SQL compiles)
>   - Ephemeral: Validate compiled SQL syntax
> - **Staging deploy (thorough):**
>   - Build all modified + all downstream in staging schema
>   - Run full tests on tables/views
>   - Compare row counts with production (anomaly detection)
> - **Production deploy:**
>   - Use dbt defer for unchanged models
>   - Full build only for modified models
>   - Automated rollback if tests fail (swap back to clone)

**Q37: How do Snowflake Dynamic Tables compare to the dbt incremental approach for building data pipelines?**
> **Dynamic Tables:**
> - (+) Zero orchestration needed (Snowflake manages refresh)
> - (+) Declarative (just write SELECT)
> - (+) TARGET_LAG gives fine-grained freshness control
> - (-) Less SQL flexibility (no procedural logic)
> - (-) Harder to manage complex DAGs
> - (-) Vendor lock-in to Snowflake
>
> **dbt Incremental:**
> - (+) Full SQL flexibility (merge, delete+insert, append)
> - (+) Portable across warehouses
> - (+) Full lineage/testing/documentation ecosystem
> - (+) Version controlled, code-reviewed
> - (-) Requires orchestration (Airflow, dbt Cloud, etc.)
> - (-) Manual merge logic can be error-prone
>
> **Production recommendation:** Use Dynamic Tables for simple streaming pipelines; use dbt incremental for complex business logic that needs testing and version control.

**Q38: How do you handle the "ephemeral explosion" anti-pattern?**
> The anti-pattern: Overuse of ephemeral leads to:
> - Compiled SQL exceeding Snowflake's limits
> - Unreadable query profiles (single massive query)
> - Impossible to debug intermediate results
>
> Fix:
> 1. Audit: Find ephemerals referenced by >3 downstream models
> 2. Convert those to views (testable, queryable, debuggable)
> 3. Keep ephemeral only for true 1:1 helper transforms
> 4. Add dbt meta tag policy: max 2 levels of ephemeral nesting
> 5. Monitor compiled SQL size in CI (fail if >500KB)

**Q39: Design a multi-environment materialization strategy (dev/staging/prod).**
> **DEV environment:**
> - All models as VIEW (fast iteration, no data movement)
> - Override via dbt_project.yml target
>
> **STAGING environment:**
> - Match production materializations
> - Use subset of data (`WHERE date > '2024-01-01'`)
> - Full test suite runs here
>
> **PRODUCTION:**
> - Staging layer: VIEW
> - Intermediate: EPHEMERAL
> - Marts: TABLE or INCREMENTAL
> - Use dbt's target variable:
> ```sql
> {% if target.name == 'dev' %}
>   {{ config(materialized='view') }}
> {% else %}
>   {{ config(materialized='table') }}
> {% endif %}
> ```

**Q40: What are the implications of each materialization on Snowflake's data governance features (row access policies, masking policies, tags)?**
> - **TABLE:** Full governance support. Policies applied directly. Tags on columns. Data classification works.
> - **VIEW:** Policies on underlying table propagate through. Can also apply policies on the view itself (layered security). Good for column-level masking without duplicating data.
> - **MATERIALIZED VIEW:** Inherits base table policies. Cannot apply separate policies on MV itself.
> - **EPHEMERAL:** No governance possible (no object exists). Policies must be on source tables or downstream tables.
> - **DYNAMIC TABLE:** Full governance support like regular tables.
>
> **Production implication:** If you need data masking on intermediate results, you CANNOT use ephemeral. Use VIEW or TABLE instead.

---

## SECTION 13: QUICK REFERENCE CHEAT SHEET

### USE VIEW WHEN:
- Staging layer (1:1 source mapping)
- Need real-time data
- Simple transforms (rename, cast, filter)
- Storage budget is tight

### USE TABLE WHEN:
- BI tools query the model
- Complex joins/aggregations
- Queried frequently (>3x between refreshes)
- Need fast, predictable query performance

### USE EPHEMERAL WHEN:
- Helper logic used by 1-2 models
- Code mappings, lookups, enums
- Want clean warehouse schema
- Model doesn't need testing

### USE INCREMENTAL WHEN:
- Large fact tables (>100M rows)
- Append-only or slowly changing data
- Full rebuild is too expensive/slow
- Event/log data with timestamps
