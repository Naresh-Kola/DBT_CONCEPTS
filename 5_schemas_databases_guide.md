# DBT Schemas & Databases - Step by Step Flow

## How dbt Decides Where to Create Your Tables

---

## Step 1: Understand the Starting Point

Before dbt runs ANY model, it needs to answer ONE question:

> "WHERE should I create this table in Snowflake?"

The answer is always: `DATABASE.SCHEMA.TABLE_NAME`

To get this answer, dbt looks at TWO files:

- **FILE 1:** `profiles.yml` (your connection + defaults)
- **FILE 2:** `dbt_project.yml` OR model config (your customizations)

---

## Step 2: What profiles.yml Provides (The Defaults)

```yaml
default:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: zhb62162
      user: NARESHKOLA4545
      role: ACCOUNTADMIN
      database: DBT_DEV_DB       # ← DEFAULT DATABASE
      warehouse: COMPUTE_WH
      schema: DEV                # ← DEFAULT SCHEMA (called "target schema")
      threads: 4
```

From this file, dbt knows:
- Default database = `DBT_DEV_DB`
- Default schema = `DEV`
- Environment name = `dev` (target.name)

---

## Step 3: Flow Without Any Custom Schema (Simplest Case)

**Model file:** `models/staging/stg_orders.sql`

```sql
SELECT ORDER_ID, CUSTOMER_ID, AMOUNT
FROM {{ source('raw', 'orders') }}
```

No `config()` block. No custom schema. Nothing extra.

### Execution Flow:

| Step | Action | Result |
|------|--------|--------|
| A | dbt reads `stg_orders.sql` | No config found. No custom schema. No custom database |
| B | dbt asks: "What database?" | No custom database in config → Uses profiles.yml default → `DBT_DEV_DB` |
| C | dbt asks: "What schema?" | No custom schema in config → Uses profiles.yml default → `DEV` |
| D | dbt asks: "What table name?" | Uses filename → `STG_ORDERS` |
| E | dbt constructs path | `DBT_DEV_DB.DEV.STG_ORDERS` |
| F | dbt runs in Snowflake | `CREATE SCHEMA IF NOT EXISTS DBT_DEV_DB.DEV;` <br> `CREATE VIEW DBT_DEV_DB.DEV.STG_ORDERS AS (...)` |

**Result in Snowflake:**
```
DBT_DEV_DB (database)
  └── DEV (schema)
       └── STG_ORDERS (view)
```

---

## Step 4: Flow With Custom Schema in Config

**Model file:** `models/staging/stg_orders.sql`

```sql
{{ config(schema='staging') }}

SELECT ORDER_ID, CUSTOMER_ID, AMOUNT
FROM {{ source('raw', 'orders') }}
```

### Execution Flow:

| Step | Action | Result |
|------|--------|--------|
| A | dbt reads `stg_orders.sql` | Finds config: `schema='staging'` |
| B | dbt asks: "What database?" | No custom database → Uses default → `DBT_DEV_DB` |
| C | dbt asks: "What schema?" | Custom schema found: `'staging'` → calls `generate_schema_name('staging', node)` |
| D | Inside `generate_schema_name` | Input: `custom_schema_name = 'staging'` <br> `default_schema = 'DEV'` <br> Is custom_schema_name none? → NO <br> Concatenate: `'DEV' + '_' + 'staging'` <br> Output: `'DEV_STAGING'` |
| E | dbt constructs path | `DBT_DEV_DB.DEV_STAGING.STG_ORDERS` |
| F | dbt runs in Snowflake | `CREATE SCHEMA IF NOT EXISTS DBT_DEV_DB.DEV_STAGING;` <br> `CREATE VIEW DBT_DEV_DB.DEV_STAGING.STG_ORDERS AS (...)` |

**Result in Snowflake:**
```
DBT_DEV_DB (database)
  └── DEV_STAGING (schema) ← NOT just "STAGING"!
       └── STG_ORDERS (view)
```

**WHY "DEV_STAGING" and not just "STAGING"?**
- The DEFAULT macro ALWAYS prepends the target schema
- Formula: `target_schema + "_" + custom_schema`
- `DEV + "_" + staging = DEV_STAGING`

---

## Step 5: The Default generate_schema_name Macro (What dbt Ships With)

```sql
{% macro generate_schema_name(custom_schema_name, node) %}

    set default_schema = target.schema
    (this comes from profiles.yml → schema field)

    IF custom_schema_name is none:
        return default_schema
        (just use whatever is in profiles.yml)

    ELSE:
        return default_schema + "_" + custom_schema_name
        (concatenate them together)

{% endmacro %}
```

