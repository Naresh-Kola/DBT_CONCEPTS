# _SOURCES.YML - COMPLETE GUIDE

**What it is, How to write it, Every property explained with examples**

---

## 1. WHAT IS _sources.yml?

`_sources.yml` is a YAML configuration file where you tell dbt about tables that **ALREADY EXIST** in your database.

These tables are **NOT** created by dbt. They were loaded by some other process — a pipeline (Fivetran, Airbyte), a `COPY INTO` command, an `INSERT` statement, or any other ETL tool.

dbt calls these tables **"SOURCES"** because they are the SOURCE of your data — the starting point of your dbt pipeline.

### Without _sources.yml:
- dbt has **NO** idea your raw tables exist.
- You'd have to hardcode table names in your models.
- No lineage, no freshness, no documentation.

### With _sources.yml:
- dbt **KNOWS** about your raw tables.
- You use `{{ source() }}` to reference them.
- Full lineage, freshness checks, and documentation.

---

## 2. WHERE DOES _sources.yml LIVE?

It goes inside your `models/` directory. Convention is to place it in the `staging/` folder because staging models are the first to read from raw tables.

```
dbt_project/
|-- dbt_project.yml
|-- profiles.yml
|-- models/
|   |-- staging/
|   |   |-- _sources.yml          <-- HERE
|   |   |-- stg_customers.sql
|   |-- intermediate/
|   |-- marts/
|-- macros/
```

### File Naming Conventions:

| File Name                | Notes                                  |
|--------------------------|----------------------------------------|
| `_sources.yml`           | Most common (underscore prefix)        |
| `_staging_sources.yml`   | If you have multiple source files      |
| `sources.yml`            | Also valid (no underscore)             |
| `schema.yml`             | Older convention (still works)         |

The underscore prefix `_` is just a human convention meaning "this is a config file, not a model." dbt does not treat it specially.

You **CAN** have **MULTIPLE** source files. dbt reads ALL `.yml` files in the `models/` directory (and subdirectories). So you could have:
- `models/staging/_sources.yml` (customer/order sources)
- `models/staging/_external_sources.yml` (API data sources)

---

## 3. COMPLETE _sources.yml EXAMPLE

Here is a full `_sources.yml` file for our CUSTOMERS table in `DBT_PROJECT_DB.RAW`:

**FILE:** `models/staging/_sources.yml`

```yaml
version: 2

sources:
  - name: raw
    description: "Raw data loaded into the RAW schema of DBT_PROJECT_DB"
    database: DBT_PROJECT_DB
    schema: RAW

    freshness:
      warn_after:
        count: 24
        period: hour
      error_after:
        count: 48
        period: hour
    loaded_at_field: _loaded_at

    tables:
      - name: customers
        description: "Customer registration data - 50 records loaded manually"
        columns:
          - name: customer_id
            description: "Unique numeric identifier for each customer (1-50)"
            tests:
              - unique
              - not_null
          - name: first_name
            description: "Customer's first name"
            tests:
              - not_null
          - name: last_name
            description: "Customer's last name"
            tests:
              - not_null
          - name: email
            description: "Customer's email address"
            tests:
              - unique
              - not_null
          - name: signup_date
            description: "Date when the customer registered"
            tests:
              - not_null
          - name: country
            description: "Customer's country of residence"
            tests:
              - not_null
              - accepted_values:
                  values: ['India', 'United States', 'United Kingdom', 'Canada', 'Australia']
          - name: _loaded_at
            description: "Timestamp when the row was inserted into the table"
```

---

## 4. EVERY PROPERTY EXPLAINED (Line by Line)

### 4A. `version: 2`

```yaml
version: 2
```

Tells dbt which `.yml` specification format to use. Always use version 2 — it is the current and only supported version. If you omit this, dbt may throw a warning or error.

### 4B. `sources:` (Top-Level Key)

```yaml
sources:
```

This is the **ROOT KEY** that contains all source definitions. Everything under here defines source groups and their tables. You can define multiple source groups under this key.

