# DBT SEEDS: COMPLETE GUIDE

What are seeds, why we use them, when to use them, when NOT to use them, and how to create and reference them in models.

---

## 1. WHAT ARE DBT SEEDS?

Seeds = **SMALL CSV FILES** that dbt loads directly into your data warehouse as tables. They live in your dbt project's `/seeds` directory.

When you run `dbt seed`, dbt:
1. Reads the CSV file from `/seeds` folder
2. Creates a table in your warehouse (or replaces it)
3. Inserts all rows from the CSV

> Think of it as: **"Version-controlled lookup data that lives WITH your code"**

**Example:**
- `/seeds/country_codes.csv` → becomes table: `DBT_DEV_DB.DEV_RAW.COUNTRY_CODES`
- `/seeds/status_mapping.csv` → becomes table: `DBT_DEV_DB.DEV_RAW.STATUS_MAPPING`

---

## 2. WHY DO WE USE SEEDS?

**REASON 1: LOOKUP / REFERENCE DATA**
Small tables that rarely change and are needed for JOINs or mappings. Examples: country codes, currency rates, department names, status codes. Instead of hardcoding these in SQL, put them in a CSV and reference them.

**REASON 2: VERSION CONTROL**
Seeds live in your Git repo alongside your models. Any change to the data is tracked in Git (who changed what, when, why). You can review changes via Pull Requests before they go to production.

**REASON 3: ENVIRONMENT CONSISTENCY**
Same seed data in dev, staging, and production. No manual INSERT statements that might differ across environments. `dbt seed` creates identical tables everywhere.

**REASON 4: TEST DATA**
Small datasets for testing your models during development. Mock data that exercises edge cases.

**REASON 5: BUSINESS LOGIC MAPPING**
When business rules live in a spreadsheet (e.g., "region X maps to zone Y"), export it as CSV, commit to seeds, and use in your models. Business users can update the CSV, create a PR, and it's automatically deployed.

**REASON 6: NO ETL PIPELINE NEEDED**
For data that doesn't come from a source system, seeds avoid building a full extraction pipeline for a 50-row lookup table.

---

## 3. WHEN TO USE SEEDS (Good Use Cases)

- ✓ Lookup tables (< 1000 rows): country codes, zip codes, state abbreviations
- ✓ Mapping tables: status code → description, old_code → new_code
- ✓ Business rule tables: tax rates by region, commission tiers
- ✓ Static configuration: warehouse mappings, team assignments
- ✓ Test fixtures: mock data for unit testing models
- ✓ One-time historical corrections: manual adjustments to fix bad data
- ✓ Enum/Reference data: payment methods, order statuses, category hierarchies

---

## 4. WHEN NOT TO USE SEEDS (Bad Use Cases)

- ✗ Large datasets (> 1000 rows): Seeds are NOT for big data. Use COPY INTO.
- ✗ Frequently changing data: If it changes daily/weekly, use a source table.
- ✗ Data from source systems: Use sources + staging models, not seeds.
- ✗ Sensitive/PII data: CSVs in Git = visible to everyone with repo access.
- ✗ Binary or complex data: Seeds only support CSV (text-based).
- ✗ Data that needs incremental loading: Seeds do full replace every time.
- ✗ Anything > 1 MB CSV file: Performance degrades, use external stages.

> **RULE OF THUMB:**
> "If the data fits in a spreadsheet tab AND rarely changes → **SEED**"
> "If it comes from a system or changes often → **SOURCE**"

---

## 5. HOW SEEDS WORK (Step by Step)

**STEP 1:** Create a CSV file in `/seeds` directory

File: `/seeds/status_mapping.csv`
```csv
status_code,status_name,status_category,is_active
PND,Pending,Open,true
SHP,Shipped,In Transit,true
DLV,Delivered,Closed,true
CAN,Cancelled,Closed,false
RET,Returned,Closed,false
PRO,Processing,Open,true
```

**STEP 2:** (Optional) Configure in `dbt_project.yml`
```yaml
seeds:
  my_project:
    +schema: reference
    status_mapping:
      +column_types:
        status_code: varchar(10)
        is_active: boolean
```