### Flow Diagram:

```
Model has config(schema='staging')
     │
     ▼
generate_schema_name('staging', node) is called
     │
     ▼
┌─────────────────────────────┐
│ default_schema = 'DEV'      │  ← from profiles.yml
└─────────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│ Is 'staging' = none?        │
│         NO                  │
└─────────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│ return 'DEV' + '_' +        │
│        'staging'            │
│      = 'DEV_STAGING'        │
└─────────────────────────────┘
     │
     ▼
Table goes to: DBT_DEV_DB.DEV_STAGING.STG_ORDERS
```

---

## Step 6: Overriding the Macro (Custom Schema Logic)

**Problem:** In production, we want schema = `'STAGING'` (not `'PROD_STAGING'`)

**Solution:** Create file `macros/generate_schema_name.sql` with YOUR logic:

```sql
{% macro generate_schema_name(custom_schema_name, node) %}

    set default_schema = target.schema

    IF custom_schema_name is none:
        return default_schema

    ELIF target.name == 'prod':
        return custom_schema_name   ← JUST the custom name

    ELSE:
        return default_schema + "_" + custom_schema_name

{% endmacro %}
```

### Flow in DEV (target.name = 'dev', target.schema = 'DEV'):

```
config(schema='staging')
     │
     ▼
generate_schema_name('staging', node)
     │
     ▼
default_schema = 'DEV'
     │
     ▼
Is 'staging' none? → NO
     │
     ▼
Is target.name == 'prod'? → NO (it's 'dev')
     │
     ▼
return 'DEV' + '_' + 'staging' = 'DEV_STAGING'
     │
     ▼
Table: DBT_DEV_DB.DEV_STAGING.STG_ORDERS ✓
```

### Flow in PROD (target.name = 'prod', target.schema = 'PROD'):

```
config(schema='staging')
     │
     ▼
generate_schema_name('staging', node)
     │
     ▼
default_schema = 'PROD'
     │
     ▼
Is 'staging' none? → NO
     │
     ▼
Is target.name == 'prod'? → YES!
     │
     ▼
return 'staging' (just the custom name, no prefix!)
     │
     ▼
Table: PROD_DB.STAGING.STG_ORDERS ✓
```

