# DBT MODELS - COMPLETE GUIDE (LOW LEVEL TO HIGH LEVEL)

---

## SECTION 1: WHAT IS A DBT MODEL?

A dbt model is simply a `.sql` file that contains a single SELECT statement.
That's it. No CREATE TABLE, no INSERT INTO. Just a SELECT.

dbt takes that SELECT and wraps it in DDL (CREATE TABLE/VIEW) for you.

**Example model file:** `models/staging/stg_customers.sql`

```sql
SELECT
    customer_id,
    first_name,
    last_name,
    email
FROM {{ source('raw', 'customers') }}
```

**When you run `dbt run`, dbt converts this into:**

```sql
CREATE OR REPLACE VIEW DBT_DEV_DB.DEV.STG_CUSTOMERS AS (
    SELECT
        customer_id,
        first_name,
        last_name,
        email
    FROM RAW.PUBLIC.CUSTOMERS
);
```

---

## SECTION 2: HOW DBT RUNS A MODEL (LOW-LEVEL INTERNALS)

### STEP-BY-STEP: What happens when you type `dbt run`

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: PARSING                                            │
│                                                             │
│ 1. dbt reads dbt_project.yml                                │
│ 2. dbt scans the models/ directory for all .sql files       │
│ 3. For each .sql file:                                      │
│    - Parses Jinja ({{ }}) expressions                       │
│    - Resolves {{ ref() }} and {{ source() }} functions       │
│    - Reads config() blocks                                  │
│    - Determines materialization type                        │
│ 4. Builds a DAG (Directed Acyclic Graph) of dependencies    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: COMPILATION                                        │
│                                                             │
│ 1. Replaces {{ ref('stg_orders') }} with actual table path  │
│    e.g., DBT_DEV_DB.DEV_STAGING.STG_ORDERS                  │
│ 2. Replaces {{ source('raw','orders') }} with actual path   │
│    e.g., RAW.PUBLIC.ORDERS                                  │
│ 3. Wraps the SELECT in appropriate DDL based on             │
│    materialization:                                         │
│    - view → CREATE VIEW AS (SELECT ...)                     │
│    - table → CREATE OR REPLACE TRANSIENT TABLE AS (SELECT..)│
│    - incremental → MERGE or INSERT                          │
│    - ephemeral → CTE (no DDL, injected inline)             │
│ 4. Compiled SQL stored in target/compiled/ directory        │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 3: EXECUTION                                          │
│                                                             │
│ 1. on-run-start hooks execute first                         │
│ 2. Models execute in DAG order (respecting dependencies)    │
│ 3. Independent models run in PARALLEL (based on threads)    │
│ 4. Dependent models wait for parents to finish              │
│ 5. For each model:                                          │
│    a. CREATE SCHEMA IF NOT EXISTS (auto-creates schema)     │
│    b. Execute the compiled DDL statement                    │
│    c. Grant privileges (if configured)                      │
│ 6. on-run-end hooks execute last                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE 4: RESULT                                             │
│                                                             │
│ - Run results stored in target/run_results.json             │
│ - Manifest stored in target/manifest.json                   │
│ - Logs stored in logs/dbt.log                               │
│ - Success/failure reported per model                        │
└─────────────────────────────────────────────────────────────┘
```

---

## SECTION 2B: REAL EXAMPLE - PARSING STEP BY STEP

Let's say you have this model file:

**FILE:** `models/staging/stg_orders.sql`

```sql
{{ config(materialized='table', schema='staging') }}

SELECT
    order_id,
    customer_id,
    order_date,
    status,
    amount
FROM {{ source('raw', 'orders') }}
WHERE order_date >= '2024-01-01'
```

### STEP 3a: PARSES JINJA ({{ }}) EXPRESSIONS

dbt finds 2 Jinja expressions in this file:

1. `{{ config(materialized='table', schema='staging') }}`
2. `{{ source('raw', 'orders') }}`

It identifies these as:
- `config()` → model configuration directive
- `source()` → reference to a source table defined in sources.yml

If you had `{{ ref('stg_customers') }}` it would also find that as a reference to another model.

### STEP 3b: RESOLVES {{ ref() }} AND {{ source() }} FUNCTIONS

dbt looks up `source('raw', 'orders')` in your `sources.yml`:

```yaml
sources:
  - name: raw
    database: DBT_DEV_DB
    schema: DEV_RAW
    tables:
      - name: orders
