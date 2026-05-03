# dbt_project.yml - Complete Guide

> What it is, every config explained, and full examples

---

## 1. What is dbt_project.yml?

`dbt_project.yml` is the **MAIN CONFIGURATION FILE** of every dbt project. It is the **FIRST file** dbt reads when you run any command.

Think of it like this:

| File | Purpose |
|------|---------|
| `profiles.yml` | **WHERE** to build (database, warehouse, connection) |
| `dbt_project.yml` | **WHAT** to build and **HOW** to build it |

It defines:
- Project name and version
- Which profile to use (links to `profiles.yml`)
- Where your models, seeds, macros, tests, snapshots live
- How each model should be materialized (table, view, incremental)
- Which schema each folder maps to
- Global variables, configurations, and settings

> **Without this file, dbt does not recognize your folder as a dbt project.**
> If dbt cannot find `dbt_project.yml`, it throws: `ERROR: no dbt_project.yml found`

---

## 2. Where Does dbt_project.yml Live?

It **MUST** be in the **ROOT** of your dbt project folder:

```
my_dbt_project/
├── dbt_project.yml       <-- MUST be here (root level)
├── profiles.yml
├── models/
├── macros/
├── seeds/
├── snapshots/
├── tests/
└── analysis/
```

dbt looks for this file in the current working directory when you run any dbt command. If it is in a subfolder, dbt won't find it.

---

## 3. Full Example with Explanation

Below is a complete `dbt_project.yml` with every section explained.

```yaml
name: 'dbt_practice'
version: '1.0.0'

profile: 'dbt_practice'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

models:
  dbt_practice:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
    metrics:
      +materialized: table
      +schema: metrics
```

---

## 4. Line-by-Line Explanation

### 4A. Project Identity

#### `name: 'dbt_practice'`

The **PROJECT NAME**. This is a unique identifier for your dbt project.

**Rules:**
- Must contain only letters, digits, and underscores
- Cannot start with a digit
- Must be lowercase (convention)
- Must be unique if you install this project as a package in another project

**Where it is used:**
1. In `dbt_project.yml` under `models:` to reference your project's models (e.g., `models: dbt_practice: staging: ...`)
2. In `ref()` calls if you use cross-project references
3. In package management if you publish this project

**Examples:**
| Value | Valid? |
|-------|--------|
| `'dbt_practice'` | ✅ Good |
| `'my_ecommerce_project'` | ✅ Good |
| `'My Project'` | ❌ Bad (spaces and uppercase) |
| `'123project'` | ❌ Bad (starts with digit) |

#### `version: '1.0.0'`

The **VERSION** of your project. This is for YOUR tracking purposes. dbt does not enforce versioning rules — it is informational.

Convention: Use semantic versioning (`MAJOR.MINOR.PATCH`)
- `1.0.0` → Initial version
- `1.1.0` → Added new models
- `2.0.0` → Breaking changes

---

### 4B. Profile Link

#### `profile: 'dbt_practice'`

This tells dbt **WHICH PROFILE** to use from `profiles.yml`.

```
dbt_project.yml                    profiles.yml
┌───────────────────────────┐      ┌───────────────────────────┐
│ profile: 'dbt_practice'  │ ───> │ dbt_practice:             │
└───────────────────────────┘      │   target: dev             │
                                   │   outputs:                │
                                   │     dev:                  │
                                   │       type: snowflake     │
                                   │       database: ...       │
                                   └───────────────────────────┘
```

The value **MUST** match a top-level key in `profiles.yml`. If it does not match, dbt throws: `Could not find profile named 'dbt_practice'`

---

### 4C. File Paths (Where dbt Looks for Your Files)

| Config | Purpose | Default |
|--------|---------|---------|
| `model-paths: ["models"]` | Look in `models/` for `.sql` model files. Every `.sql` file (and subfolders) is treated as a model. | `["models"]` |
| `analysis-paths: ["analyses"]` | Look in `analyses/` for analysis `.sql` files. Compiled but **NOT executed** during `dbt run`. | `["analyses"]` |
| `test-paths: ["tests"]` | Look in `tests/` for custom test `.sql` files (singular tests that must return 0 rows to pass). | `["tests"]` |
| `seed-paths: ["seeds"]` | Look in `seeds/` for `.csv` files. dbt loads these into your database as tables. | `["seeds"]` |
| `macro-paths: ["macros"]` | Look in `macros/` for `.sql` macro files. This is where `generate_schema_name.sql` lives! | `["macros"]` |
| `snapshot-paths: ["snapshots"]` | Look in `snapshots/` for snapshot `.sql` files (SCD/history tracking). | `["snapshots"]` |