### Actual Macro Code:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- elif target.name == 'prod' -%}
        {{ custom_schema_name | trim }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

---

## Step 7: Setting Schema via dbt_project.yml (No config() in Each File)

Instead of writing `config(schema='staging')` in EVERY model file, set it ONCE in `dbt_project.yml` for entire folders:

```yaml
models:
  my_project:
    staging:          # ← folder name: models/staging/
      +schema: staging
    intermediate:     # ← folder name: models/intermediate/
      +schema: intermediate
    marts:            # ← folder name: models/marts/
      +schema: marts
```

### Flow:

```
dbt finds: models/staging/stg_orders.sql
     │
     ▼
dbt checks: does model file have config(schema=...)? → NO
     │
     ▼
dbt checks: does dbt_project.yml have +schema for this folder? → YES: 'staging'
     │
     ▼
dbt calls: generate_schema_name('staging', node)
     │
     ▼
(same flow as before)
     │
     ▼
In dev: DBT_DEV_DB.DEV_STAGING.STG_ORDERS
In prod: PROD_DB.STAGING.STG_ORDERS
```

### Priority Order (if same model has multiple schema settings):

| Priority | Source | |
|----------|--------|-|
| 1 (highest) | `config()` in model file | WINS |
| 2 | config in YAML properties | |
| 3 | `+schema` in `dbt_project.yml` | |
| 4 (lowest) | `profiles.yml` default | LOSES |

---

## Step 8: Database — How dbt Decides Which Database

Same concept as schema, but for DATABASE.

**Default behavior:** ALL models go to the database in `profiles.yml` (`DBT_DEV_DB`)

**To use a custom database:**

```sql
{{ config(database='ANALYTICS_DB', schema='marts') }}
SELECT * FROM {{ ref('stg_orders') }}
```

### Flow With Custom Database:

```
Model: fct_orders.sql
Config: database='ANALYTICS_DB', schema='marts'
     │
     ▼
STEP 1: Resolve Database
  dbt calls: generate_database_name('ANALYTICS_DB')
  Is custom_database none? → NO
  Return: 'ANALYTICS_DB'
     │
     ▼
STEP 2: Resolve Schema
  dbt calls: generate_schema_name('marts')
  In dev: DEV + '_' + marts = DEV_MARTS
  In prod: marts (just the custom name)
     │
     ▼
STEP 3: Resolve Table Name
  Filename: fct_orders.sql → FCT_ORDERS
     │
     ▼
STEP 4: Construct Full Path
  In dev:  ANALYTICS_DB.DEV_MARTS.FCT_ORDERS
  In prod: ANALYTICS_DB.MARTS.FCT_ORDERS
     │
     ▼
STEP 5: Create Schema & Table
  CREATE SCHEMA IF NOT EXISTS ANALYTICS_DB.DEV_MARTS;
  CREATE TABLE ANALYTICS_DB.DEV_MARTS.FCT_ORDERS AS (...);
```

**Important difference:**
- **Schema:** dbt creates automatically (`CREATE SCHEMA IF NOT EXISTS`)
- **Database:** dbt does NOT create. You must create it yourself first!

```sql
CREATE DATABASE IF NOT EXISTS ANALYTICS_DB;
CREATE DATABASE IF NOT EXISTS RAW_DB;
```

---

## Step 9: Custom Database Macro (generate_database_name)

**Why override?**
- In DEV: All models should go to `DBT_DEV_DB` (ignore custom database)
- In PROD: Models should go to their specified databases

```
macros/generate_database_name.sql

{% macro generate_database_name(custom_database_name) %}

    IF custom_database_name is none:
        return target.database (from profiles.yml)

    ELIF target.name == 'prod':
        return custom_database_name (use what model says)

    ELSE: (dev environment)
        return target.database (ignore custom, use default)

{% endmacro %}
```

### Trace in DEV:

```
Model config: database='ANALYTICS_DB'
target.name = 'dev'
target.database = 'DBT_DEV_DB'

→ Is custom none? NO
→ Is target.name 'prod'? NO
→ Return target.database = 'DBT_DEV_DB'

RESULT: DBT_DEV_DB.DEV_MARTS.FCT_ORDERS (all in one dev database!)
```

### Trace in PROD:

```
Model config: database='ANALYTICS_DB'
target.name = 'prod'
target.database = 'PROD_DB'

→ Is custom none? NO
→ Is target.name 'prod'? YES!
→ Return custom_database_name = 'ANALYTICS_DB'

RESULT: ANALYTICS_DB.MARTS.FCT_ORDERS (goes to actual database!)
```

### Actual Macro Code:

```sql
{% macro generate_database_name(custom_database_name, node) -%}
    {%- set default_database = target.database -%}
    {%- if custom_database_name is none -%}
        {{ default_database }}
    {%- elif target.name == 'prod' -%}
        {{ custom_database_name | trim }}
    {%- else -%}
        {{ default_database }}
    {%- endif -%}
{%- endmacro %}
```

---

## Step 10: Creating Database via on-run-start Hook

Since dbt won't auto-create databases, use hooks:

```yaml
# dbt_project.yml
on-run-start:
  - "CREATE DATABASE IF NOT EXISTS {{ target.database }}"
  - "CREATE DATABASE IF NOT EXISTS ANALYTICS_DB"
  - "CREATE DATABASE IF NOT EXISTS RAW_DB"
```

### Flow:

```
You run: dbt run
     │
     ▼
PHASE 1: on-run-start hooks execute
  → CREATE DATABASE IF NOT EXISTS DBT_DEV_DB
  → CREATE DATABASE IF NOT EXISTS ANALYTICS_DB
  → CREATE DATABASE IF NOT EXISTS RAW_DB
     │
     ▼
PHASE 2: Models start building
  → stg_orders builds
  → stg_customers builds
  → fct_orders builds
  (databases already exist from Phase 1)
     │
     ▼
PHASE 3: on-run-end hooks execute
  (if you defined any)
```

### Or Use a Macro to Create Databases Dynamically:

**File:** `macros/create_databases.sql`

```sql
{% macro create_databases() %}
    {% set databases = ['RAW_DB', 'ANALYTICS_DB', 'REPORTING_DB'] %}
    {% for db in databases %}
        {% set create_sql %}
            CREATE DATABASE IF NOT EXISTS {{ db }}
        {% endset %}
        {% do run_query(create_sql) %}
    {% endfor %}
{% endmacro %}
```

**dbt_project.yml:**

```yaml
on-run-start:
  - "{{ create_databases() }}"
```

---

## Step 11: Complete Flow — Multiple Models, One dbt run

### Project Structure:

```
models/
├── staging/
│   ├── stg_orders.sql         (no config, uses dbt_project.yml)
│   └── stg_customers.sql      (no config, uses dbt_project.yml)
└── marts/
    └── fct_orders.sql         (no config, uses dbt_project.yml)
```

### dbt_project.yml:

```yaml
models:
  my_project:
    staging:
      +schema: staging
      +materialized: view
    marts:
      +schema: marts
      +materialized: table
```

### Execution (dbt run):

**Phase 1: Parse**
- dbt reads ALL files in the project
- Finds 3 models
- Reads `dbt_project.yml` for folder configs
- Builds DAG:
  ```
  stg_orders ──┐
               ├──→ fct_orders
  stg_customers┘
  ```

**Phase 2: Resolve Paths**

| Model | Database | Schema | Full Path |
|-------|----------|--------|-----------|
| stg_orders | `generate_database_name(none)` → `DBT_DEV_DB` | `generate_schema_name('staging')` → `DEV_STAGING` | `DBT_DEV_DB.DEV_STAGING.STG_ORDERS` |
| stg_customers | `generate_database_name(none)` → `DBT_DEV_DB` | `generate_schema_name('staging')` → `DEV_STAGING` | `DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS` |
| fct_orders | `generate_database_name(none)` → `DBT_DEV_DB` | `generate_schema_name('marts')` → `DEV_MARTS` | `DBT_DEV_DB.DEV_MARTS.FCT_ORDERS` |

**Phase 3: Execute (in DAG order)**

1. `CREATE SCHEMA IF NOT EXISTS DBT_DEV_DB.DEV_STAGING;`
2. `CREATE VIEW DBT_DEV_DB.DEV_STAGING.STG_ORDERS AS (...)` — runs in PARALLEL with:
   `CREATE VIEW DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS AS (...)`
3. `CREATE SCHEMA IF NOT EXISTS DBT_DEV_DB.DEV_MARTS;`
4. `CREATE TABLE DBT_DEV_DB.DEV_MARTS.FCT_ORDERS AS (...)` — runs AFTER staging models

### Final Result in Snowflake:

```
DBT_DEV_DB
  ├── DEV_STAGING
  │    ├── STG_ORDERS (view)
  │    └── STG_CUSTOMERS (view)
  └── DEV_MARTS
       └── FCT_ORDERS (table)
```

---

## Step 12: Same Project in Prod — What Changes?

**profiles.yml for prod:**

```yaml
default:
  target: prod
  outputs:
    prod:
      type: snowflake
      database: PROD_DB
      schema: PROD
      ...
```

With our custom `generate_schema_name` macro:

| Model | DEV Result | PROD Result |
|-------|-----------|-------------|
| stg_orders | `DBT_DEV_DB.DEV_STAGING.STG_ORDERS` | `PROD_DB.STAGING.STG_ORDERS` |
| stg_customers | `DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS` | `PROD_DB.STAGING.STG_CUSTOMERS` |
| fct_orders | `DBT_DEV_DB.DEV_MARTS.FCT_ORDERS` | `PROD_DB.MARTS.FCT_ORDERS` |

- **DEV:** schemas have prefix (`DEV_STAGING`, `DEV_MARTS`)
- **PROD:** schemas are clean (`STAGING`, `MARTS`)

This is the power of the custom macro! Same code, different behavior per environment.

---

## Step 13: Key Differences — Schema vs Database

| Question | Schema | Database |
|----------|--------|----------|
| Does dbt auto-create it? | YES | NO |
| Has a generate macro? | `generate_schema_name` | `generate_database_name` |
| Default behavior with custom? | PREPENDS target (`DEV_staging`) | USES custom directly |
| Set per model? | `config(schema=)` | `config(database=)` |
| Set per folder? | `+schema` in `dbt_project.yml` | `+database` in `dbt_project.yml` |
| Override file location? | `macros/generate_schema_name.sql` | `macros/generate_database_name.sql` |

---

## Step 14: Troubleshooting Flow

### "My model went to the WRONG schema!"

```
CHECK THIS FLOW:
     │
     ▼
1. Does model have config(schema=...)?
   YES → that value is used
   NO → go to step 2
     │
     ▼
2. Does YAML properties have schema?
   YES → that value is used
   NO → go to step 3
     │
     ▼
3. Does dbt_project.yml have +schema for this model's folder?
   YES → that value is used
   NO → go to step 4
     │
     ▼
4. Use profiles.yml default schema (target.schema)
     │
     ▼
5. Pass value to generate_schema_name macro → get final schema name
     │
     ▼
6. Is there a custom macro in macros/generate_schema_name.sql?
   YES → uses YOUR logic
   NO → uses DEFAULT (prepend target)
```

### "dbt says database doesn't exist!"

```
dbt does NOT create databases!

Fix: Run this FIRST in Snowflake:
  CREATE DATABASE IF NOT EXISTS <name>;

Or add to on-run-start hook
```