```

**RESOLUTION:**
- `{{ source('raw', 'orders') }}` → `DBT_DEV_DB.DEV_RAW.ORDERS`
- `{{ ref('stg_customers') }}` → `DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS`
  (uses `generate_schema_name` macro to determine target schema)

This is also where dbt **BUILDS THE DAG**:
- `source('raw','orders')` means this model DEPENDS ON that source
- `ref('stg_customers')` would mean it DEPENDS ON stg_customers model
- dbt ensures parents run BEFORE children

### STEP 3c: READS config() BLOCKS

```sql
{{ config(materialized='table', schema='staging') }}
```

dbt extracts:
- `materialized = 'table'` → will use CREATE TRANSIENT TABLE AS
- `schema = 'staging'` → custom schema name

**CONFIG HIERARCHY (highest priority wins):**

| Priority | Location | 
|----------|----------|
| 1 (Highest) | `{{ config() }}` in the .sql file |
| 2 | YAML config in schema.yml for that model |
| 3 (Lowest) | dbt_project.yml model configs |

So if `dbt_project.yml` says `+materialized: view` but your model says `{{ config(materialized='table') }}` → **TABLE wins** (model-level overrides project-level)

### STEP 3d: DETERMINES MATERIALIZATION TYPE

Based on config, dbt now knows: `materialized = 'table'`

This tells dbt WHICH SQL TEMPLATE to use during compilation:

| Materialization | SQL Generated |
|-----------------|---------------|
| `table` | `CREATE OR REPLACE TRANSIENT TABLE {schema}.{model_name} AS ({SELECT})` (default is transient; use `transient=false` for permanent) |
| `view` | `CREATE OR REPLACE VIEW {schema}.{model_name} AS ({SELECT})` |
| `incremental` | First run → CREATE TABLE; Next runs → `MERGE INTO...` |
| `ephemeral` | No SQL generated. Inlined as CTE in downstream models. |

### FULL TRACE: FROM .sql FILE TO EXECUTED SQL

**YOUR MODEL FILE (what you write):**
```sql
{{ config(materialized='table', schema='staging') }}

SELECT
    order_id, customer_id, order_date, status, amount
FROM {{ source('raw', 'orders') }}
WHERE order_date >= '2024-01-01'
```

**SCHEMA RESOLUTION (via generate_schema_name macro):**
- `target.schema = 'DEV'`
- `custom_schema = 'staging'` (from config)
- `target.name = 'dev'` (not prod)
- **Result: `DEV_STAGING`**

**FINAL EXECUTED SQL (what runs on Snowflake):**
```sql
CREATE OR REPLACE TRANSIENT TABLE DBT_DEV_DB.DEV_STAGING.STG_ORDERS AS (
    SELECT
        order_id, customer_id, order_date, status, amount
    FROM DBT_DEV_DB.DEV_RAW.ORDERS
    WHERE order_date >= '2024-01-01'
);
```

**WHAT YOU SEE IN TERMINAL:**
```
05:10:10  1 of 3 OK created table model DEV_STAGING.STG_ORDERS [SUCCESS 10 in 1.2s]
```

**WHERE TO FIND COMPILED SQL:**
- `target/compiled/dbt_tutorial/models/staging/stg_orders.sql` — contains the SELECT only
- `target/run/dbt_tutorial/models/staging/stg_orders.sql` — contains the full CREATE TABLE AS statement

---

## SECTION 3: MATERIALIZATIONS (How the SELECT becomes an object)

dbt supports 4 built-in materializations:

| Type | What dbt creates in Snowflake |
|------|-------------------------------|
| **view** | `CREATE VIEW AS (SELECT ...)` — Default materialization. No data stored, runs query each time. Fast to build, slow to query on large data. Use for: light transformations, staging models |
| **table** | `CREATE TRANSIENT TABLE AS (SELECT ...)` [default in dbt-snowflake] — Stores data physically (transient by default in dbt). Rebuilds ENTIRE table every run (DROP + CREATE). Fast to query, slow to build on large data. Use for: final marts, small-medium datasets |
| **incremental** | `MERGE INTO / INSERT INTO` (only new/changed rows) — First run: CREATE TABLE (full build). Subsequent runs: only process new/changed data. Requires an `is_incremental()` filter. Use for: large fact tables, event data, logs |
| **ephemeral** | Nothing! (no object created in database) — Injected as a CTE into downstream models. Cannot be queried directly. Use for: reusable logic, intermediate calculations |

---

## TYPES OF VIEWS IN SNOWFLAKE & HOW TO SELECT THEM IN DBT

| VIEW TYPE | DESCRIPTION |
|-----------|-------------|
| **Regular View** | Stores only the SQL definition, no data. Query runs every time you SELECT from it. Cheapest (no storage cost). Slowest to query (recomputes every time). |
| **Secure View** | Same as regular but hides SQL definition. Other users CANNOT see the query logic. Used for data sharing & security. Slightly less optimized (optimizer can't peek inside). |
| **Materialized View** | Stores precomputed results (like a table). Auto-refreshes when base table changes. Fast to query. Costs storage + maintenance credits. LIMITED: only simple queries, no joins, no UDFs. |
| **Dynamic Table** | Not a view, but similar concept. Auto-refreshes on a schedule (TARGET_LAG). Supports complex queries (joins, aggregations). Use when materialized view is too limited. |

### How to select each type in dbt:

```sql
-- 1. REGULAR VIEW (default):
{{ config(materialized='view') }}