**STEP 3:** Run `dbt seed`
This creates the table in your warehouse.

**STEP 4:** Reference in models using `{{ ref('status_mapping') }}`

---

## 6. PRACTICAL EXAMPLE: CREATING SEEDS AND A MODEL

### 6.1 SEED FILE: `/seeds/status_mapping.csv`
```csv
status_code,status_name,status_category,is_active
PND,Pending,Open,true
SHP,Shipped,In Transit,true
DLV,Delivered,Closed,true
CAN,Cancelled,Closed,false
RET,Returned,Closed,false
PRO,Processing,Open,true
```

### 6.2 SEED FILE: `/seeds/region_mapping.csv`
```csv
state_code,state_name,region,timezone
MH,Maharashtra,West,IST
KA,Karnataka,South,IST
DL,Delhi,North,IST
TN,Tamil Nadu,South,IST
UP,Uttar Pradesh,North,IST
GJ,Gujarat,West,IST
WB,West Bengal,East,IST
RJ,Rajasthan,North,IST
AP,Andhra Pradesh,South,IST
OR,Odisha,East,IST
```

### 6.3 SEED FILE: `/seeds/payment_method_mapping.csv`
```csv
payment_code,payment_name,payment_category,processing_fee_pct
CC,Credit Card,Card,2.5
DC,Debit Card,Card,1.5
BT,Bank Transfer,Bank,0.5
UPI,UPI,Digital,0.0
COD,Cash on Delivery,Cash,0.0
NB,Net Banking,Bank,1.0
WL,Wallet,Digital,1.0
```

### 6.4 CONFIGURATION IN `dbt_project.yml`
```yaml
seeds:
  my_project:
    +schema: seeds
    status_mapping:
      +column_types:
        status_code: varchar(10)
        status_name: varchar(50)
        status_category: varchar(20)
        is_active: boolean
    region_mapping:
      +column_types:
        state_code: varchar(5)
        state_name: varchar(50)
        region: varchar(20)
        timezone: varchar(10)
    payment_method_mapping:
      +column_types:
        payment_code: varchar(10)
        payment_name: varchar(50)
        payment_category: varchar(20)
        processing_fee_pct: float
```

### 6.5 MODEL THAT USES SEEDS: `/models/marts/orders_enriched.sql`
```sql
{{ config(
    materialized='table',
    schema='marts'
) }}

SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status AS status_code,
    sm.status_name,
    sm.status_category,
    sm.is_active AS is_open_order,
    p.payment_method AS payment_code,
    pm.payment_name,
    pm.payment_category,
    pm.processing_fee_pct,
    o.amount,
    ROUND(o.amount * pm.processing_fee_pct / 100, 2) AS processing_fee,
    o.amount - ROUND(o.amount * pm.processing_fee_pct / 100, 2) AS net_amount
FROM {{ source('raw', 'orders') }} o
LEFT JOIN {{ ref('status_mapping') }} sm
    ON o.status = sm.status_code
LEFT JOIN {{ source('raw', 'payments') }} p
    ON o.order_id = p.order_id
LEFT JOIN {{ ref('payment_method_mapping') }} pm
    ON p.payment_method = pm.payment_code
```

### 6.6 WHAT HAPPENS WHEN YOU RUN `dbt seed`

```
$ dbt seed

Running with dbt=1.8.0
Found 3 seeds

Concurrency: 4 threads

1 of 3 START seed file seeds.status_mapping .......................... [RUN]
1 of 3 OK loaded seed file seeds.status_mapping ...................... [INSERT 6 in 0.45s]
2 of 3 START seed file seeds.region_mapping .......................... [RUN]
2 of 3 OK loaded seed file seeds.region_mapping ...................... [INSERT 10 in 0.38s]
3 of 3 START seed file seeds.payment_method_mapping .................. [RUN]
3 of 3 OK loaded seed file seeds.payment_method_mapping .............. [INSERT 7 in 0.41s]

Finished running 3 seeds in 2.15s
Completed successfully.
```

