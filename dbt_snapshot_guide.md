# DBT SNAPSHOTS GUIDE: Slowly Changing Dimensions (SCD Type 2)

Snapshots capture how a source table changes over time. dbt implements SCD Type 2, which means when a row changes, it does NOT overwrite the old record. Instead, it keeps the old record with a validity range and inserts the new version alongside it.

This gives you a **FULL HISTORY** of every change made to every row.

---

## Table of Contents

1. [What Are Snapshots?](#1-what-are-snapshots)
2. [SCD Type 2 Explained](#2-scd-type-2-explained)
3. [Snapshot Strategies: Timestamp vs Check](#3-snapshot-strategies-timestamp-vs-check)
4. [Setup: Create Source Tables & Sample Data](#4-setup-create-source-tables--sample-data)
5. [Setup: Staging Models](#5-setup-staging-models)
6. [Snapshot: Timestamp Strategy (Products)](#6-snapshot-timestamp-strategy-products)
7. [Snapshot: Check Strategy (Employees)](#7-snapshot-check-strategy-employees)
8. [How to Run Snapshots](#8-how-to-run-snapshots)
9. [Simulating Changes & Re-running](#9-simulating-changes--re-running-snapshots)
10. [Querying Snapshot Tables](#10-querying-snapshot-tables)
11. [Directory Structure](#11-directory-structure)
12. [Schema Configuration](#12-schema-configuration-in-dbt_projectyml)
13. [Hard Delete vs Soft Delete](#13-hard-delete-vs-soft-delete)
14. [Invalidating Hard Deletes](#14-invalidating-hard-deletes)
15. [Key Differences: Timestamp vs Check](#15-key-differences-timestamp-vs-check-summary)
16. [Best Practices](#16-best-practices)

---

## 1. What Are Snapshots?

In a data warehouse, source data changes over time:
- A customer changes their email address
- A product price gets updated
- An employee gets promoted to a new role

**Without snapshots**, you only see the CURRENT state.
**With snapshots**, you see EVERY historical state with timestamps.

dbt snapshots give you these meta columns:

| Column | Purpose |
|--------|---------|
| `dbt_scd_id` | Unique key for each version of a row |
| `dbt_updated_at` | When this version was captured |
| `dbt_valid_from` | When this version became active |
| `dbt_valid_to` | When this version was superseded (`NULL` = current) |

---

## 2. SCD Type 2 Explained

SCD Type 2 preserves history by creating a **NEW ROW** for each change.

**Example:** Product "Laptop" price changes from 999.99 → 1099.99

### BEFORE snapshot (source table - only current state):

| PRODUCT_ID | NAME   | PRICE   |
|------------|--------|---------|
| 1          | Laptop | 1099.99 | ← old price 999.99 is LOST |

### AFTER snapshot (snapshot table - full history):

| PRODUCT_ID | NAME   | PRICE   | DBT_VALID_FROM      | DBT_VALID_TO        |
|------------|--------|---------|----------------------|---------------------|
| 1          | Laptop | 999.99  | 2025-01-01 00:00:00  | 2025-06-15 10:00:00 |
| 1          | Laptop | 1099.99 | 2025-06-15 10:00:00  | NULL                |

`NULL` in `DBT_VALID_TO` = **current row**

---

## 3. Snapshot Strategies: Timestamp vs Check

| | TIMESTAMP STRATEGY | CHECK STRATEGY |
|-|-|-|
| **Requires** | An `updated_at` column in the source | Does NOT need a timestamp column |
| **Detects changes by** | Comparing timestamps | Comparing column values directly |
| **Performance** | Faster on large tables | Slower (compares every checked column) |
| **Reliability** | More reliable | Only detects changes in specified columns |
| **Use when** | Source has a reliable `updated_at` | Source has NO timestamp column or you don't trust it |

---

## 4. Setup: Create Source Tables & Sample Data

Run these SQL statements in Snowflake to create the source tables.

### 4a. PRODUCTS table (has UPDATED_AT → good for timestamp strategy)

```sql
CREATE OR REPLACE TABLE DBT_PROJECT_DB.RAW.PRODUCTS (
    PRODUCT_ID    NUMBER,
    PRODUCT_NAME  VARCHAR(100),
    CATEGORY      VARCHAR(50),
    PRICE         NUMBER(10,2),
    IS_ACTIVE     BOOLEAN,
    UPDATED_AT    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _LOADED_AT    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

INSERT INTO DBT_PROJECT_DB.RAW.PRODUCTS
    (PRODUCT_ID, PRODUCT_NAME, CATEGORY, PRICE, IS_ACTIVE, UPDATED_AT)
VALUES
    (1,  'Laptop Pro 15',       'Electronics',  999.99,  TRUE,  '2025-01-01 00:00:00'),
    (2,  'Wireless Mouse',      'Electronics',  29.99,   TRUE,  '2025-01-01 00:00:00'),
    (3,  'Office Chair',        'Furniture',     249.99,  TRUE,  '2025-01-15 00:00:00'),
    (4,  'Standing Desk',       'Furniture',     549.99,  TRUE,  '2025-02-01 00:00:00'),
    (5,  'Noise Cancelling Headphones', 'Electronics', 199.99, TRUE, '2025-02-01 00:00:00'),
    (6,  'USB-C Hub',           'Accessories',   59.99,   TRUE,  '2025-03-01 00:00:00'),
    (7,  'Mechanical Keyboard', 'Accessories',   89.99,   TRUE,  '2025-03-01 00:00:00'),
    (8,  'Monitor 27 inch',     'Electronics',   399.99,  TRUE,  '2025-03-15 00:00:00'),
    (9,  'Webcam HD',           'Accessories',   49.99,   TRUE,  '2025-04-01 00:00:00'),
    (10, 'Laptop Stand',        'Accessories',   39.99,   TRUE,  '2025-04-01 00:00:00');
```

### 4b. EMPLOYEES table (NO updated_at → use check strategy)

```sql
CREATE OR REPLACE TABLE DBT_PROJECT_DB.RAW.EMPLOYEES (
    EMPLOYEE_ID   NUMBER,
    FULL_NAME     VARCHAR(100),
    DEPARTMENT    VARCHAR(50),
    JOB_TITLE     VARCHAR(100),
    SALARY        NUMBER(10,2),
    MANAGER_ID    NUMBER,
    _LOADED_AT    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

INSERT INTO DBT_PROJECT_DB.RAW.EMPLOYEES
    (EMPLOYEE_ID, FULL_NAME, DEPARTMENT, JOB_TITLE, SALARY, MANAGER_ID)
VALUES
    (101, 'Rohit Sharma',    'Engineering',  'Software Engineer',     85000.00,  201),
    (102, 'Priya Patel',     'Engineering',  'Senior Engineer',       110000.00, 201),
    (103, 'James Wilson',    'Marketing',    'Marketing Manager',     95000.00,  202),
    (104, 'Emily Brown',     'Sales',        'Sales Representative',  70000.00,  203),
    (105, 'Amit Kumar',      'Engineering',  'Tech Lead',             130000.00, 201),
    (106, 'Sarah Johnson',   'HR',           'HR Specialist',         75000.00,  204),
    (107, 'Michael Davis',   'Sales',        'Sales Manager',         105000.00, NULL),
    (108, 'Ananya Reddy',    'Engineering',  'Junior Developer',      65000.00,  105);
```

### 4c. Add these tables to your sources.yml

```yaml
version: 2
sources:
  - name: raw
    database: DBT_PROJECT_DB
    schema: RAW
    tables:
      - name: products
        description: "Product catalog with prices and categories"
      - name: employees
        description: "Employee directory with department and salary info"
```

---

## 5. Setup: Staging Models

Snapshots should read from **STAGING models**, not directly from raw sources. This ensures your data is clean and typed before snapshotting.

### 5a. File: `models/staging/stg_products.sql`

```sql
{{ config(materialized='view') }}

SELECT
    PRODUCT_ID,
    PRODUCT_NAME,
    CATEGORY,
    PRICE,
    IS_ACTIVE,
    UPDATED_AT,
    _LOADED_AT
FROM {{ source('raw', 'products') }}
```

### 5b. File: `models/staging/stg_employees.sql`

```sql
{{ config(materialized='view') }}

SELECT
    EMPLOYEE_ID,
    FULL_NAME,
    DEPARTMENT,
    JOB_TITLE,
    SALARY,
    MANAGER_ID,
    _LOADED_AT
FROM {{ source('raw', 'employees') }}
```

---

## 6. Snapshot: Timestamp Strategy (Products)

Use this when your source has a reliable `UPDATED_AT` column. dbt compares the `UPDATED_AT` value to decide if a row has changed.

### File: `snapshots/snap_products.sql`

```sql
{% snapshot snap_products %}

{{
    config(
        target_database = 'DBT_PROJECT_DB',
        target_schema   = 'SNAPSHOTS',
        unique_key      = 'PRODUCT_ID',
        strategy        = 'timestamp',
        updated_at      = 'UPDATED_AT'
    )
}}

SELECT
    PRODUCT_ID,
    PRODUCT_NAME,
    CATEGORY,
    PRICE,
    IS_ACTIVE,
    UPDATED_AT
FROM {{ ref('stg_products') }}

{% endsnapshot %}
```

### How It Works (Step by Step)

1. **First run** (`dbt snapshot`):
   - dbt creates the `SNAP_PRODUCTS` table in `DBT_PROJECT_DB.SNAPSHOTS`
   - Inserts ALL rows from `stg_products`
   - Sets `dbt_valid_from = UPDATED_AT`, `dbt_valid_to = NULL`

2. **Subsequent runs:**
   - dbt compares `UPDATED_AT` in source vs snapshot
   - If `UPDATED_AT` is newer → old row gets `dbt_valid_to = now()`, new row inserted with `dbt_valid_to = NULL`
   - If `UPDATED_AT` is same → no change, row is skipped
   - New rows (new `PRODUCT_ID`) are inserted fresh

### Detailed Example with Source Data

**SOURCE TABLE** (`stg_products`) on Jan 1, 2025:

| PRODUCT_ID | PRODUCT_NAME  | PRICE  | UPDATED_AT          |
|------------|---------------|--------|---------------------|
| 1          | Laptop Pro 15 | 999.99 | 2025-01-01 00:00:00 |
| 2          | Wireless Mouse| 29.99  | 2025-01-01 00:00:00 |

---

#### STEP 1: First `dbt snapshot` run (Jan 1, 2025)

dbt sees NO existing snapshot table → creates it and inserts ALL rows.
`dbt_valid_from` = source `UPDATED_AT` value, `dbt_valid_to` = NULL (all rows are current)

**SNAP_PRODUCTS after first run:**

| PRODUCT_ID | PRODUCT_NAME  | PRICE  | DBT_VALID_FROM      | DBT_VALID_TO | DBT_UPDATED_AT      |
|------------|---------------|--------|----------------------|--------------|---------------------|
| 1          | Laptop Pro 15 | 999.99 | 2025-01-01 00:00:00  | NULL         | 2025-01-01 00:00:00 |
| 2          | Wireless Mouse| 29.99  | 2025-01-01 00:00:00  | NULL         | 2025-01-01 00:00:00 |

Both rows have `dbt_valid_to = NULL` → they are **CURRENT** versions.

---

#### STEP 2: Source data changes (Feb 15, 2025)

Someone runs:
```sql
UPDATE products SET price=1099.99, updated_at='2025-02-15 09:30:00' WHERE product_id = 1;
```

**SOURCE TABLE now:**

| PRODUCT_ID | PRODUCT_NAME  | PRICE   | UPDATED_AT          |
|------------|---------------|---------|---------------------|
| 1          | Laptop Pro 15 | 1099.99 | 2025-02-15 09:30:00 | ← CHANGED |
| 2          | Wireless Mouse| 29.99   | 2025-01-01 00:00:00 | ← same |

---

#### STEP 3: Second `dbt snapshot` run (Feb 15, 2025)

dbt compares each row's `UPDATED_AT` in source vs snapshot:

- **PRODUCT_ID=1:** source `updated_at` (2025-02-15) > snapshot `updated_at` (2025-01-01)
  → **CHANGE DETECTED!**
  → Old row: set `dbt_valid_to = 2025-02-15 09:30:00` (no longer current)
  → New row: insert with `dbt_valid_from = 2025-02-15 09:30:00`, `dbt_valid_to = NULL`

- **PRODUCT_ID=2:** source `updated_at` (2025-01-01) = snapshot `updated_at` (2025-01-01)
  → **NO CHANGE** → skip, do nothing

**SNAP_PRODUCTS after second run:**

| PRODUCT_ID | PRODUCT_NAME  | PRICE   | DBT_VALID_FROM      | DBT_VALID_TO        | DBT_UPDATED_AT      | Status |
|------------|---------------|---------|----------------------|---------------------|---------------------|--------|
| 1          | Laptop Pro 15 | 999.99  | 2025-01-01 00:00:00  | 2025-02-15 09:30:00 | 2025-01-01 00:00:00 | EXPIRED |
| 1          | Laptop Pro 15 | 1099.99 | 2025-02-15 09:30:00  | NULL                | 2025-02-15 09:30:00 | CURRENT |
| 2          | Wireless Mouse| 29.99   | 2025-01-01 00:00:00  | NULL                | 2025-01-01 00:00:00 | CURRENT (unchanged) |

---

#### STEP 4: New product added + another price change (Mar 10, 2025)

```sql
INSERT INTO products VALUES (3, 'USB-C Hub', 59.99, '2025-03-10 14:00:00');
UPDATE products SET price=34.99, updated_at='2025-03-10 14:00:00' WHERE product_id=2;
```

**SOURCE TABLE now:**

| PRODUCT_ID | PRODUCT_NAME  | PRICE   | UPDATED_AT          |
|------------|---------------|---------|---------------------|
| 1          | Laptop Pro 15 | 1099.99 | 2025-02-15 09:30:00 | ← same |
| 2          | Wireless Mouse| 34.99   | 2025-03-10 14:00:00 | ← CHANGED |
| 3          | USB-C Hub     | 59.99   | 2025-03-10 14:00:00 | ← NEW ROW |

---

#### STEP 5: Third `dbt snapshot` run (Mar 10, 2025)

dbt compares:
- **PRODUCT_ID=1:** no change → skip
- **PRODUCT_ID=2:** `updated_at` changed → invalidate old, insert new
- **PRODUCT_ID=3:** new `unique_key` not in snapshot → insert fresh

**SNAP_PRODUCTS after third run:**

| PRODUCT_ID | PRODUCT_NAME  | PRICE   | DBT_VALID_FROM      | DBT_VALID_TO        | DBT_UPDATED_AT      | Status |
|------------|---------------|---------|----------------------|---------------------|---------------------|--------|
| 1          | Laptop Pro 15 | 999.99  | 2025-01-01 00:00:00  | 2025-02-15 09:30:00 | 2025-01-01 00:00:00 | EXPIRED |
| 1          | Laptop Pro 15 | 1099.99 | 2025-02-15 09:30:00  | NULL                | 2025-02-15 09:30:00 | CURRENT |
| 2          | Wireless Mouse| 29.99   | 2025-01-01 00:00:00  | 2025-03-10 14:00:00 | 2025-01-01 00:00:00 | EXPIRED |
| 2          | Wireless Mouse| 34.99   | 2025-03-10 14:00:00  | NULL                | 2025-03-10 14:00:00 | CURRENT |
| 3          | USB-C Hub     | 59.99   | 2025-03-10 14:00:00  | NULL                | 2025-03-10 14:00:00 | CURRENT (new) |

**Key Takeaways:**
- PRODUCT_ID=1 has 2 rows: price was 999.99 before Feb 15, then 1099.99 after
- PRODUCT_ID=2 has 2 rows: price went from 29.99 to 34.99 on Mar 10
- PRODUCT_ID=3 has 1 row: brand new, no history yet
- `dbt_valid_to = NULL` always means "this is the current version"
- To get current data only: `SELECT * FROM snap_products WHERE dbt_valid_to IS NULL`
- To get data as of Feb 1: `WHERE dbt_valid_from <= '2025-02-01' AND (dbt_valid_to > '2025-02-01' OR dbt_valid_to IS NULL)`

---

## 7. Snapshot: Check Strategy (Employees)

Use this when your source has **NO** `updated_at` column. dbt compares actual column **VALUES** to detect changes.

### File: `snapshots/snap_employees.sql`

```sql
{% snapshot snap_employees %}

{{
    config(
        target_database = 'DBT_PROJECT_DB',
        target_schema   = 'SNAPSHOTS',
        unique_key      = 'EMPLOYEE_ID',
        strategy        = 'check',
        check_cols      = ['DEPARTMENT', 'JOB_TITLE', 'SALARY', 'MANAGER_ID']
    )
}}

SELECT
    EMPLOYEE_ID,
    FULL_NAME,
    DEPARTMENT,
    JOB_TITLE,
    SALARY,
    MANAGER_ID
FROM {{ ref('stg_employees') }}

{% endsnapshot %}
```

### How It Works (Step by Step)

1. **First run** (`dbt snapshot`):
   - Creates `SNAP_EMPLOYEES` in `DBT_PROJECT_DB.SNAPSHOTS`
   - Inserts ALL rows
   - Sets `dbt_valid_from = now()`, `dbt_valid_to = NULL`

2. **Subsequent runs:**
   - dbt compares each `check_col` value in source vs snapshot
   - If ANY checked column changed → old row invalidated, new row inserted
   - If all checked columns same → no change

> **NOTE:** `check_cols = 'all'` will compare EVERY column (slower, but thorough)

### Detailed Example with Source Data

`check_cols = ['DEPARTMENT', 'JOB_TITLE', 'SALARY', 'MANAGER_ID']`
(dbt will ONLY compare these 4 columns to detect changes)

**SOURCE TABLE** (`stg_employees`) on Jan 1, 2025:

| EMPLOYEE_ID | FULL_NAME     | DEPARTMENT  | JOB_TITLE | SALARY | MANAGER_ID |
|-------------|---------------|-------------|-----------|--------|------------|
| 101         | Alice Johnson | Engineering | Developer | 90000  | 501        |
| 102         | Bob Smith     | Marketing   | Analyst   | 75000  | 502        |

---

#### STEP 1: First `dbt snapshot` run (Jan 1, 2025 at 10:00 AM)

No snapshot table exists → dbt creates it and inserts ALL rows.
Since there's NO `updated_at` column, `dbt_valid_from = now()` (the time dbt runs)

**SNAP_EMPLOYEES after first run:**

| EMPLOYEE_ID | FULL_NAME     | DEPARTMENT  | JOB_TITLE | SALARY | MANAGER_ID | DBT_VALID_FROM      | DBT_VALID_TO | DBT_UPDATED_AT      |
|-------------|---------------|-------------|-----------|--------|------------|----------------------|--------------|---------------------|
| 101         | Alice Johnson | Engineering | Developer | 90000  | 501        | 2025-01-01 10:00:00  | NULL         | 2025-01-01 10:00:00 |
| 102         | Bob Smith     | Marketing   | Analyst   | 75000  | 502        | 2025-01-01 10:00:00  | NULL         | 2025-01-01 10:00:00 |

`dbt_valid_to = NULL` → both rows are **CURRENT**.

---

#### STEP 2: Source data changes (sometime before Feb 20)

Alice got promoted and a raise. Bob is unchanged.

**SOURCE TABLE now:**

| EMPLOYEE_ID | FULL_NAME     | DEPARTMENT  | JOB_TITLE    | SALARY | MANAGER_ID |
|-------------|---------------|-------------|--------------|--------|------------|
| 101         | Alice Johnson | Engineering | Sr Developer | 110000 | 501        | ← JOB_TITLE & SALARY changed |
| 102         | Bob Smith     | Marketing   | Analyst      | 75000  | 502        | ← same |

---

#### STEP 3: Second `dbt snapshot` run (Feb 20, 2025 at 10:00 AM)

dbt compares `check_cols` (DEPARTMENT, JOB_TITLE, SALARY, MANAGER_ID):

- **EMPLOYEE_ID=101:**
  - snapshot: `JOB_TITLE='Developer'`, `SALARY=90000`
  - source: `JOB_TITLE='Sr Developer'`, `SALARY=110000`
  → **CHANGE DETECTED** (2 of 4 check_cols differ)
  → Old row: set `dbt_valid_to = 2025-02-20 10:00:00`
  → New row: insert with `dbt_valid_from = 2025-02-20 10:00:00`, `dbt_valid_to = NULL`

- **EMPLOYEE_ID=102:**
  - All check_cols same → **skip**

**SNAP_EMPLOYEES after second run:**

| EMPLOYEE_ID | FULL_NAME     | DEPARTMENT  | JOB_TITLE    | SALARY | MANAGER_ID | DBT_VALID_FROM      | DBT_VALID_TO        | DBT_UPDATED_AT      | Status |
|-------------|---------------|-------------|--------------|--------|------------|----------------------|---------------------|---------------------|--------|
| 101         | Alice Johnson | Engineering | Developer    | 90000  | 501        | 2025-01-01 10:00:00  | 2025-02-20 10:00:00 | 2025-01-01 10:00:00 | EXPIRED |
| 101         | Alice Johnson | Engineering | Sr Developer | 110000 | 501        | 2025-02-20 10:00:00  | NULL                | 2025-02-20 10:00:00 | CURRENT |
| 102         | Bob Smith     | Marketing   | Analyst      | 75000  | 502        | 2025-01-01 10:00:00  | NULL                | 2025-01-01 10:00:00 | CURRENT (unchanged) |

---

#### STEP 4: Bob changes department + new employee (before Apr 5)

**SOURCE TABLE now:**

| EMPLOYEE_ID | FULL_NAME     | DEPARTMENT  | JOB_TITLE    | SALARY | MANAGER_ID |
|-------------|---------------|-------------|--------------|--------|------------|
| 101         | Alice Johnson | Engineering | Sr Developer | 110000 | 501        | ← same |
| 102         | Bob Smith     | Sales       | Analyst      | 75000  | 503        | ← DEPARTMENT & MANAGER_ID changed |
| 103         | Carol Lee     | Engineering | Intern       | 45000  | 501        | ← NEW EMPLOYEE |

---

#### STEP 5: Third `dbt snapshot` run (Apr 5, 2025 at 10:00 AM)

- **EMPLOYEE_ID=101:** all same → skip
- **EMPLOYEE_ID=102:** DEPARTMENT & MANAGER_ID differ → invalidate old, insert new
- **EMPLOYEE_ID=103:** new unique_key → insert fresh

**SNAP_EMPLOYEES after third run:**

| EMPLOYEE_ID | FULL_NAME     | DEPARTMENT  | JOB_TITLE    | SALARY | MANAGER_ID | DBT_VALID_FROM      | DBT_VALID_TO        | DBT_UPDATED_AT      | Status |
|-------------|---------------|-------------|--------------|--------|------------|----------------------|---------------------|---------------------|--------|
| 101         | Alice Johnson | Engineering | Developer    | 90000  | 501        | 2025-01-01 10:00:00  | 2025-02-20 10:00:00 | 2025-01-01 10:00:00 | EXPIRED |
| 101         | Alice Johnson | Engineering | Sr Developer | 110000 | 501        | 2025-02-20 10:00:00  | NULL                | 2025-02-20 10:00:00 | CURRENT |
| 102         | Bob Smith     | Marketing   | Analyst      | 75000  | 502        | 2025-01-01 10:00:00  | 2025-04-05 10:00:00 | 2025-01-01 10:00:00 | EXPIRED |
| 102         | Bob Smith     | Sales       | Analyst      | 75000  | 503        | 2025-04-05 10:00:00  | NULL                | 2025-04-05 10:00:00 | CURRENT |
| 103         | Carol Lee     | Engineering | Intern       | 45000  | 501        | 2025-04-05 10:00:00  | NULL                | 2025-04-05 10:00:00 | CURRENT (new) |

**Key Difference vs Timestamp Strategy:**
- **Timestamp strategy:** `dbt_valid_from` = source's `UPDATED_AT` value (exact time of change)
- **Check strategy:** `dbt_valid_from` = `now()` at snapshot run time (you only know WHEN dbt detected it, not when it actually changed)
- Check is useful when source has NO `updated_at` column, but timestamps are less precise

### Option: `check_cols = 'all'`

Instead of listing specific columns, you can monitor ALL columns:

```sql
{{
    config(
        ...
        strategy   = 'check',
        check_cols = 'all'
    )
}}
```

---

## 8. How to Run Snapshots

```bash
# Run ALL snapshots
dbt snapshot

# Run a specific snapshot
dbt snapshot --select snap_products

# Run multiple specific snapshots
dbt snapshot --select snap_products snap_employees
```

> **IMPORTANT:**
> - Snapshots are **NOT** included in `dbt run` or `dbt build`
> - You must explicitly run `dbt snapshot`
> - Snapshots should run on a **SCHEDULE** (e.g., daily) to capture changes

---

## 9. Simulating Changes & Re-running Snapshots

After the first `dbt snapshot`, simulate changes to see SCD Type 2 in action.

### Step 1: Run initial snapshot

```bash
dbt snapshot
```

### Step 2: Simulate price changes in PRODUCTS

```sql
UPDATE DBT_PROJECT_DB.RAW.PRODUCTS
SET PRICE = 1099.99, UPDATED_AT = CURRENT_TIMESTAMP()
WHERE PRODUCT_ID = 1;  -- Laptop Pro 15: 999.99 → 1099.99

SELECT * FROM DBT_PROJECT_DB.RAW.PRODUCTS;
SELECT * FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_PRODUCTS ORDER BY PRODUCT_ID;

UPDATE DBT_PROJECT_DB.RAW.PRODUCTS
SET PRICE = 34.99, UPDATED_AT = CURRENT_TIMESTAMP()
WHERE PRODUCT_ID = 2;  -- Wireless Mouse: 29.99 → 34.99

UPDATE DBT_PROJECT_DB.RAW.PRODUCTS
SET IS_ACTIVE = FALSE, UPDATED_AT = CURRENT_TIMESTAMP()
WHERE PRODUCT_ID = 9;  -- Webcam HD: discontinued
```

### Step 3: Simulate employee changes

```sql
UPDATE DBT_PROJECT_DB.RAW.EMPLOYEES
SET JOB_TITLE = 'Senior Software Engineer', SALARY = 105000.00
WHERE EMPLOYEE_ID = 101;  -- Rohit got promoted!

SELECT * FROM DBT_PROJECT_DB.RAW.EMPLOYEES;

UPDATE DBT_PROJECT_DB.RAW.EMPLOYEES
SET DEPARTMENT = 'Engineering', JOB_TITLE = 'Data Engineer'
WHERE EMPLOYEE_ID = 103;  -- James moved from Marketing → Engineering
```

### Step 4: Add a new employee

```sql
INSERT INTO DBT_PROJECT_DB.RAW.EMPLOYEES
    (EMPLOYEE_ID, FULL_NAME, DEPARTMENT, JOB_TITLE, SALARY, MANAGER_ID)
VALUES
    (109, 'Vikram Singh', 'Engineering', 'DevOps Engineer', 95000.00, 105);
```

### Step 5: Re-run snapshots to capture changes

```bash
dbt snapshot
```

### Step 6: Query the snapshot tables to see the history!

(See Section 10 below)

---

## 10. Querying Snapshot Tables

### 10a. See ALL historical versions of a product

```sql
SELECT
    PRODUCT_ID,
    PRODUCT_NAME,
    PRICE,
    IS_ACTIVE,
    DBT_VALID_FROM,
    DBT_VALID_TO,
    CASE WHEN DBT_VALID_TO IS NULL THEN 'CURRENT' ELSE 'EXPIRED' END AS ROW_STATUS
FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_PRODUCTS
WHERE PRODUCT_ID = 1
ORDER BY DBT_VALID_FROM;
```

**Expected output:**

| PRODUCT_ID | PRODUCT_NAME  | PRICE   | IS_ACTIVE | DBT_VALID_FROM      | DBT_VALID_TO        | ROW_STATUS |
|------------|---------------|---------|-----------|----------------------|---------------------|------------|
| 1          | Laptop Pro 15 | 999.99  | TRUE      | 2025-01-01 00:00:00  | 2025-xx-xx xx:xx:xx | EXPIRED    |
| 1          | Laptop Pro 15 | 1099.99 | TRUE      | 2025-xx-xx xx:xx:xx  | NULL                | CURRENT    |

### 10b. Get CURRENT state only (latest version of each row)

```sql
SELECT *
FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_PRODUCTS
WHERE DBT_VALID_TO IS NULL;
```

### 10c. Get the state of products AT A SPECIFIC POINT IN TIME

```sql
SELECT *
FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_PRODUCTS
WHERE DBT_VALID_FROM <= '2025-03-01 00:00:00'
  AND (DBT_VALID_TO > '2025-03-01 00:00:00' OR DBT_VALID_TO IS NULL);
```

### 10d. Track employee role changes over time

```sql
SELECT
    EMPLOYEE_ID,
    FULL_NAME,
    JOB_TITLE,
    SALARY,
    DEPARTMENT,
    DBT_VALID_FROM,
    DBT_VALID_TO,
    CASE WHEN DBT_VALID_TO IS NULL THEN 'CURRENT' ELSE 'PREVIOUS' END AS VERSION
FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_EMPLOYEES
WHERE EMPLOYEE_ID = 101
ORDER BY DBT_VALID_FROM;
```

Expected: Shows Rohit's promotion from Software Engineer → Senior Software Engineer

### 10e. Find all employees who changed departments

```sql
SELECT
    a.EMPLOYEE_ID,
    a.FULL_NAME,
    a.DEPARTMENT   AS OLD_DEPARTMENT,
    b.DEPARTMENT   AS NEW_DEPARTMENT,
    a.JOB_TITLE    AS OLD_TITLE,
    b.JOB_TITLE    AS NEW_TITLE,
    b.DBT_VALID_FROM AS CHANGED_AT
FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_EMPLOYEES a
JOIN DBT_PROJECT_DB.SNAPSHOTS.SNAP_EMPLOYEES b
  ON a.EMPLOYEE_ID = b.EMPLOYEE_ID
  AND a.DBT_VALID_TO = b.DBT_VALID_FROM
WHERE a.DEPARTMENT != b.DEPARTMENT;
```

### 10f. Count how many times each product's price changed

```sql
SELECT
    PRODUCT_ID,
    PRODUCT_NAME,
    COUNT(*) - 1 AS PRICE_CHANGES,
    MIN(PRICE)   AS LOWEST_PRICE,
    MAX(PRICE)   AS HIGHEST_PRICE
FROM DBT_PROJECT_DB.SNAPSHOTS.SNAP_PRODUCTS
GROUP BY PRODUCT_ID, PRODUCT_NAME
HAVING COUNT(*) > 1
ORDER BY PRICE_CHANGES DESC;
```

---

## 11. Directory Structure

```
my_dbt_project/
├── dbt_project.yml
├── models/
│   └── staging/
│       ├── stg_products.sql          ← staging view
│       └── stg_employees.sql         ← staging view
├── snapshots/
│   ├── snap_products.sql             ← timestamp strategy
│   └── snap_employees.sql            ← check strategy
└── ...
```

> **NOTE:** Snapshots live in the `snapshots/` directory, **NOT** in `models/`. This is a dbt convention. dbt looks for `{% snapshot %}` blocks in this folder.

---

## 12. Schema Configuration in dbt_project.yml

Add this to your `dbt_project.yml` to control where snapshots land:

```yaml
snapshots:
  dbt_practice:                    # your project name
    +target_database: DBT_PROJECT_DB
    +target_schema: SNAPSHOTS      # all snapshots go to this schema
```

You can also override per-snapshot using the `config()` block (as shown above).

> **IMPORTANT:** The target schema (SNAPSHOTS) must exist:
> ```sql
> CREATE SCHEMA IF NOT EXISTS DBT_PROJECT_DB.SNAPSHOTS;
> ```

---

## 13. Hard Delete vs Soft Delete

| | HARD DELETE | SOFT DELETE |
|-|-------------|-------------|
| **What happens** | Row is physically REMOVED from the table (`DELETE FROM`) | Row stays but a flag marks it deleted (e.g., `IS_DELETED = TRUE`) |
| **Can you recover it?** | No (unless backups or Time Travel) | Yes, the data is still there |
| **Snapshot impact** | dbt does NOT detect it by default — the deleted row stays as `dbt_valid_to = NULL` (appears still active) | dbt detects the flag change and creates a new snapshot version showing `IS_DELETED = TRUE` |
| **Fix in dbt** | Set `invalidate_hard_deletes = True` (see Section 14) | No special config needed — normal snapshot handles it |
| **Example** | `DELETE FROM products WHERE product_id = 5;` | `UPDATE products SET is_deleted = TRUE WHERE product_id = 5;` |
| **Best practice** | Avoid in analytical systems; use soft deletes instead | Preferred for data warehouses — preserves full audit trail |

### Soft Delete Example in a Snapshot

If your source uses a soft delete pattern (`IS_ACTIVE = FALSE` or `IS_DELETED = TRUE`), the snapshot will naturally capture this as a column value change.

Source row changes: `IS_ACTIVE = TRUE` → `IS_ACTIVE = FALSE`

**Snapshot result:**

| PRODUCT_ID | IS_ACTIVE | DBT_VALID_FROM      | DBT_VALID_TO        |
|------------|-----------|----------------------|---------------------|
| 5          | TRUE      | 2025-02-01 00:00:00  | 2025-06-10 12:00:00 |
| 5          | FALSE     | 2025-06-10 12:00:00  | NULL                |

**Hard Delete:** Row disappears from source → snapshot still shows `dbt_valid_to = NULL` (stale/ghost row) unless you enable `invalidate_hard_deletes`.

---

## 14. Invalidating Hard Deletes

By default, if a row is DELETED from the source, the snapshot keeps it as a current row (`dbt_valid_to = NULL`). It doesn't know it was deleted.

To handle hard deletes, add `invalidate_hard_deletes = True`:

```sql
{% snapshot snap_products_with_deletes %}

{{
    config(
        target_database        = 'DBT_PROJECT_DB',
        target_schema          = 'SNAPSHOTS',
        unique_key             = 'PRODUCT_ID',
        strategy               = 'timestamp',
        updated_at             = 'UPDATED_AT',
        invalidate_hard_deletes = True
    )
}}

SELECT * FROM {{ ref('stg_products') }}

{% endsnapshot %}
```

With this config, if a product is deleted from the source:
- Its snapshot row gets `dbt_valid_to = now()` (marked as no longer current)
- You can identify deleted records: `WHERE dbt_valid_to IS NOT NULL`

---

## 15. Key Differences: Timestamp vs Check (Summary)

| FEATURE | TIMESTAMP | CHECK |
|---------|-----------|-------|
| **Requires column** | Yes (`updated_at`) | No |
| **Detects changes via** | Timestamp comparison | Column value comparison |
| **Performance** | Fast (single comparison) | Slower (multi-column) |
| **Reliability** | High (if timestamp is always updated) | May miss changes if cols not in `check_cols` change |
| **dbt_valid_from** | = source `updated_at` | = snapshot run time |
| **Config required** | `strategy = 'timestamp'`, `updated_at = 'COL_NAME'` | `strategy = 'check'`, `check_cols = [...]` |
| **Best for** | Tables with audit columns | Tables without timestamps |

---

## 16. Best Practices

1. **ALWAYS snapshot from staging models, not raw sources**
   → Ensures clean, typed data enters your snapshot

2. **Use TIMESTAMP strategy whenever possible**
   → It's faster and more reliable than check

3. **Run snapshots on a SCHEDULE**
   → If you only run weekly, you'll miss mid-week changes
   → Recommended: daily or more frequently for critical data

4. **Keep your unique_key truly unique**
   → Duplicates cause incorrect SCD tracking
   → Add a dbt test: `unique` on your unique_key column

5. **Don't SELECT * in snapshots**
   → Explicitly list columns to avoid breaking changes when source adds columns

6. **Use a dedicated schema (e.g., SNAPSHOTS)**
   → Keeps snapshot tables organized and separate from models

7. **Add tests to your snapshot in schema.yml:**
   → Test `unique` on `dbt_scd_id`
   → Test `not_null` on `unique_key`, `dbt_valid_from`

8. **Handle hard deletes if your source deletes rows**
   → Set `invalidate_hard_deletes = True`

9. **Never modify the snapshot table directly**
   → dbt manages the SCD columns; manual changes will corrupt history

---

## Quick Reference: Complete Workflow

| Step | Action |
|------|--------|
| 1 | Create source tables → Run Section 4 SQL |
| 2 | Create SNAPSHOTS schema → `CREATE SCHEMA IF NOT EXISTS DBT_PROJECT_DB.SNAPSHOTS;` |
| 3 | Add sources to `sources.yml` → See Section 4c |
| 4 | Create staging models → See Section 5 |
| 5 | Create snapshot files → See Sections 6 & 7 |
| 6 | Run first snapshot → `dbt snapshot` |
| 7 | Simulate changes → Run Section 9 SQL |
| 8 | Re-run snapshot → `dbt snapshot` |
| 9 | Query history → Run Section 10 queries |