-- 2. SECURE VIEW:
{{ config(materialized='view', secure=true) }}

-- 3. MATERIALIZED VIEW:
{{ config(materialized='materialized_view') }}
-- NOTE: Snowflake restrictions - no joins, no subqueries, no window functions, no UDFs

-- 4. DYNAMIC TABLE:
{{ config(materialized='dynamic_table', target_lag='1 hour', snowflake_warehouse='COMPUTE_WH') }}
```

### When to use which view:

| Scenario | Best Choice |
|----------|-------------|
| Staging models (light transforms) | Regular View |
| Shared with other accounts/roles | Secure View |
| Simple aggregation on one table | Materialized View |
| Complex transforms needing freshness | Dynamic Table |
| Large fact tables (millions of rows) | Table (incremental) |

---

## TYPES OF TABLES IN SNOWFLAKE & HOW TO SELECT THEM IN DBT

> **DEFAULT:** In dbt-snowflake, `materialized='table'` creates a **TRANSIENT TABLE** by default (not permanent). Use `transient=false` to create a permanent table.

| TABLE TYPE | DESCRIPTION |
|------------|-------------|
| **Transient Table** | DEFAULT in dbt-snowflake adapter. No Fail-safe period (saves storage cost). Time Travel limited to 0 or 1 day. Use for intermediate/replaceable data. |
| **Permanent Table** | NOT default. Use: `transient=false`. Persists until explicitly dropped. Has Time Travel (default 1 day, up to 90 days). Has Fail-safe (7 days additional recovery). Full storage cost. |
| **Temporary Table** | Exists only for the session duration. Dropped automatically when session ends. No Time Travel, no Fail-safe. Not visible to other sessions/users. dbt does NOT support this directly. |
| **Dynamic Table** | Auto-refreshes based on target_lag. Snowflake manages the refresh pipeline. Supports complex SQL (joins, aggregations). Use instead of incremental for auto-refresh needs. |
| **Iceberg Table** | Open table format (Apache Iceberg). Interoperable with Spark, Flink, etc. Data stored in external cloud storage. Use for multi-engine/lakehouse architectures. |

### How to select each type in dbt:

```sql
-- 1. TRANSIENT TABLE (default in dbt-snowflake):
{{ config(materialized='table') }}

-- 2. PERMANENT TABLE:
{{ config(materialized='table', transient=false) }}

-- 3. DYNAMIC TABLE:
{{ config(
    materialized='dynamic_table',
    target_lag='1 hour',
    snowflake_warehouse='COMPUTE_WH'
) }}

