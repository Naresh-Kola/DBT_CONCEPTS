# profiles.yml - Complete Guide

> What it is, why you need it, and how to configure it for Snowflake

---

## 1. What is profiles.yml?

`profiles.yml` is a configuration file that tells dbt **HOW to connect** to your data warehouse (in our case, Snowflake).

| File | Purpose |
|------|---------|
| `dbt_project.yml` | **WHAT** to build (models, tests, schemas, etc.) |
| `profiles.yml` | **WHERE** to build it (which database, schema, warehouse) |

> Without `profiles.yml`, dbt has NO idea where to send your SQL. It would be like writing a letter but not knowing the address.

---

## 2. Where Does profiles.yml Live?

There are two possible locations:

**OPTION A: Inside your dbt project folder (RECOMMENDED for Snowflake)**

```
my_dbt_project/
├── dbt_project.yml
├── profiles.yml          <-- RIGHT HERE, same level as dbt_project.yml
├── models/
├── macros/
└── seeds/
```

**OPTION B: In your home directory (default for dbt Core on local machines)**

```
~/.dbt/profiles.yml       <-- Hidden .dbt folder in home directory
```

> For Snowflake Workspaces, **ALWAYS** put it inside the project folder. Snowflake does not read from `~/.dbt/` when running dbt natively.

---

## 3. How profiles.yml Connects to dbt_project.yml

The two files are **LINKED** by the "profile" name.

**Step 1:** In `dbt_project.yml`, you specify a profile name:

```yaml
# FILE: dbt_project.yml
name: 'dbt_practice'
version: '1.0.0'
profile: 'dbt_practice'       # <-- THIS NAME
```

**Step 2:** In `profiles.yml`, you define a profile with **THAT SAME NAME**:

```yaml
# FILE: profiles.yml
dbt_practice:                  # <-- MUST MATCH the profile name above
  target: dev
  outputs:
    dev:
      type: snowflake
      ...
```

**The Connection:**

```
dbt_project.yml                    profiles.yml
┌───────────────────────────┐      ┌───────────────────────────┐
│ profile: 'dbt_practice'  │ ───> │ dbt_practice:             │
└───────────────────────────┘      │   target: dev             │
                                   │   outputs:                │
  dbt reads the profile name       │     dev:                  │
  from dbt_project.yml, then       │       type: snowflake     │
  finds the matching profile       │       database: ...       │
  in profiles.yml                  └───────────────────────────┘
```

> If the names DON'T match, dbt throws: `Could not find profile named 'dbt_practice'`

---

## 4. Complete profiles.yml Example with Explanation

Below is a full `profiles.yml` file for Snowflake, with every field explained.

```yaml
dbt_practice:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: izb88976
      user: ROHITSHARMA
      role: OPENFLOW_ADMIN
      database: DBT_PROJECT_DB
      warehouse: DBT_WH
      schema: RAW
      threads: 4
```

### Line-by-Line Explanation

#### `dbt_practice:`

The **PROFILE NAME**. It must match the `profile:` value in `dbt_project.yml`. You can name it anything, but it must be consistent across both files.

#### `target: dev`

Tells dbt **WHICH environment** to use by default. You can define multiple environments (`dev`, `staging`, `prod`) under `outputs:` and `target` picks the active one.

```bash
dbt run                  # Uses "dev" (because target: dev)
dbt run --target prod    # Uses "prod" environment
```

#### `outputs:`

Parent key that holds **ALL** your environment definitions. Each child key under `outputs:` is a named environment.

#### `dev:`

The **ENVIRONMENT NAME**. It matches the `target: dev` above. Everything indented under `dev:` defines the connection settings for the development environment.

#### `type: snowflake`

Tells dbt which **ADAPTER** to use.

| Type | Database |
|------|----------|
| `snowflake` | Snowflake |
| `postgres` | PostgreSQL |
| `bigquery` | Google BigQuery |
| `redshift` | Amazon Redshift |

#### `account: izb88976`

Your Snowflake **ACCOUNT IDENTIFIER**. Found in your Snowflake URL:

```
https://izb88976.snowflakecomputing.com
         ^^^^^^^^
         This is your account identifier
```