**What dbt does behind the scenes:**
1. `CREATE TABLE IF NOT EXISTS status_mapping (status_code VARCHAR, ...);`
2. `TRUNCATE TABLE status_mapping;` (or DROP + CREATE on full-refresh)
3. `INSERT INTO status_mapping VALUES ('PND', 'Pending', 'Open', TRUE), ...;`

### 6.7 RUNNING SPECIFIC SEEDS

```bash
# Run ALL seeds:
dbt seed

# Run specific seed:
dbt seed --select status_mapping

# Run multiple specific seeds:
dbt seed --select status_mapping region_mapping

# Full refresh (drop and recreate):
dbt seed --full-refresh

# Run specific seed with full refresh:
dbt seed --select status_mapping --full-refresh
```

---

## 7. SEED CONFIGURATION OPTIONS

In `dbt_project.yml`:
```yaml
seeds:
  my_project:
    +schema: reference_data     # Target schema (default: your profile schema)
    +quote_columns: true        # Quote column names (for special chars)
    +tags: ['seed', 'reference']
    +enabled: true              # Enable/disable specific seeds

    status_mapping:
      +column_types:            # Override auto-detected types
        status_code: varchar(10)
        is_active: boolean

    large_seed:
      +enabled: false           # Disable a seed without deleting file
```

### COLUMN TYPE OVERRIDES (Important!)

Without `column_types`, dbt **GUESSES** types from CSV content:
- `'123'` → might become INTEGER (but you wanted VARCHAR for zip codes!)
- `'true'` → might become VARCHAR (but you wanted BOOLEAN!)
- `'2024-01-01'` → might become VARCHAR (but you wanted DATE!)

**ALWAYS** specify `column_types` for columns where auto-detection might be wrong (booleans, dates, zip codes, codes with leading zeros).

---

## 8. TESTING SEEDS

Seeds can have tests just like models! Define in a `schema.yml` file:

**`/seeds/schema.yml`:**
```yaml
version: 2

seeds:
  - name: status_mapping
    description: "Maps status codes to human-readable names"
    columns:
      - name: status_code
        description: "Short code for order status"
        tests:
          - unique
          - not_null
      - name: status_name
        tests:
          - not_null
      - name: status_category
        tests:
          - accepted_values:
              values: ['Open', 'In Transit', 'Closed']

  - name: region_mapping
    columns:
      - name: state_code
        tests:
          - unique
          - not_null
```

Run tests: `dbt test --select status_mapping`

---

## 9. SEEDS vs SOURCES vs MODELS

| ASPECT | SEEDS | SOURCES | MODELS |
|--------|-------|---------|--------|
| Data origin | CSV in Git repo | External system | SQL transform |
| Size | Small (< 1000) | Any size | Any size |
| Changes | Rarely | Frequently | On every run |
| Loaded by | `dbt seed` | ETL/ELT pipeline | `dbt run` |
| Referenced | `{{ ref('x') }}` | `{{ source() }}` | `{{ ref('x') }}` |
| Version ctl | ✓ (in Git) | ✗ (external) | ✓ (in Git) |
| Use for | Lookup/mapping | Raw data | Transformations |

---

## 10. REAL-WORLD EXAMPLES OF SEEDS IN PRODUCTION

**EXAMPLE 1: Tax rates by state** (changes 1-2 times per year)
```csv
# /seeds/tax_rates.csv
state_code,tax_rate,effective_from
MH,18.0,2024-01-01
KA,12.0,2024-01-01
DL,5.0,2024-01-01
```

**EXAMPLE 2: Employee ID → Department mapping** (changes monthly)
```csv
# /seeds/dept_mapping.csv
emp_id,department,cost_center
E001,Engineering,CC100
E002,Marketing,CC200
```

**EXAMPLE 3: Data quality thresholds per table** (for DQ framework)
```csv
# /seeds/dq_thresholds.csv
table_name,check_type,threshold_pct,severity
orders,row_count_delta,5.0,HIGH
customers,null_rate,10.0,MEDIUM
```