-- 4. ICEBERG TABLE:
{{ config(
    materialized='table',
    table_format='iceberg',
    external_volume='my_external_volume',
    catalog='snowflake',
    base_location='my_iceberg_table'
) }}

-- 5. INCREMENTAL TABLE (append/merge pattern):
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}
```

Set transient globally in `dbt_project.yml`:
```yaml
models:
  my_project:
    staging:
      +transient: true       # default, explicit
    marts:
      +transient: false      # permanent tables
```

### Storage cost comparison:

| TABLE TYPE | TIME TRAVEL | FAIL-SAFE | RELATIVE COST |
|------------|-------------|-----------|---------------|
| Permanent | 1-90 days | 7 days | $$$ |
| Transient | 0-1 day | None | $$ |
| Temporary | 0-1 day | None | $ (session) |
| Dynamic | 1-90 days | 7 days | $$$ + compute |
| Iceberg | None* | None* | $$ (external) |

*Iceberg tables managed by Snowflake catalog do support Time Travel*

---

## SECTION 4: MODEL EXAMPLES BY MATERIALIZATION

### EXAMPLE 1: VIEW (Default)

**File:** `models/staging/stg_customers.sql`

```sql
{{ config(materialized='view') }}

SELECT
    id AS customer_id,
    TRIM(first_name) AS first_name,
    TRIM(last_name) AS last_name,
    LOWER(email) AS email,
    created_at
FROM {{ source('raw', 'customers') }}
```

**What dbt compiles this to:**

```sql
CREATE OR REPLACE VIEW DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS AS (
    SELECT
        id AS customer_id,
        TRIM(first_name) AS first_name,
        TRIM(last_name) AS last_name,
        LOWER(email) AS email,
        created_at
    FROM DBT_DEV_DB.DEV_RAW.CUSTOMERS
);
```

### EXAMPLE 2: TABLE

**File:** `models/marts/dim_customers.sql`

```sql
{{ config(materialized='table') }}

SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    COUNT(o.order_id) AS total_orders,
    SUM(o.amount) AS lifetime_value,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS last_order_date
FROM {{ ref('stg_customers') }} c
LEFT JOIN {{ ref('stg_orders') }} o ON c.customer_id = o.customer_id
GROUP BY 1, 2, 3, 4
```

**What dbt compiles this to:**

```sql
CREATE OR REPLACE TRANSIENT TABLE DBT_DEV_DB.DEV_STAGING.DIM_CUSTOMERS AS (
    SELECT
        c.customer_id,
        c.first_name,
        c.last_name,
        c.email,
        COUNT(o.order_id) AS total_orders,
        SUM(o.amount) AS lifetime_value,
        MIN(o.order_date) AS first_order_date,
        MAX(o.order_date) AS last_order_date
    FROM DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS c
    LEFT JOIN DBT_DEV_DB.DEV_STAGING.STG_ORDERS o ON c.customer_id = o.customer_id
    GROUP BY 1, 2, 3, 4
);
```

### EXAMPLE 3: INCREMENTAL

**File:** `models/marts/fct_orders.sql`

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT
    order_id,
    customer_id,
    order_date,
    amount,
    status,
    updated_at
FROM {{ source('raw', 'orders') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**FIRST RUN - dbt compiles to:**

```sql
CREATE OR REPLACE TRANSIENT TABLE DBT_DEV_DB.DEV_STAGING.FCT_ORDERS AS (
    SELECT order_id, customer_id, order_date, amount, status, updated_at
    FROM DBT_DEV_DB.DEV_RAW.ORDERS
    -- No WHERE clause on first run (builds full table)
);
```

**SUBSEQUENT RUNS - dbt compiles to:**

```sql
MERGE INTO DBT_DEV_DB.DEV_STAGING.FCT_ORDERS AS target
USING (
    SELECT order_id, customer_id, order_date, amount, status, updated_at
    FROM DBT_DEV_DB.DEV_RAW.ORDERS
    WHERE updated_at > (SELECT MAX(updated_at) FROM DBT_DEV_DB.DEV_STAGING.FCT_ORDERS)
) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET
    target.customer_id = source.customer_id,
    target.order_date = source.order_date,
    target.amount = source.amount,
    target.status = source.status,
    target.updated_at = source.updated_at
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, order_date, amount, status, updated_at)
VALUES (source.order_id, source.customer_id, source.order_date, source.amount, source.status, source.updated_at);
```

### EXAMPLE 4: EPHEMERAL

**File:** `models/staging/stg_order_amounts.sql`

```sql
{{ config(materialized='ephemeral') }}