> For dbt Projects on Snowflake (running inside Snowflake), the account and user can be left as empty strings or arbitrary values because dbt runs under the current session context. But it is good practice to include them for clarity.

#### `user: ROHITSHARMA`

The Snowflake **USERNAME** that dbt will use to connect. Must be a valid Snowflake user with appropriate privileges.

> When running dbt inside a Snowflake Workspace, authentication is handled by the session — you do NOT need a password field.

#### `role: OPENFLOW_ADMIN`

The Snowflake **ROLE** that dbt will use. This role determines what dbt can access and create. The role must have:

- `USAGE` on the warehouse (to run queries)
- `USAGE` on the database (to access it)
- `CREATE SCHEMA` on the database (to create new schemas)
- `CREATE TABLE` / `CREATE VIEW` on schemas (to build models)

> Choose a role that has enough privileges for what dbt needs to do, but not more (principle of least privilege).

#### `database: DBT_PROJECT_DB`

The Snowflake **DATABASE** where dbt will create schemas and models. All your dbt models (tables/views) will be created inside this database.

Example: If database is `DBT_PROJECT_DB` and schema is `STAGING`, dbt creates: `DBT_PROJECT_DB.STAGING.STG_ORDERS`

#### `warehouse: DBT_WH`

The Snowflake **VIRTUAL WAREHOUSE** that dbt will use to execute queries. The warehouse provides the compute power (CPU, memory) to run the SQL that dbt generates.

Important:
- It must be running (or set to auto-resume) when dbt runs
- Larger warehouses = faster dbt runs but higher cost
- The role must have `USAGE` privilege on this warehouse

#### `schema: RAW`

The **DEFAULT SCHEMA**. This is the most important field for understanding how dbt creates schemas.

| Scenario | What Happens | Example |
|----------|-------------|---------|
| Model has **NO** `+schema` config | dbt puts the model in this default schema | `DBT_PROJECT_DB.RAW.MY_MODEL` |
| Model **HAS** `+schema` (e.g., `staging`) **without** custom macro | dbt creates `RAW_STAGING` (prefix + custom) | `DBT_PROJECT_DB.RAW_STAGING.STG_ORDERS` |
| Model **HAS** `+schema` (e.g., `staging`) **with** custom macro | dbt creates `STAGING` (just the custom name) | `DBT_PROJECT_DB.STAGING.STG_ORDERS` |

> This is the `default_schema` variable that the `generate_schema_name` macro uses when `custom_schema_name` is None.

#### `threads: 4`

The number of **PARALLEL** models dbt can build at the same time.

| Threads | Behavior |
|---------|----------|
| `1` | Builds models one at a time (slowest, safest) |
| `4` | Builds up to 4 models in parallel (good default) |
| `8` | Builds up to 8 in parallel (faster, more compute) |

> Higher threads = faster dbt runs, but uses more warehouse resources. Start with 4, increase if your warehouse can handle it.

---

## 5. Multiple Environments Example

You can define multiple environments (`dev`, `staging`, `prod`) in one file:

```yaml
dbt_practice:
  target: dev                          # Default environment
  outputs:

    dev:                               # Development environment
      type: snowflake
      account: izb88976
      user: ROHITSHARMA
      role: OPENFLOW_ADMIN
      database: DBT_PROJECT_DB
      warehouse: DEV_WH
      schema: DEV_RAW
      threads: 4

    staging:                           # Staging/QA environment
      type: snowflake
      account: izb88976
      user: ROHITSHARMA
      role: DBT_STAGING_ROLE
      database: DBT_PROJECT_DB
      warehouse: STAGING_WH
      schema: STG_RAW
      threads: 4

    prod:                              # Production environment
      type: snowflake
      account: izb88976
      user: DBT_SERVICE_USER
      role: DBT_PROD_ROLE
      database: PROD_DB
      warehouse: PROD_WH
      schema: RAW
      threads: 8
```

### How to Switch Between Environments

```bash
dbt run                  # Uses "dev" (because target: dev)
dbt run --target staging # Uses "staging" environment
dbt run --target prod    # Uses "prod" environment
```

### Why Multiple Environments?