**Example with multiple source groups:**

```yaml
sources:
  - name: raw              # First source group
    schema: RAW
    tables: ...
  - name: external_api     # Second source group
    schema: API_DATA
    tables: ...
```

### 4C. `- name: raw` (Source Group Name)

```yaml
- name: raw
```

This is the **SOURCE GROUP NAME**. It is an identifier YOU choose. It groups related tables together.

**THIS NAME IS USED IN YOUR MODELS:**

```sql
{{ source('raw', 'customers') }}
         ^^^^^
         This comes from "name: raw"
```

**Naming Convention:** Name it after the schema or data origin:
- `name: raw` — tables in RAW schema
- `name: salesforce` — tables loaded from Salesforce
- `name: stripe` — tables loaded from Stripe
- `name: google_analytics` — tables loaded from GA

> **IMPORTANT:** The source group name does NOT have to match the schema name. But if it doesn't, you **MUST** specify the `schema:` property explicitly.

**CASE 1:** name matches schema (`schema:` is optional)
```yaml
- name: raw             # dbt assumes schema = RAW
  tables: ...
```

**CASE 2:** name does NOT match schema (`schema:` is required)
```yaml
- name: salesforce      # dbt would assume schema = SALESFORCE (wrong!)
  schema: RAW           # You MUST tell dbt the actual schema
  tables: ...
```

### 4D. `description:` (Source Group Level)

```yaml
description: "Raw data loaded into the RAW schema of DBT_PROJECT_DB"
```

**OPTIONAL** but recommended. A human-readable description of this source group. Shows up in dbt docs (the auto-generated documentation website).

You can write multi-line descriptions using YAML syntax:

```yaml
description: >
  Raw data loaded into DBT_PROJECT_DB.RAW by our ETL pipeline.
  Data is refreshed daily at 2 AM UTC.
  Contact: data-engineering@company.com
```

### 4E. `database: DBT_PROJECT_DB`

```yaml
database: DBT_PROJECT_DB
```

The Snowflake **DATABASE** where the source tables live.

**When to use:**
- If your source tables are in a **DIFFERENT** database than your `profiles.yml` database, you **MUST** specify this.
- If they are in the **SAME** database, you can omit it (dbt uses the database from `profiles.yml` by default).

**Example:**
- `profiles.yml` says `database: DBT_PROJECT_DB`, source tables are in `DBT_PROJECT_DB` → You can **OMIT** `database:` (same database)
- `profiles.yml` says `database: DBT_PROJECT_DB`, source tables are in `RAW_DATABASE` → You **MUST** specify `database: RAW_DATABASE`

> **Recommendation:** Always include it for clarity, even if it matches.

### 4F. `schema: RAW`

```yaml
schema: RAW
```

The Snowflake **SCHEMA** where the source tables live.

**Behavior:**
- If specified: dbt uses this schema.
- If NOT specified: dbt uses the source group NAME as the schema.

**Example:**
```yaml
- name: raw              # If schema: is omitted, dbt uses "RAW"
  tables: ...               # (converts name to uppercase)

- name: salesforce       # If schema: is omitted, dbt uses "SALESFORCE"
  schema: RAW            # But we want RAW, so we specify it
  tables: ...
```

**Combined with `database:`,** dbt resolves the full path:
`database: DBT_PROJECT_DB` + `schema: RAW` = `DBT_PROJECT_DB.RAW`

### 4G. `freshness:` (Source Freshness Configuration)

```yaml
freshness:
  warn_after:
    count: 24
    period: hour
  error_after:
    count: 48
    period: hour
```

Source freshness lets dbt **CHECK** if your raw data is up to date. This is optional but extremely useful for catching pipeline failures.

**How it works:**
1. You run: `dbt source freshness`
2. dbt queries the source table for `MAX(loaded_at_field)`
3. dbt calculates how old the most recent data is
4. dbt compares that age against your thresholds