SELECT
    order_id,
    amount,
    amount * 0.1 AS tax,
    amount * 1.1 AS total_with_tax
FROM {{ source('raw', 'orders') }}
```

**This creates NOTHING in the database!**

When another model references it:

```sql
-- File: models/marts/order_summary.sql
SELECT * FROM {{ ref('stg_order_amounts') }}
```

dbt injects it as a CTE:

```sql
CREATE VIEW DBT_DEV_DB.DEV_STAGING.ORDER_SUMMARY AS (
    WITH __dbt__cte__stg_order_amounts AS (
        SELECT
            order_id,
            amount,
            amount * 0.1 AS tax,
            amount * 1.1 AS total_with_tax
        FROM DBT_DEV_DB.DEV_RAW.ORDERS
    )
    SELECT * FROM __dbt__cte__stg_order_amounts
);
```

---

## SECTION 5: THE ref() FUNCTION (Model Dependencies)

`ref()` is the **MOST IMPORTANT** function in dbt. It does TWO things:
1. Resolves the model name to its actual database path
2. Builds a dependency (DAG edge) between models

**Example:**
```sql
{{ ref('stg_customers') }}
→ resolves to: DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS
→ AND tells dbt: "this model depends on stg_customers"
```

**WHY ref() matters:**
- dbt knows what order to run models
- dbt can build lineage graphs
- Paths automatically change between environments (dev/prod)
- You never hardcode table names

**DAG Example:**

```
source('raw','customers')  source('raw','orders')
       │                          │
       ▼                          ▼
stg_customers              stg_orders
       │                          │
       └──────────┬───────────────┘
                  ▼
           dim_customers
```

**Execution order:**
1. stg_customers (no model dependencies) ─┐ PARALLEL
2. stg_orders (no model dependencies)     ─┘
3. dim_customers (waits for both above)

---

## SECTION 6: THE source() FUNCTION

`source()` references raw tables that exist OUTSIDE your dbt project.
These are defined in a `sources.yml` file.

```yaml
# File: models/staging/sources.yml
version: 2
sources:
  - name: raw
    database: RAW_DB
    schema: PUBLIC
    tables:
      - name: customers
        description: "Raw customer data from application DB"
      - name: orders
        description: "Raw order data from application DB"
```

**Usage in model:**
```sql
{{ source('raw', 'customers') }}
→ resolves to: RAW_DB.PUBLIC.CUSTOMERS
```

**Benefits:**
- Centralized definition of raw tables
- dbt can test source freshness
- Clear boundary between "raw" and "transformed" data

---

## SECTION 7: MODEL CONFIGURATION (config block)

You configure models in THREE places (priority order):

| Priority | Location |
|----------|----------|
| 1 (Highest) | `config()` in the model file |
| 2 | YAML properties file |
| 3 (Lowest) | `dbt_project.yml` |

**Available config options for Snowflake:**

```sql
{{ config(
    materialized = 'table',          -- view/table/incremental/ephemeral
    schema = 'marts',                -- custom schema
    database = 'ANALYTICS_DB',       -- custom database
    alias = 'customers',             -- rename the output object
    tags = ['daily', 'finance'],     -- for selecting/filtering
    enabled = true,                  -- true/false to enable/disable
    pre_hook = "ALTER SESSION...",    -- SQL to run before model
    post_hook = "GRANT SELECT...",   -- SQL to run after model
    transient = true,                -- Snowflake transient table (default)
    cluster_by = ['date_column'],    -- Snowflake clustering key
    copy_grants = true,              -- preserve grants on rebuild
    secure = false                   -- create as secure view
) }}
```

---

## SECTION 8: SETTING CONFIG IN dbt_project.yml (Folder-Level)

Instead of repeating `config()` in every file, set it for entire folders:

```yaml
# dbt_project.yml:
models:
  my_project:
    staging:                    # applies to models/staging/
      +materialized: view
      +schema: staging
    intermediate:               # applies to models/intermediate/
      +materialized: ephemeral
    marts:                      # applies to models/marts/
      +materialized: table
      +schema: marts
      finance:                  # applies to models/marts/finance/
        +schema: finance
        +tags: ['finance', 'daily']