**EXAMPLE 4: Feature flags / configuration**
```csv
# /seeds/feature_flags.csv
feature_name,is_enabled,environment
new_dashboard,true,prod
beta_report,true,dev
beta_report,false,prod
```

**EXAMPLE 5: Historical exchange rates** (doesn't come from source system)
```csv
# /seeds/exchange_rates.csv
currency,rate_to_usd,effective_date
INR,0.012,2024-01-01
EUR,1.08,2024-01-01
GBP,1.27,2024-01-01
```

---

## 11. BEST PRACTICES

### DO:
- ✓ Keep seeds SMALL (< 1000 rows, < 1 MB)
- ✓ Always define `column_types` for ambiguous columns
- ✓ Add tests (`unique`, `not_null`) on key columns
- ✓ Add descriptions in `schema.yml`
- ✓ Use meaningful file names (`status_mapping.csv` not `data.csv`)
- ✓ Put seeds in subdirectories if you have many (`/seeds/lookups/`, `/seeds/config/`)
- ✓ Review seed changes in PRs (they're data changes!)
- ✓ Run `dbt seed` before `dbt run` in your pipeline

### DON'T:
- ✗ Don't put large datasets in seeds (use sources/stages)
- ✗ Don't put sensitive data in seeds (visible in Git)
- ✗ Don't use seeds as a replacement for proper ETL
- ✗ Don't forget to commit the CSV to Git (it won't exist in other envs!)
- ✗ Don't use seeds for data that changes daily (use incremental models)
- ✗ Don't include BOM (byte order mark) in CSV files (causes header issues)

---

## 12. INTERVIEW QUESTIONS ON SEEDS

---

### LEVEL 1: BASICS

**Q1: What are dbt seeds?**
> Small CSV files stored in the `/seeds` directory that dbt loads into the warehouse as tables using the `dbt seed` command.

**Q2: Where do seed files live in a dbt project?**
> In the `/seeds` directory at the root of the dbt project.

**Q3: What command loads seeds into the warehouse?**
> `dbt seed` (loads all seeds) or `dbt seed --select seed_name` (specific seed).

**Q4: How do you reference a seed in a model?**
> Using `{{ ref('seed_name') }}` — same syntax as referencing models.

**Q5: What file format do seeds support?**
> Only CSV files (`.csv`).

**Q6: Can you test seeds?**
> Yes. Define tests in a `schema.yml` file under the seeds section, same as you would for models (unique, not_null, accepted_values, etc.).

**Q7: What happens when you run `dbt seed` if the table already exists?**
> By default, dbt truncates and reloads the data (full replace). With `--full-refresh`, it drops and recreates the table entirely.

---

### LEVEL 2: INTERMEDIATE

**Q8: When should you use a seed vs a source?**
> Seed: small, static, version-controlled data (< 1000 rows, rarely changes). Source: data from external systems, any size, changes frequently. Rule: "If it comes from a system → source. If it's manually managed → seed."

**Q9: How do you override column types in seeds?**
> In `dbt_project.yml` under seeds → project_name → seed_name → `+column_types`. Example: `+column_types: { zip_code: varchar(10), is_active: boolean }`

**Q10: Why is column_types important?**
> Without it, dbt auto-detects types from CSV content. `'07001'` might become integer 7001 (losing leading zero). `'true'`/`'false'` might stay as VARCHAR. Column types ensure correct data typing.

**Q11: How do you put seeds in a different schema?**
> In `dbt_project.yml`: `seeds: my_project: +schema: reference_data` Or per-seed: `seeds: my_project: my_seed: +schema: lookups`

**Q12: Can seeds have documentation?**
> Yes. Add descriptions in a `schema.yml` file in the seeds directory, same structure as model documentation. Shows up in dbt docs.

**Q13: How do you handle seed changes in a team?**
> Treat CSV changes like code changes — create a branch, make edits, open a PR, review the diff, merge, then run `dbt seed` in CI/CD.

**Q14: What's the execution order: seeds or models first?**
> Seeds must run BEFORE models that reference them. Pipeline: `dbt seed` → `dbt run` → `dbt test`. Or use `dbt build` (runs seeds, models, and tests in dependency order).

---

### LEVEL 3: ADVANCED

**Q15: What's the maximum recommended size for a seed?**
> < 1000 rows, < 1 MB file size. Beyond this, performance degrades because dbt uses INSERT statements (not COPY INTO or bulk loading). For larger datasets, use external stages + sources.

**Q16: How does `dbt build` handle seeds in the DAG?**
> `dbt build` includes seeds in the dependency graph. If model X depends on `ref('my_seed')`, dbt will load the seed BEFORE running model X. This ensures correct execution order.

**Q17: Can you use Jinja in seed CSV files?**
> No. Seeds are plain CSV files. No Jinja, no SQL, no templating. If you need dynamic data generation, use a model with `{{ dbt_utils.generate_series() }}` or a Python model instead.

**Q18: How do you disable a seed without deleting the file?**
> In `dbt_project.yml`: `seeds: my_project: seed_name: +enabled: false`. The file stays in the repo but dbt skips it during `dbt seed`.

**Q19: What happens if a seed CSV has a column header that's a reserved word?**
> Set `+quote_columns: true` in `dbt_project.yml`. This wraps column names in quotes during CREATE TABLE, avoiding reserved word conflicts.

**Q20: How do seeds work in CI/CD pipelines?**
> In CI/CD (e.g., GitHub Actions, dbt Cloud):
> 1. On PR: `dbt seed` runs in a PR-specific schema for testing
> 2. On merge to main: `dbt seed` runs in production schema
> 3. If CSV changed → table gets recreated with new data
> 4. If CSV unchanged → `dbt seed` still runs (idempotent, no harm)

**Q21: Can seeds reference other seeds or models?**
> No. Seeds are just CSV files loaded into tables. They cannot contain SQL or references. Only MODELS can reference seeds (via `{{ ref() }}`).

**Q22: How do you handle seeds that need to be different per environment?**
> - **Option 1:** Use a single seed and add an 'environment' column. Filter in models: `WHERE environment = '{{ target.name }}'`
> - **Option 2:** Use separate seed files (`config_dev.csv`, `config_prod.csv`) and enable/disable per target in `dbt_project.yml`.
> - **Option 3:** Use `vars()` and conditional logic in models, not seeds.

**Q23: In your migration project, how did you use seeds?**
> "We used seeds for 3 things:
> 1. Type mapping table (source_type → target_type for all 450 tables)
> 2. DQ threshold configuration (table → check type → threshold)
> 3. Migration wave assignment (table → wave_number → priority)
>
> All were < 500 rows, rarely changed, and needed to be consistent across dev/staging/prod environments."

---

### LEVEL 4: TRICKY / GOTCHAS

**Q24: A developer added a 50,000-row CSV as a seed. What do you tell them?**
> "Seeds are not for large datasets. 50K rows means:
> - Slow `dbt seed` (uses INSERT statements, not bulk load)
> - Git repo bloated (50K-row CSV is tracked in every commit)
> - CI/CD slower (seed runs on every deploy)
>
> Instead: Load via COPY INTO from a stage, define as a source, and reference with `{{ source() }}`."

**Q25: Your seed has a column 'date' and dbt seed fails. Why?**
> `date` is a reserved keyword in Snowflake. Fix:
> 1. Rename the column in CSV (`order_date` instead of `date`)
> 2. Or set `+quote_columns: true` in `dbt_project.yml`

**Q26: You updated a seed CSV but `dbt seed` shows 0 rows changed. Why?**
> Possible causes:
> 1. You edited the wrong file (check path)
> 2. The seed is disabled (`+enabled: false`)
> 3. You forgot to save the file before running
> 4. You're running in wrong environment (dev vs prod)
>
> Fix: Try `dbt seed --select seed_name --full-refresh`

**Q27: Can you use seeds with incremental models?**
> Yes! Seeds can be referenced in incremental models. Common pattern: JOIN with a seed for enrichment. But the seed itself is always full-replace (not incremental).