**`warn_after`:**
If the newest data is MORE than 24 hours old, dbt gives a **WARNING**. The pipeline is probably just delayed. Not critical yet.

```
dbt output: "WARN: source raw.customers is 26 hours old"
```

**`error_after`:**
If the newest data is MORE than 48 hours old, dbt gives an **ERROR**. Something is definitely broken upstream.

```
dbt output: "ERROR: source raw.customers is 50 hours old"
```

**Available periods:** `minute`, `hour`, `day`

**Examples:**
```yaml
warn_after:  { count: 30, period: minute }   # Warn after 30 min
error_after: { count: 1, period: day }        # Error after 1 day
```

**Scope:** Defined at source group level → applies to ALL tables in the group. You can also override it per table (see section 6).

### 4H. `loaded_at_field: _loaded_at`

```yaml
loaded_at_field: _loaded_at
```

Tells dbt **WHICH COLUMN** to check for freshness. This column should contain the timestamp of when data was last loaded.

When dbt runs freshness check, it generates this SQL:

```sql
SELECT MAX(_loaded_at) AS max_loaded_at
FROM DBT_PROJECT_DB.RAW.CUSTOMERS
```

Then it checks: is `max_loaded_at` older than 24 hours? 48 hours?

**Common column names for this:**
- `_loaded_at`
- `_etl_loaded_at`
- `updated_at`
- `_fivetran_synced` (if using Fivetran)
- `_airbyte_extracted` (if using Airbyte)

> **IMPORTANT:** If you define `freshness:` but NOT `loaded_at_field:`, dbt will throw an error. Both must be present together.

### 4I. `tables:` (Table Definitions)

```yaml
tables:
  - name: customers
```

This lists the **ACTUAL TABLES** in the source schema. Each table is defined with `- name: <table_name>`.

The name **MUST** match the actual Snowflake table name (case-insensitive).

**THIS NAME IS USED IN YOUR MODELS:**

```sql
{{ source('raw', 'customers') }}
                  ^^^^^^^^^^^
                  This comes from "- name: customers"
```

**Full resolution:**
`database: DBT_PROJECT_DB` + `schema: RAW` + `name: customers` = `DBT_PROJECT_DB.RAW.CUSTOMERS`

**Multiple tables:**

```yaml
tables:
  - name: customers
  - name: orders
  - name: products
  - name: payments
```

### 4J. Table-Level `description:`

```yaml
- name: customers
  description: "Customer registration data - 50 records loaded manually"
```

Optional. Describes what this specific table contains. Shows up in dbt docs under the source lineage.

### 4K. `columns:` (Column Definitions)

```yaml
columns:
  - name: customer_id
    description: "Unique numeric identifier for each customer"
```

Lists columns with descriptions. This is **OPTIONAL** but recommended.

**Purpose:**
1. **DOCUMENTATION:** Column descriptions show up in dbt docs.
2. **TESTS:** You can attach tests to columns (see next section).
3. **CONTRACTS:** In advanced setups, you can enforce column types.

You do NOT need to list every column. List only the ones you want to document or test. dbt doesn't care if you skip columns.

### 4L. `tests:` (Column-Level Tests)

```yaml
columns:
  - name: customer_id
    tests:
      - unique
      - not_null
```

Tests validate your **SOURCE DATA** before dbt models use it. If a test fails, it means your raw data has quality issues.

#### Built-in Tests:

**`unique`**
- Checks: Are all values in this column unique?
- Fails if: Any duplicate values exist.
- SQL dbt generates:
  ```sql
  SELECT customer_id, COUNT(*) AS cnt
  FROM DBT_PROJECT_DB.RAW.CUSTOMERS
  GROUP BY customer_id
  HAVING COUNT(*) > 1
  ```
  If this returns ANY rows, the test **FAILS**.