```

The `+` prefix means "apply this config to all models in this folder"

---

## SECTION 9: PROJECT STRUCTURE (Best Practice)

```
my_dbt_project/
├── dbt_project.yml
├── profiles.yml
├── packages.yml
├── macros/
│   ├── generate_schema_name.sql
│   └── custom_tests.sql
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── stg_customers.sql
│   │   ├── stg_orders.sql
│   │   └── stg_payments.sql
│   ├── intermediate/
│   │   └── int_customer_orders.sql
│   └── marts/
│       ├── schema.yml
│       ├── dim_customers.sql
│       └── fct_orders.sql
├── tests/
│   └── assert_positive_amounts.sql
├── seeds/
│   └── country_codes.csv
└── snapshots/
    └── scd_customers.sql
```

**Layer descriptions:**

| Layer | Purpose | Naming |
|-------|---------|--------|
| staging | 1:1 with sources, light cleaning | `stg_<source>_<table>` |
| intermediate | Business logic, joins | `int_<description>` |
| marts | Final tables for BI/analytics | `dim_` or `fct_` |

---

## SECTION 10: INCREMENTAL MODELS (Deep Dive)

### Why incremental?
- Table materialization rebuilds ALL data every run
- For 1 billion rows, that's expensive and slow
- Incremental only processes NEW or CHANGED data

### How it works:

```sql
{{ config(
    materialized='incremental',
    unique_key='event_id',         -- for deduplication
    incremental_strategy='merge'   -- merge/append/delete+insert
) }}

SELECT
    event_id,
    user_id,
    event_type,
    created_at
FROM {{ source('raw', 'events') }}