**You can customize paths:**
```yaml
model-paths: ["transformations"]
# Or multiple paths:
model-paths: ["models", "more_models"]
```

**Example test file:** `tests/assert_positive_revenue.sql`
```sql
SELECT * FROM {{ ref('fct_sales') }} WHERE revenue < 0
-- If this returns any rows, the test FAILS.
```

**Example seed file:** `seeds/country_codes.csv`
```csv
code,name
US,United States
IN,India
```
Run `dbt seed` → Creates a table called `COUNTRY_CODES`

---

### 4D. Clean Targets

```yaml
clean-targets:
  - "target"
  - "dbt_packages"
```

When you run `dbt clean`, dbt **DELETES** these folders:

| Folder | Purpose | Safe to delete? |
|--------|---------|-----------------|
| `target/` | Contains compiled SQL and run artifacts. dbt writes here every time you run/compile. | ✅ Yes — dbt regenerates it |
| `dbt_packages/` | Contains installed dbt packages (like npm's `node_modules`). | ✅ Yes — reinstall with `dbt deps` |

> `dbt clean` is like a **reset button** for your project's generated files.

---

### 4E. Models Configuration (The Most Important Section)

```yaml
models:
  dbt_practice:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
    metrics:
      +materialized: table
      +schema: metrics
```

#### Breaking It Down Level by Level:

**LEVEL 1: `models:`**
Top-level key. Everything under here configures how dbt builds your models.

**LEVEL 2: `dbt_practice:`**
This **MUST** match the `name:` at the top of `dbt_project.yml`. It tells dbt: "The following configs apply to THIS project's models."

Why is the project name needed here? Because you can install **external dbt packages** (other projects), and each package has its own models. This key ensures you're configuring YOUR project's models, not a package's.

```yaml
models:
  dbt_practice:          # YOUR project's models
    staging:
      +materialized: view
  dbt_utils:             # An installed package's models
    +materialized: table
```

**LEVEL 3: `staging:` / `intermediate:` / `marts:` / `metrics:`**
These are **FOLDER NAMES** inside `models/`. dbt matches these keys to actual folder names in your file system.

```
models/
├── staging/           <-- matches "staging:" key
├── intermediate/      <-- matches "intermediate:" key
├── marts/             <-- matches "marts:" key
└── metrics/           <-- matches "metrics:" key
```

> If the folder name doesn't match the key, the config is **IGNORED**. dbt is case-sensitive here.

**LEVEL 4: `+materialized:` and `+schema:`**

The **"+" prefix** is important. It means "apply this config to ALL models in this folder and ALL subfolders."

| Prefix | Behavior |
|--------|----------|
| Without `+` | Config applies ONLY at this specific level |
| With `+` | Config **CASCADES** down to all children |

```yaml
staging:
  +materialized: view       # All models in staging/ are views
  raw_data:                 # Subfolder: models/staging/raw_data/
                            # Also inherits +materialized: view
```

---

## 5. Understanding +materialized (How Models Are Built)

`+materialized` tells dbt **WHAT TYPE** of object to create in Snowflake.

### Type 1: `view`

```yaml
+materialized: view
```

dbt creates a Snowflake **VIEW**. A view stores NO data — it is just a saved SQL query. Every time someone queries the view, the SQL runs fresh.

```sql
-- SQL dbt generates:
CREATE OR REPLACE VIEW schema.model_name AS (
  SELECT ... your model SQL ...
);
```

**Best for:** staging, intermediate layers
**Pros:** No storage cost, always shows latest data, fast to build
**Cons:** Slower to query (runs SQL every time), can be slow for complex logic

### Type 2: `table`

```yaml
+materialized: table
```

dbt creates a Snowflake **TABLE** with data physically stored. The data is a snapshot of the query result at build time.

```sql
-- SQL dbt generates:
CREATE OR REPLACE TABLE schema.model_name AS (
  SELECT ... your model SQL ...
);
```

**Best for:** marts, metrics, final outputs
**Pros:** Fast to query (data pre-computed), BI tools love tables
**Cons:** Uses storage, data is only as fresh as last dbt run

### Type 3: `incremental`

```yaml
+materialized: incremental
```

dbt **APPENDS** only NEW or CHANGED rows instead of rebuilding the entire table. Critical for large tables (millions of rows).

```sql
-- First run: Creates the table
CREATE TABLE schema.model_name AS (SELECT ...);

-- Subsequent runs: Inserts only new rows
INSERT INTO schema.model_name
  SELECT ... WHERE updated_at > (SELECT MAX(updated_at) FROM schema.model_name);
```

**Best for:** Large fact tables, event logs, transaction history
**Pros:** Much faster than full table rebuild, cost-efficient
**Cons:** More complex to write (need `is_incremental()` logic in model)

### Type 4: `ephemeral`

```yaml
+materialized: ephemeral
```

dbt creates **NOTHING** in Snowflake. The model exists only as a CTE (Common Table Expression) that gets injected into downstream models.

**Best for:** Reusable logic that doesn't need its own table/view
**Pros:** No object in database, clean warehouse
**Cons:** Cannot query directly, can make downstream SQL very long

### Summary Table

| Type | Creates Object? | Stores Data? | Best For |
|------|----------------|--------------|----------|
| `view` | Yes (VIEW) | No | staging, intermediate |
| `table` | Yes (TABLE) | Yes | marts, metrics |
| `incremental` | Yes (TABLE) | Yes (append) | large fact tables |
| `ephemeral` | No | No | reusable CTEs |

---

## 6. Understanding +schema (How Schemas Are Assigned)

`+schema` tells dbt **WHICH SCHEMA** to put the model in.

### How It Works:

1. dbt reads `+schema` from `dbt_project.yml` (e.g., `+schema: staging`)
2. dbt calls the `generate_schema_name` macro
3. The macro returns the final schema name

| Scenario | Formula | Example |
|----------|---------|---------|
| **Without** custom macro (dbt default) | `<profiles.yml schema>` + `_` + `<custom schema>` | `RAW` + `_` + `staging` = `RAW_STAGING` |
| **With** custom macro (your override) | `<custom schema>` | `staging` = `STAGING` (clean!) |

> See `dbt_custom_schemas_guide.sql` for the full explanation.

---

## 7. Additional Configs You Can Add

Beyond `+materialized` and `+schema`, there are many other configs:

### `+tags`

Tags let you **GROUP** models and run them selectively.

```yaml
models:
  dbt_practice:
    staging:
      +tags: ["daily", "staging"]
    marts:
      +tags: ["daily", "marts", "finance"]
```

```bash
dbt run --select tag:daily     # Runs only models tagged "daily"
dbt run --select tag:finance   # Runs only models tagged "finance"
```

### `+enabled`

Enable or disable models. Disabled models are skipped during `dbt run`.

```yaml
models:
  dbt_practice:
    old_models:
      +enabled: false         # All models in old_models/ are skipped
```

### `+persist_docs`

Tells dbt to push column and model descriptions to Snowflake. Without this, descriptions exist only in dbt docs, not in Snowflake.

```yaml
models:
  dbt_practice:
    +persist_docs:
      relation: true          # Pushes model description to Snowflake
      columns: true           # Pushes column descriptions to Snowflake
```

### `+pre-hook` and `+post-hook`

SQL statements to run **BEFORE** or **AFTER** a model is built.

```yaml
models:
  dbt_practice:
    marts:
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO ROLE ANALYST_ROLE"
```

This automatically grants SELECT on every mart table after it is built.

### `+database`

Override the target database for specific models.

```yaml
models:
  dbt_practice:
    exports:
      +database: SHARED_DB    # These models go to SHARED_DB instead
```

---

## 8. Seed and Snapshot Configurations

### Seeds

```yaml
seeds:
  dbt_practice:
    +schema: seeds
    country_codes:                 # Specific seed file name (without .csv)
      +column_types:
        code: varchar(10)
        name: varchar(100)
```

This creates the `country_codes` table in the `SEEDS` schema with specific column types instead of dbt's default (`varchar(256)`).

### Snapshots

```yaml
snapshots:
  dbt_practice:
    +schema: snapshots
```

All snapshots will be created in the `SNAPSHOTS` schema.

---

## 9. Vars (Project Variables)

```yaml
vars:
  start_date: '2023-01-01'
  default_country: 'US'
  enable_debug: false
```

Variables are values you can reference in your models using `{{ var('start_date') }}`.

**Example model SQL:**
```sql
SELECT * FROM orders
WHERE order_date >= '{{ var("start_date") }}'
```

**Override at runtime:**
```bash
dbt run --vars '{"start_date": "2024-01-01"}'
```

This is useful for parameterizing your models without hardcoding values.

---

## 10. Full dbt_project.yml with All Configs

Here is an advanced example with everything put together:

```yaml
name: 'dbt_practice'
version: '1.0.0'
config-version: 2

profile: 'dbt_practice'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

vars:
  start_date: '2023-01-01'
  default_country: 'US'

models:
  dbt_practice:
    +persist_docs:
      relation: true
      columns: true
    staging:
      +materialized: view
      +schema: staging
      +tags: ["daily", "staging"]
    intermediate:
      +materialized: view
      +schema: intermediate
      +tags: ["daily"]
    marts:
      +materialized: table
      +schema: marts
      +tags: ["daily", "business"]
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO ROLE ANALYST_ROLE"
    metrics:
      +materialized: table
      +schema: metrics
      +tags: ["weekly", "reporting"]

seeds:
  dbt_practice:
    +schema: seeds

snapshots:
  dbt_practice:
    +schema: snapshots
```

---

## 11. How dbt_project.yml Fits into the Overall Flow

When you run `dbt run`, here is how `dbt_project.yml` is used:

1. **dbt finds and reads** `dbt_project.yml` (must be in root)
2. **dbt reads** the project name and version: `name: 'dbt_practice'`, `version: '1.0.0'`
3. **dbt reads** the profile name: `profile: 'dbt_practice'` → Goes to `profiles.yml` to get connection
4. **dbt reads** file paths: `model-paths: ["models"]` → Scans `models/` for `.sql` files; `macro-paths: ["macros"]` → Loads all macros
5. **For EACH `.sql` file** found in `models/`, dbt:
   - Checks which folder the file is in (e.g., `models/staging/`)
   - Looks up the config for that folder (`staging: +materialized: view, +schema: staging`)
   - Calls `generate_schema_name` macro to get the final schema
   - Builds the model (`CREATE VIEW/TABLE`) in the resolved schema

### Visual Flow

```
dbt_project.yml
┌────────────────────────────────┐
│ name: dbt_practice             │
│ profile: dbt_practice  ───────────> profiles.yml → Snowflake connection
│ model-paths: ["models"] ──┐   │    (database, warehouse, role, schema)
│                            │   │
│ models:                    │   │
│   dbt_practice:            │   │
│     staging:               ├───┼──> models/staging/*.sql
│       +materialized: view  │   │      → CREATE VIEW in STAGING schema
│       +schema: staging     │   │
│     marts:                 ├───┼──> models/marts/*.sql
│       +materialized: table │   │      → CREATE TABLE in MARTS schema
│       +schema: marts       │   │
└────────────────────────────────┘
```

---

## 12. Common Mistakes and How to Fix Them

| # | Mistake | Problem | Fix |
|---|---------|---------|-----|
| 1 | Project name mismatch | `name: 'dbt_practice'` but `models: my_project:` | Key under `models:` must match `name:` exactly |
| 2 | Profile name mismatch | `profile: 'dbt_practice'` but `profiles.yml` has `my_profile:` | Both must use the same name |
| 3 | Folder name mismatch | YAML says `staging:` but folder is `models/stg/` | Folder name and YAML key must match exactly |
| 4 | Missing `+` prefix | `materialized: view` instead of `+materialized: view` | Add `+` to cascade config to subfolders |
| 5 | Tabs in YAML | YAML does NOT allow tabs | Use **spaces only** (2 spaces per indent) |

---

## 13. Quick Reference Table

| Config Key | Example Value | Purpose |
|------------|--------------|---------|
| `name` | `'dbt_practice'` | Project identifier |
| `version` | `'1.0.0'` | Project version (informational) |
| `profile` | `'dbt_practice'` | Links to `profiles.yml` |
| `model-paths` | `["models"]` | Where `.sql` models live |
| `seed-paths` | `["seeds"]` | Where `.csv` seed files live |
| `macro-paths` | `["macros"]` | Where Jinja macros live |
| `test-paths` | `["tests"]` | Where custom test SQL files live |
| `snapshot-paths` | `["snapshots"]` | Where snapshot files live |
| `analysis-paths` | `["analyses"]` | Where analysis SQL files live |
| `clean-targets` | `["target","dbt_packages"]` | Folders deleted by `dbt clean` |
| `+materialized` | `view/table/incremental/ephemeral` | How to build the model |
| `+schema` | `staging/marts/etc.` | Which schema to use |
| `+tags` | `["daily","finance"]` | Labels for selective runs |
| `+enabled` | `true/false` | Enable or skip models |
| `+persist_docs` | `relation/columns` | Push docs to Snowflake |
| `+pre-hook` | SQL string | Run SQL before model builds |
| `+post-hook` | SQL string | Run SQL after model builds |
| `+database` | `SHARED_DB` | Override target database |
| `vars` | `key: value` | Project-wide variables |