**`not_null`**
- Checks: Are all values in this column non-null?
- Fails if: Any NULL values exist.
- SQL dbt generates:
  ```sql
  SELECT customer_id
  FROM DBT_PROJECT_DB.RAW.CUSTOMERS
  WHERE customer_id IS NULL
  ```
  If this returns ANY rows, the test **FAILS**.

**`accepted_values`**
- Checks: Are all values in the allowed list?
- Fails if: Any value is NOT in the list.
- Example:
  ```yaml
  - name: country
    tests:
      - accepted_values:
          values: ['India', 'United States', 'United Kingdom', 'Canada', 'Australia']
  ```
- SQL dbt generates:
  ```sql
  SELECT country
  FROM DBT_PROJECT_DB.RAW.CUSTOMERS
  WHERE country NOT IN ('India', 'United States', 'United Kingdom', 'Canada', 'Australia')
  ```
  If this returns ANY rows, the test **FAILS**.

**`relationships`**
- Checks: Does every value in this column exist in another table's column? (Foreign key check)
- Example:
  ```yaml
  - name: customer_id
    tests:
      - relationships:
          to: ref('dim_customers')
          field: customer_id
  ```
- Fails if: A `customer_id` in this table doesn't exist in `dim_customers`.

#### How to Run Tests:

```bash
dbt test                                # Run ALL tests (sources + models)
dbt test --select source:raw            # Run only source 'raw' tests
dbt test --select source:raw.customers  # Run only customers source tests
```

---

## 5. HOW `{{ source() }}` WORKS IN MODELS

Once `_sources.yml` is defined, you use `{{ source() }}` in your models:

**FILE:** `models/staging/stg_customers.sql`

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'customers') }}
)

SELECT
    customer_id,
    first_name,
    last_name,
    first_name || ' ' || last_name AS full_name,
    email,
    signup_date,
    country,
    _loaded_at
FROM source
```

### What Happens at Compile Time:

| Step | What |
|------|------|
| **YOUR CODE:** | `SELECT * FROM {{ source('raw', 'customers') }}` |
| **dbt READS _sources.yml:** | source group `raw` → `database: DBT_PROJECT_DB`, `schema: RAW`; table `customers` → `name: customers` |
| **dbt COMPILES TO:** | `SELECT * FROM DBT_PROJECT_DB.RAW.CUSTOMERS` |

You can verify this by running: `dbt compile --select stg_customers`
This shows the compiled SQL without executing it.

---

## 6. PER-TABLE OVERRIDES

You can override source-group-level settings for individual tables.

### Example: Different freshness per table

```yaml
version: 2

sources:
  - name: raw
    database: DBT_PROJECT_DB
    schema: RAW
    freshness:
      warn_after: { count: 24, period: hour }
      error_after: { count: 48, period: hour }
    loaded_at_field: _loaded_at

    tables:
      - name: customers
        description: "Updated weekly - relaxed freshness"
        freshness:
          warn_after: { count: 7, period: day }
          error_after: { count: 14, period: day }

      - name: orders
        description: "Updated hourly - strict freshness"
        freshness:
          warn_after: { count: 2, period: hour }
          error_after: { count: 6, period: hour }

      - name: products
        description: "Static data - no freshness check"
        freshness: null
```

**What happens:**
- `customers` → Uses its OWN freshness (7 days warn, 14 days error)
- `orders` → Uses its OWN freshness (2 hours warn, 6 hours error)
- `products` → `freshness: null` DISABLES freshness checks for this table

**You can also override these per table:**
- `identifier:` — Override the actual table name in Snowflake
- `loaded_at_field:` — Use a different timestamp column per table
- `database:` — Read from a different database per table
- `schema:` — Read from a different schema per table

### Example: Table name in Snowflake differs from source name

```yaml
tables:
  - name: customers
    identifier: RAW_CUSTOMER_DATA_V2