| Environment | Purpose |
|-------------|---------|
| **DEV** | Your personal sandbox. Experiment freely, break things. Uses a smaller warehouse (cheaper). |
| **STAGING** | Shared testing environment. Test before going to production. Mimics production settings. |
| **PROD** | Production. Real data, real dashboards depend on this. Uses a larger warehouse, runs under a service account. |

---

## 6. Important Rules for profiles.yml on Snowflake

| # | Rule | Details |
|---|------|---------|
| 1 | **NO PASSWORDS** | When running dbt inside Snowflake (Workspace or dbt Project object), do NOT include `password:` or `authenticator:` fields. Authentication is handled by the Snowflake session automatically. If you include a password, deployment will be blocked. |
| 2 | **NO `env_var()`** | Environment variables like `{{ env_var('MY_SECRET') }}` are NOT supported when running dbt inside Snowflake. dbt runs server-side, not on your local machine. Use `--vars` for project variables instead. |
| 3 | **TARGET SCHEMA MUST EXIST** | The default schema (e.g., `RAW`) specified in `profiles.yml` must already exist in Snowflake BEFORE you create a dbt Project object. Custom schemas (`+schema: staging`) are auto-created by dbt, but the default one must exist beforehand. |
| 4 | **PROFILE NAME MUST MATCH** | The top-level key in `profiles.yml` (e.g., `dbt_practice:`) MUST match the `profile:` value in `dbt_project.yml`. Mismatched names = error. |
| 5 | **YAML INDENTATION MATTERS** | YAML is whitespace-sensitive. Use **SPACES** (not tabs) for indentation. Incorrect indentation will cause dbt to fail to parse the file. Standard: 2 spaces per indent level. |

---

## 7. How profiles.yml Fits into the Overall dbt Flow

When you run `dbt run`, here is how `profiles.yml` is used:

1. **dbt reads** `dbt_project.yml` → Finds: `profile: 'dbt_practice'`
2. **dbt reads** `profiles.yml` → Finds the `dbt_practice` profile → Reads `target: dev` → Loads the `dev` output config
3. **dbt connects** to Snowflake using: Account `izb88976`, User `ROHITSHARMA`, Role `OPENFLOW_ADMIN`, Database `DBT_PROJECT_DB`, Warehouse `DBT_WH`
4. **For each model**, dbt determines the schema:
   - If `+schema` is set → calls `generate_schema_name` macro
   - If `+schema` not set → uses default schema: `RAW`
5. **dbt executes** the SQL against Snowflake: `CREATE VIEW DBT_PROJECT_DB.STAGING.STG_ORDERS AS (...)`

### Visual Flow

```
dbt_project.yml          profiles.yml               Snowflake
┌──────────────────┐     ┌────────────────────┐     ┌──────────────────┐
│ profile:         │     │ dbt_practice:      │     │                  │
│  'dbt_practice' ─┼────>│   target: dev      │     │ Account: izb88976│
│                  │     │   outputs:         │     │ DB: DBT_PROJECT_ │
│ models:          │     │     dev:           ├────>│      DB          │
│   staging:       │     │       database:    │     │ WH: DBT_WH      │
│     +schema:     │     │        DBT_PROJ.DB │     │ Role: OPENFLOW_  │
│      staging     │     │       warehouse:   │     │       ADMIN      │
│     +materialized│     │        DBT_WH      │     │                  │
│      view        │     │       schema: RAW  │     │ Creates schemas  │
└──────────────────┘     └────────────────────┘     │ and models here  │
                                                    └──────────────────┘
```

---

## 8. Quick Reference

| Field | Example | Purpose |
|-------|---------|---------|
| `type` | `snowflake` | Which database adapter to use |
| `account` | `izb88976` | Your Snowflake account identifier |
| `user` | `ROHITSHARMA` | Snowflake username |
| `role` | `OPENFLOW_ADMIN` | Snowflake role for permissions |
| `database` | `DBT_PROJECT_DB` | Target database for all models |
| `warehouse` | `DBT_WH` | Compute warehouse for running queries |
| `schema` | `RAW` | Default schema (fallback) |
| `threads` | `4` | Max parallel models to build |
| `target` | `dev` | Which environment to use by default |