{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

### Key concepts:

| Concept | Explanation |
|---------|-------------|
| `is_incremental()` | Returns TRUE on subsequent runs (when table already exists) |
| `{{ this }}` | References the EXISTING table (to compare against) |
| `unique_key` | Column(s) used for MERGE matching |
| `--full-refresh` | Forces full rebuild (ignores is_incremental) |

### Incremental strategies:

| Strategy | Behavior | Use When |
|----------|----------|----------|
| `append` | Just INSERT new rows (no dedup) | Event logs, immutable data |
| `merge` | MERGE (update existing + insert new) | Mutable data with a unique key |
| `delete+insert` | DELETE matching rows, then INSERT | Complex updates, partitioned data |

---

## SECTION 11: SNAPSHOT MODELS (SCD Type 2)

Snapshots track how data changes over time (Slowly Changing Dimensions).

**File:** `snapshots/scd_customers.sql`

```sql
{% snapshot scd_customers %}
{{ config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at'
) }}

SELECT * FROM {{ source('raw', 'customers') }}

{% endsnapshot %}
```

**What this does:**
- First run: copies all rows, adds `dbt_valid_from`, `dbt_valid_to`
- Subsequent runs: if a row changes, marks old row with `dbt_valid_to = now()` and inserts new row with `dbt_valid_from = now()`

**Run with:** `dbt snapshot`

---

## SECTION 12: TESTING

### Schema tests (in YAML):

```yaml
models:
  - name: dim_customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: email
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              arguments:
                values: ['active', 'inactive', 'churned']
```

### Singular tests (custom SQL in tests/ folder):

**File:** `tests/assert_positive_amounts.sql`

```sql
SELECT *
FROM {{ ref('fct_orders') }}
WHERE amount < 0
```

If this returns ANY rows → test FAILS.

### Run tests:

```bash
dbt test                           # all tests
dbt test --select stg_customers    # tests for one model
dbt test --select tag:critical     # tagged tests only
```

---

## SECTION 13: SEEDS (Loading CSV Files)

Seeds are CSV files that dbt loads into your database as tables.

**File:** `seeds/country_codes.csv`
```csv
code,name,continent
US,United States,North America
UK,United Kingdom,Europe
IN,India,Asia
```

**Commands:**
```bash
dbt seed                    # load all CSVs
dbt seed --select country_codes  # load one
```

**Use in models:**
```sql
SELECT * FROM {{ ref('country_codes') }}
```

**Use cases:** lookup tables, mapping tables, static reference data

---

## SECTION 14: DAG (Directed Acyclic Graph) & Execution Order

```
SOURCES                STAGING              MARTS
─────────────         ────────────         ────────────

raw.customers ──→ stg_customers ──┐
                                   ├──→ dim_customers
raw.orders ────→ stg_orders ──────┤
                                   ├──→ fct_orders
raw.payments ──→ stg_payments ────┘
```

**Execution (with threads: 4):**
- Batch 1: stg_customers, stg_orders, stg_payments (parallel)
- Batch 2: dim_customers, fct_orders (parallel, after batch 1)

**threads setting in profiles.yml:**
- `threads: 4` → up to 4 models run simultaneously
- Higher threads = faster builds (if warehouse can handle it)

---

## SECTION 15: MODEL PROPERTIES & DOCUMENTATION

Define metadata, descriptions, and tests in YAML:

```yaml
# File: models/marts/schema.yml
version: 2
models:
  - name: dim_customers
    description: "Customer dimension with lifetime metrics"
    columns:
      - name: customer_id
        description: "Primary key from source system"
        tests:
          - unique
          - not_null
      - name: lifetime_value
        description: "Total spend across all orders"
        tests:
          - not_null
```

**Commands:**
```bash
dbt docs generate   # creates documentation site
dbt docs serve      # opens in browser with lineage graph
```

---

## SECTION 16: MODEL HOOKS (pre-hook & post-hook)

Hooks run SQL before or after a model builds.

**Use cases:**
- Grant permissions after table creation
- Run ALTER SESSION before model
- Insert into audit log after model runs

```sql
{{ config(
    materialized='table',
    post_hook=[
        "GRANT SELECT ON {{ this }} TO ROLE ANALYST_ROLE",
        "GRANT SELECT ON {{ this }} TO ROLE BI_ROLE"
    ]
) }}

SELECT * FROM {{ ref('stg_customers') }}
```

After dbt creates the table, it automatically runs:
```sql
GRANT SELECT ON DBT_DEV_DB.DEV_STAGING.DIM_CUSTOMERS TO ROLE ANALYST_ROLE;
GRANT SELECT ON DBT_DEV_DB.DEV_STAGING.DIM_CUSTOMERS TO ROLE BI_ROLE;
```

---

## SECTION 17: THE {{ this }} VARIABLE

`{{ this }}` refers to the current model's relation (database.schema.table).

Most commonly used in:
1. Incremental models (to reference existing data)
2. Post-hooks (to grant on the just-created object)

**In an incremental model:**
```sql
{{ this }} → DBT_DEV_DB.DEV_STAGING.FCT_ORDERS (the existing table)

{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

---

## SECTION 18: ALIASES (Renaming Output Objects)

By default, dbt names the output object after the filename:
- `stg_customers.sql` → `STG_CUSTOMERS`

To override:

```sql
{{ config(alias='customers') }}
SELECT * FROM {{ source('raw', 'customers') }}
```

**Result:** `DBT_DEV_DB.DEV_STAGING.CUSTOMERS` (not STG_CUSTOMERS)

**Use case:** Your BI tool expects "CUSTOMERS" but your file naming convention requires "stg_customers.sql"

---

## SECTION 19: TAGS (Organizing & Selecting Models)

```sql
-- In model file:
{{ config(tags=['daily', 'finance']) }}
```

```yaml
# In dbt_project.yml:
models:
  my_project:
    marts:
      finance:
        +tags: ['finance', 'daily']
```

**Running tagged models:**
```bash
dbt run --select tag:daily       # all daily models
dbt run --select tag:finance     # all finance models
dbt test --select tag:critical   # test critical models only
```

**Common tag patterns:**
- Frequency: `hourly`, `daily`, `weekly`
- Domain: `finance`, `marketing`, `product`
- Priority: `critical`, `standard`, `experimental`

---

## SECTION 20: FULL WORKING EXAMPLE (End-to-End)

**STEP 1:** Source definition (`models/staging/sources.yml`)
```yaml
version: 2
sources:
  - name: raw_data
    database: DBT_DEV_DB
    schema: DEV_RAW
    tables:
      - name: customers
```

**STEP 2:** Staging model (`models/staging/stg_customers.sql`)
```sql
{{ config(materialized='view') }}
SELECT
    customer_id,
    TRIM(first_name) AS first_name,
    TRIM(last_name) AS last_name,
    LOWER(email) AS email,
    city,
    created_at
FROM {{ source('raw_data', 'customers') }}
```

**STEP 3:** Mart model (`models/marts/dim_customers.sql`)
```sql
{{ config(materialized='table') }}
SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    email,
    city,
    created_at,
    DATEDIFF('day', created_at, CURRENT_DATE()) AS days_since_signup
FROM {{ ref('stg_customers') }}
```

**STEP 4:** Run it!
```bash
dbt run
```

**Output:**
```
Running 1 of 2: stg_customers (view)....... OK
Running 2 of 2: dim_customers (table)...... OK
Finished running 1 view, 1 table in 5.2s
Completed successfully. 2 models, 0 errors.
```

---

## SECTION 21: COMMON COMMANDS REFERENCE

| Command | What it does |
|---------|-------------|
| `dbt run` | Build all models |
| `dbt run --select model_name` | Build one model |
| `dbt run --select +model_name` | Build model + all ancestors |
| `dbt run --full-refresh` | Rebuild incremental models fully |
| `dbt test` | Run all tests |
| `dbt build` | Run + test (in DAG order) |
| `dbt compile` | Generate SQL without executing |
| `dbt compile --select model_name` | See compiled SQL for one model |
| `dbt list` | List all resources |
| `dbt list --select tag:daily` | List tagged resources |
| `dbt docs generate` | Generate documentation |
| `dbt seed` | Load CSV files from seeds/ |
| `dbt snapshot` | Run SCD Type 2 snapshots |

---

## SECTION 22: WHEN TO USE WHICH MATERIALIZATION

| Scenario | Best Choice | Reason | Example |
|----------|-------------|--------|---------|
| < 10M rows | table | Fast rebuild | dim_customers |
| > 100M rows | incremental | Only process new | fct_page_views |
| Light ETL | view | No storage cost | stg_customers |
| Reusable CTE | ephemeral | No DB object | int_calc_tax |
| Always fresh | view | Real-time data | current_inventory |
| BI dashboard | table | Fast queries | revenue_summary |
| Event stream | incremental | Append only | fct_clicks |
| Temp logic | ephemeral | Clean up DAG | helper_date_spine |

---

## SECTION 23: COMMON MISTAKES & TROUBLESHOOTING

| Mistake | Problem | Fix |
|---------|---------|-----|
| Hardcoding table names | Breaks across environments and breaks DAG | Use `{{ ref('model') }}` |
| Circular dependencies | model_a refs model_b, model_b refs model_a | Restructure logic to break the cycle |
| Using `SELECT *` in production | Doesn't catch schema changes | List explicit columns |
| Forgetting `is_incremental()` filter | Reprocesses ALL data every run | Add WHERE clause with `{% if is_incremental() %}` |
| Wrong incremental strategy | Using 'append' when data can be updated → duplicates | Use 'merge' with unique_key for mutable data |
| Too many table materializations | Slow builds, high storage costs | Use views for staging, ephemeral for intermediate |

---

## END OF GUIDE

**Summary:**
- A dbt model = a SELECT statement in a .sql file
- dbt wraps it in DDL based on materialization
- `ref()` builds dependencies → DAG → execution order
- `source()` points to raw external tables
- 4 materializations: view, table, incremental, ephemeral
- dbt-snowflake creates TRANSIENT tables by default
- Config can be set in: model file, YAML, dbt_project.yml
- Use staging → intermediate → marts layering
- Test everything with schema tests + singular tests