```

In your model: `{{ source('raw', 'customers') }}`
dbt compiles to: `DBT_PROJECT_DB.RAW.RAW_CUSTOMER_DATA_V2`

`identifier` lets you use a clean name in your code while pointing to an ugly or versioned table name in Snowflake.

---

## 7. QUOTING (Handling Special Characters)

If your table or column names have special characters, spaces, or mixed case, you can force dbt to quote them:

**Source group level:**

```yaml
sources:
  - name: raw
    quoting:
      database: false
      schema: false
      identifier: false
```

**Table level:**

```yaml
tables:
  - name: My Table
    quoting:
      identifier: true
```

Compiles to: `DBT_PROJECT_DB.RAW."My Table"` (quoted!)

> **Default:** Snowflake is case-insensitive, so quoting is usually NOT needed. Only use it if table names have spaces, mixed case, or special characters.

---

## 8. TAGS ON SOURCES

You can tag sources for selective test runs:

```yaml
sources:
  - name: raw
    tags: ["daily", "critical"]
    tables:
      - name: customers
        tags: ["pii"]
```

**Run by tag:**

```bash
dbt test --select tag:pii          # Tests only PII sources
dbt test --select tag:critical     # Tests all critical sources
```

---

## 9. THE COMPLETE HIERARCHY OF _sources.yml

Here is every property at every level:

```yaml
version: 2                                    # Always 2

sources:                                      # Root key
  - name: <source_group_name>                 # REQUIRED: group identifier
    description: "<text>"                      # Optional: group description
    database: <database_name>                  # Optional: Snowflake database
    schema: <schema_name>                      # Optional: Snowflake schema
    freshness:                                 # Optional: freshness config
      warn_after: {count: N, period: P}        #   Warning threshold
      error_after: {count: N, period: P}       #   Error threshold
    loaded_at_field: <column_name>             # Optional: timestamp column
    quoting:                                   # Optional: quoting settings
      database: true/false
      schema: true/false
      identifier: true/false
    tags: [<tag1>, <tag2>]                     # Optional: source-level tags

    tables:                                    # REQUIRED: list of tables
      - name: <table_name>                     # REQUIRED: actual table name
        description: "<text>"                  # Optional: table description
        identifier: <actual_snowflake_name>    # Optional: override table name
        freshness:                             # Optional: per-table override
          warn_after: {count: N, period: P}
          error_after: {count: N, period: P}
        freshness: null                        # Optional: disable freshness
        loaded_at_field: <column_name>         # Optional: per-table override
        quoting:                               # Optional: per-table quoting
          identifier: true/false
        tags: [<tag1>, <tag2>]                 # Optional: table-level tags

        columns:                               # Optional: column definitions
          - name: <column_name>                # Column name
            description: "<text>"              # Optional: column description
            tests:                             # Optional: column tests
              - unique                         #   No duplicate values
              - not_null                       #   No NULL values
              - accepted_values:               #   Value must be in list
                  values: [val1, val2, ...]
              - relationships:                 #   Foreign key check
                  to: ref('other_model')
                  field: <column_in_other>
```

---

## 10. QUICK REFERENCE

| Property        | Level    | Required? | Purpose                         |
|-----------------|----------|-----------|---------------------------------|
| version         | Root     | Yes       | Always set to 2                 |
| sources         | Root     | Yes       | Contains all source groups      |
| name            | Group    | Yes       | Source group identifier          |
| description     | Any      | No        | Human-readable description      |
| database        | Group    | No        | Snowflake database name         |
| schema          | Group    | No        | Snowflake schema name           |
| freshness       | Group    | No        | Data freshness thresholds       |
| loaded_at_field | Group    | No*       | Column for freshness check      |
| tables          | Group    | Yes       | List of source tables           |
| name            | Table    | Yes       | Actual Snowflake table name     |
| identifier      | Table    | No        | Override actual table name      |
| columns         | Table    | No        | Column definitions and tests    |
| tests           | Column   | No        | Data quality tests              |
| tags            | Any      | No        | Labels for selective runs       |
| quoting         | Any      | No        | Quote identifiers               |

> \* `loaded_at_field` is required IF `freshness:` is defined.
