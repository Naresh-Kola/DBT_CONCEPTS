# DBT TESTS: COMPLETE GUIDE (SCRATCH TO PRO)

Tests in dbt are assertions about your data. They verify that your transformations produce correct, reliable results. If a test fails, dbt returns an error — alerting you BEFORE bad data reaches production.

---

## Table of Contents

1. [Why Do We Need Tests?](#section-1-why-do-we-need-tests)
2. [Types of Tests in dbt](#section-2-types-of-tests-in-dbt)
3. [Generic Tests (Built-in)](#section-3-generic-tests-built-in)
4. [What Each Built-in Test Does (Under the Hood)](#section-4-what-each-built-in-test-does-under-the-hood)
5. [Singular Tests](#section-5-singular-tests)
6. [Test Severity & Thresholds](#section-6-test-severity--thresholds)
7. [Custom Generic Tests](#section-7-custom-generic-tests)
8. [Source Tests](#section-8-source-tests)
9. [dbt_utils Package Tests](#section-9-dbt_utils-package-tests)
10. [dbt_expectations Package Tests](#section-10-dbt_expectations-package-tests)
11. [Test Selection & Commands](#section-11-test-selection--commands)
12. [Store Failures](#section-12-store-failures)
13. [Test Tags & Organization](#section-13-test-tags--organization)
13A. [Deep Dive — freshness, tags, and ref() Explained](#section-13a-deep-dive--freshness-tags-and-ref-explained)
14. [Real-Life Example — E-Commerce Data Pipeline](#section-14-real-life-example--e-commerce-data-pipeline)
15. [Quick Reference Cheat Sheet](#section-15-quick-reference-cheat-sheet)

---

## SECTION 1: WHY DO WE NEED TESTS?

Without tests, you have NO guarantee that:
- A column that should be unique actually IS unique
- Required fields are not null
- Foreign keys reference valid records
- Business logic is correct (e.g., revenue >= 0)

dbt tests act as a safety net for your data pipeline.

Run them with:
```bash
dbt test
```
Or combined:
```bash
dbt build    # runs models + tests together
```

---

## SECTION 2: TYPES OF TESTS IN DBT

dbt has **TWO** main categories of tests:

1. **GENERIC TESTS** — defined in YAML, reusable, parameterized
2. **SINGULAR TESTS** — standalone SQL files in `/tests` folder

Additionally, with packages like `dbt_expectations` and `dbt_utils`, you get **HUNDREDS** of pre-built generic tests.

---

## SECTION 3: GENERIC TESTS (Built-in)

dbt ships with 4 built-in generic tests. You define them in your `schema.yml` (or any `.yml` file in your models directory).

| # | Test | Purpose |
|---|------|---------|
| 3.1 | `not_null` | Ensures a column has no NULL values |
| 3.2 | `unique` | Ensures all values in a column are unique |
| 3.3 | `accepted_values` | Ensures column values are within an allowed set |
| 3.4 | `relationships` | Ensures referential integrity (foreign key check) |

### Example YAML: `models/schema.yml`

```yaml
version: 2

models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - not_null
          - unique

      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'returned']

      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('customers')
              field: customer_id
```

---

## SECTION 4: WHAT EACH BUILT-IN TEST DOES (Under the Hood)

### 4.1 not_null

Compiled SQL:
```sql
SELECT order_id
FROM {{ ref('orders') }}
WHERE order_id IS NULL;
```
If this returns ANY rows → **TEST FAILS**

### 4.2 unique

Compiled SQL:
```sql
SELECT order_id, COUNT(*)
FROM {{ ref('orders') }}
GROUP BY order_id
HAVING COUNT(*) > 1;
```
If this returns ANY rows → **TEST FAILS** (duplicates exist)

### 4.3 accepted_values

Compiled SQL:
```sql
SELECT status
FROM {{ ref('orders') }}
WHERE status NOT IN ('placed', 'shipped', 'completed', 'returned');
```
If this returns ANY rows → **TEST FAILS** (unexpected values)

### 4.4 relationships

**PURPOSE:** Find "orphan" records — orders that reference a `customer_id` which does NOT exist in the customers table.

**HOW IT WORKS (step by step):**

**STEP 1: LEFT JOIN orders → customers**

Every order row is kept. If a matching customer exists, the customer columns are filled in. If NO match, customer columns = NULL.

```
orders (left)            customers (right)         Result after LEFT JOIN
┌──────┬─────────────┐   ┌─────────────┐          ┌──────┬──────────┬──────────────┐
│ o_id │ customer_id │   │ customer_id │          │ o_id │ o.cust   │ c.cust       │
├──────┼─────────────┤   ├─────────────┤          ├──────┼──────────┼──────────────┤
│  1   │     101     │   │     101     │   →      │  1   │   101    │   101        │ ← match
│  2   │     102     │   │     103     │   →      │  2   │   102    │   NULL       │ ← NO match (orphan!)
│  3   │     103     │   └─────────────┘   →      │  3   │   103    │   103        │ ← match
│  4   │     NULL    │                     →      │  4   │   NULL   │   NULL       │ ← null customer_id
└──────┴─────────────┘                            └──────┴──────────┴──────────────┘
```

**STEP 2: WHERE c.customer_id IS NULL**

Filters to rows where the JOIN found NO match (customer doesn't exist). This catches rows 2 and 4 from above.

**STEP 3: AND o.customer_id IS NOT NULL**

Excludes rows where the order itself has a NULL customer_id (row 4). We only care about orders that CLAIM to have a customer but that customer doesn't exist. NULL customer_id is a separate concern (handled by a `not_null` test).

**FINAL RESULT:** Only row 2 is returned (order_id=2, customer_id=102). Customer 102 does NOT exist in customers table → this is an **ORPHAN**.

Compiled SQL:
```sql
SELECT o.customer_id
FROM {{ ref('orders') }} o
LEFT JOIN {{ ref('customers') }} c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL AND o.customer_id IS NOT NULL;
```
If this returns ANY rows → **TEST FAILS** (orphan records found)

**REAL WORLD ANALOGY:**
Imagine a hotel check-in list (orders) referencing room numbers (customers). This test finds check-ins that reference rooms that DON'T EXIST in the hotel. Room "NULL" (no room assigned) is ignored — that's a different problem.

---

## SECTION 5: SINGULAR TESTS

Singular tests are standalone `.sql` files placed in the `/tests` directory. Each file contains a SELECT query. If the query returns ANY rows, the test **FAILS**.

Use singular tests for complex, one-off business logic validations.

### Example: `tests/assert_total_payment_is_positive.sql`

```sql
SELECT
    order_id,
    SUM(amount) AS total_amount
FROM {{ ref('payments') }}
GROUP BY order_id
HAVING total_amount < 0
```

### Example: `tests/assert_orders_have_payments.sql`

```sql
SELECT
    o.order_id
FROM {{ ref('orders') }} o
LEFT JOIN {{ ref('payments') }} p ON o.order_id = p.order_id
WHERE p.order_id IS NULL
  AND o.status = 'completed'
```

---

## SECTION 6: TEST SEVERITY & THRESHOLDS

By default, any failing row causes **ERROR**. You can configure:
- `severity: warn` → dbt warns but does NOT fail the run
- `error_if:` → fail only if row count exceeds threshold
- `warn_if:` → warn only if row count exceeds threshold

### Example YAML with severity config:

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - not_null:
              severity: warn

          - unique:
              config:
                severity: error
                error_if: ">10"
                warn_if: ">5"
```

---

## SECTION 7: CUSTOM GENERIC TESTS

You can write your OWN reusable generic tests by creating a macro in the `/macros` or `/tests/generic` directory.

### Example: `macros/test_is_positive.sql`

```sql
{% test is_positive(model, column_name) %}

SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

**Usage in YAML:**
```yaml
models:
  - name: payments
    columns:
      - name: amount
        tests:
          - is_positive
```

### Example: `macros/test_row_count_between.sql`

```sql
{% test row_count_between(model, min_count, max_count) %}

SELECT 1
FROM (
    SELECT COUNT(*) AS row_count
    FROM {{ model }}
) sub
WHERE row_count < {{ min_count }} OR row_count > {{ max_count }}

{% endtest %}
```

**Usage in YAML:**
```yaml
models:
  - name: orders
    tests:
      - row_count_between:
          min_count: 100
          max_count: 1000000
```

---

## SECTION 8: SOURCE TESTS

Tests are NOT just for models! You can test raw source tables too. This catches bad data BEFORE it enters your pipeline.

### Example: `models/sources.yml`

```yaml
version: 2

sources:
  - name: raw_ecommerce
    database: RAW_DB
    schema: PUBLIC
    tables:
      - name: raw_orders
        columns:
          - name: id
            tests:
              - not_null
              - unique
          - name: order_date
            tests:
              - not_null

      - name: raw_customers
        columns:
          - name: id
            tests:
              - not_null
              - unique
```

---

## SECTION 9: dbt_utils PACKAGE TESTS

**WHAT:** A library of reusable tests from dbt Labs (the creators of dbt).

**HOW TO INSTALL:** Add to `packages.yml` and run `dbt deps`

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: ">=1.0.0"
```

### 9.1 unique_combination_of_columns

**WHAT IT DOES:** Checks that the COMBINATION of multiple columns is unique. The built-in `unique` test only works on ONE column. This test handles composite keys (two or more columns together).

**WHEN TO USE:** When a single column is NOT unique, but the combination IS. Example: An order can have many items, so `order_id` alone repeats. But (`order_id` + `item_id`) together should be unique — no duplicate line items.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT order_id, item_id, COUNT(*)
FROM order_items
GROUP BY order_id, item_id
HAVING COUNT(*) > 1;
```
Returns rows where the same pair appears more than once. If ANY rows return → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: order_items
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - order_id
            - item_id
```

**EXAMPLE DATA:**
```
┌──────────┬─────────┐        ┌──────────┬─────────┐
│ order_id │ item_id │        │ order_id │ item_id │
├──────────┼─────────┤        ├──────────┼─────────┤
│   1      │   A     │  PASS  │   1      │   A     │  FAIL
│   1      │   B     │  ────► │   1      │   A     │  ← duplicate!
│   2      │   A     │        │   2      │   A     │
└──────────┴─────────┘        └──────────┴─────────┘
```

### 9.2 expression_is_true

**WHAT IT DOES:** Validates that a custom SQL expression is TRUE for every row. Think of it as a "write any rule you want" test.

**WHEN TO USE:** When you need to check a business rule that involves comparing columns or running a calculation. No built-in test covers it.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT *
FROM orders
WHERE NOT (total_amount >= subtotal);
```
Finds rows where `total_amount < subtotal` (which should never happen). If ANY rows return → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: orders
    tests:
      - dbt_utils.expression_is_true:
          expression: "total_amount >= subtotal"
```

**EXAMPLE DATA:**
```
┌──────────┬──────────────┬──────────┐
│ order_id │ total_amount │ subtotal │  Result
├──────────┼──────────────┼──────────┤
│   1      │     150      │   120    │  PASS (150 >= 120 ✓)
│   2      │     200      │   200    │  PASS (200 >= 200 ✓)
│   3      │      80      │   100    │  FAIL (80 >= 100 ✗) ← bug!
└──────────┴──────────────┴──────────┘
```

### 9.3 recency

**WHAT IT DOES:** Checks that the most recent record is within a given time window. Ensures your data pipeline is still running and fresh.

**WHEN TO USE:** When you need to guarantee data freshness. Example: "We should have received an order in the last 3 days." If the newest `order_date` is 5 days ago → something is broken.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT 1
WHERE (
  SELECT MAX(order_date) FROM orders
) < DATEADD(day, -3, CURRENT_TIMESTAMP);
```
If the max date is older than 3 days ago → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: orders
    tests:
      - dbt_utils.recency:
          datepart: day
          field: order_date
          interval: 3
```

**EXAMPLE (today = 2026-04-17):**

| MAX(order_date) | Age | Result |
|---|---|---|
| 2026-04-16 | 1 day | PASS (within 3-day window) |
| 2026-04-10 | 7 days | FAIL (exceeds 3-day window) |

### 9.4 at_least_one

**WHAT IT DOES:** Checks that a column has AT LEAST ONE non-null value. It does NOT require all values to be non-null (that's `not_null`). It just ensures the column is not entirely empty.

**WHEN TO USE:** For optional columns that should have SOME data. Example: Not every order uses a `discount_code`, but if the entire column is NULL across all rows, something is wrong with the data load.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT 1
WHERE (
  SELECT COUNT(discount_code) FROM orders
) = 0;
```
COUNT ignores NULLs. If count = 0, every value is NULL → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: orders
    columns:
      - name: discount_code
        tests:
          - dbt_utils.at_least_one
```

**EXAMPLE DATA:**
```
┌──────────────┐     ┌──────────────┐
│ discount_code│     │ discount_code│
├──────────────┤     ├──────────────┤
│  SUMMER10    │     │  NULL        │
│  NULL        │     │  NULL        │  ALL NULL
│  NULL        │     │  NULL        │
└──────────────┘     └──────────────┘
    PASS ✓                FAIL ✗
```

### 9.5 not_constant

**WHAT IT DOES:** Checks that a column has MORE THAN ONE distinct value. If every row has the same value, the test fails.

**WHEN TO USE:** When a column SHOULD have variety. Example: A "category" column with 1000 products should NOT all say "Uncategorized". If it does, the ETL likely has a bug.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT 1
WHERE (
  SELECT COUNT(DISTINCT category) FROM products
) <= 1;
```
If only 0 or 1 distinct values exist → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: products
    columns:
      - name: category
        tests:
          - dbt_utils.not_constant
```

**EXAMPLE DATA:**
```
┌──────────────┐     ┌──────────────┐
│  category    │     │  category    │
├──────────────┤     ├──────────────┤
│  Electronics │     │  Electronics │
│  Clothing    │     │  Electronics │  ALL SAME
│  Books       │     │  Electronics │
└──────────────┘     └──────────────┘
    PASS ✓                FAIL ✗
```

### 9.6 equal_rowcount

**WHAT IT DOES:** Checks that two models have the EXACT same number of rows.

**WHEN TO USE:** When transforming data 1-to-1 (no filtering, no duplication). Example: `stg_orders` should have the same row count as `raw_orders` if staging only renames/casts columns without filtering. If counts differ → rows were accidentally dropped or duplicated.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT 1
WHERE (SELECT COUNT(*) FROM stg_orders)
   != (SELECT COUNT(*) FROM raw_orders);
```
If counts don't match → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: stg_orders
    tests:
      - dbt_utils.equal_rowcount:
          compare_model: ref('raw_orders')
```

**EXAMPLE:**

| stg_orders | raw_orders | Result |
|---|---|---|
| 10,000 rows | 10,000 rows | PASS |
| 9,998 rows | 10,000 rows | FAIL (2 rows lost!) |

---

## SECTION 10: dbt_expectations PACKAGE TESTS

**WHAT:** A comprehensive testing library inspired by Great Expectations (a popular Python data quality framework). Provides 50+ tests.

**HOW TO INSTALL:** Add to `packages.yml` and run `dbt deps`

```yaml
packages:
  - package: calogica/dbt_expectations
    version: ">=0.10.0"
```

### 10.1 expect_column_values_to_be_between

**WHAT IT DOES:** Ensures every value in a column falls within a min/max range.

**WHEN TO USE:** For numeric columns that have known valid boundaries. Example: Quantity ordered should be between 1 and 10,000. A value of -5 or 999,999 would indicate corrupt data.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT *
FROM order_items
WHERE quantity < 1 OR quantity > 10000;
```
Finds values outside the allowed range → **TEST FAILS** if any found

**YAML:**
```yaml
columns:
  - name: quantity
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 1
          max_value: 10000
```

**EXAMPLE DATA:**

| quantity | Result |
|---|---|
| 5 | PASS (within 1–10000) |
| 10000 | PASS (boundary, still valid) |
| 0 | FAIL (below min) |
| -3 | FAIL (below min) |

### 10.2 expect_column_values_to_match_regex

**WHAT IT DOES:** Validates that every value in a column matches a regular expression pattern.

**WHEN TO USE:** For columns with a known format. Example: Emails must look like "user@domain.com". Phone numbers must match a digit pattern, ZIP codes must be 5 digits, etc.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT *
FROM customers
WHERE NOT REGEXP_LIKE(email, '^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$');
```
Finds emails that don't match the pattern → **TEST FAILS** if any found

**YAML:**
```yaml
columns:
  - name: email
    tests:
      - dbt_expectations.expect_column_values_to_match_regex:
          regex: "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$"
```

**EXAMPLE DATA:**

| email | Result |
|---|---|
| john@gmail.com | PASS |
| jane.doe@co.uk | PASS |
| not-an-email | FAIL (no @ symbol) |
| @missing.com | FAIL (no local part) |

### 10.3 expect_table_row_count_to_be_between

**WHAT IT DOES:** Checks that the TOTAL row count of a table is within an expected range. Unlike per-row tests, this checks the table as a whole.

**WHEN TO USE:** When you know a table should have a reasonable number of rows. Example: `daily_revenue` should have between 1 and 100,000 rows. If it has 0 → data didn't load. If it has 10 million → duplicates or bad join.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT 1
WHERE (SELECT COUNT(*) FROM daily_revenue) < 1
   OR (SELECT COUNT(*) FROM daily_revenue) > 100000;
```
If count is outside range → **TEST FAILS**

**YAML:**
```yaml
models:
  - name: daily_revenue
    tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1
          max_value: 100000
```

**EXAMPLE:**

| Row count | Result |
|---|---|
| 500 | PASS |
| 0 | FAIL (table is empty!) |
| 500,000 | FAIL (way too many rows — possible duplication) |

### 10.4 expect_column_values_to_be_of_type

**WHAT IT DOES:** Validates that a column's DATA TYPE matches what you expect. Catches schema drift — when upstream changes break your column types.

**WHEN TO USE:** When a column MUST be a specific type for downstream logic. Example: "price" must be NUMBER, not VARCHAR. If someone changes the source and price becomes text, calculations break.

**HOW IT WORKS:** Inspects the column metadata (INFORMATION_SCHEMA) to verify the data type matches the expected type.

**YAML:**
```yaml
columns:
  - name: price
    tests:
      - dbt_expectations.expect_column_values_to_be_of_type:
          column_type: NUMBER
```

**EXAMPLE:**

| price column type | Result |
|---|---|
| NUMBER(10,2) | PASS (matches NUMBER) |
| VARCHAR(100) | FAIL (expected NUMBER, got VARCHAR) |

### 10.5 expect_column_distinct_count_to_be_greater_than

**WHAT IT DOES:** Checks that a column has MORE than N distinct values. Catches low-cardinality problems — when data lacks expected variety.

**WHEN TO USE:** When a column should have diversity. Example: A "country" column across 1 million customers should have more than 5 distinct countries. If only 1 country exists, the ETL might be filtering incorrectly.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT 1
WHERE (SELECT COUNT(DISTINCT country) FROM customers) <= 5;
```
If distinct count is 5 or fewer → **TEST FAILS**

**YAML:**
```yaml
columns:
  - name: country
    tests:
      - dbt_expectations.expect_column_distinct_count_to_be_greater_than:
          value: 5
```

**EXAMPLE:**

| Distinct countries | Result |
|---|---|
| 42 | PASS (42 > 5) |
| 3 | FAIL (3 is not > 5, data looks incomplete) |

### 10.6 expect_column_values_to_not_be_in_set

**WHAT IT DOES:** Ensures that a column NEVER contains specific forbidden values. The opposite of `accepted_values` — this is a "blocklist" test.

**WHEN TO USE:** When certain values should NEVER appear in your clean data. Example: After soft-deleting users, your active users table should never have status = 'DELETED' or 'BANNED'.

**HOW IT WORKS (compiled SQL):**
```sql
SELECT *
FROM users
WHERE status IN ('DELETED', 'BANNED');
```
Finds rows with forbidden values → **TEST FAILS** if any found

**YAML:**
```yaml
columns:
  - name: status
    tests:
      - dbt_expectations.expect_column_values_to_not_be_in_set:
          value_set: ['DELETED', 'BANNED']
```

**EXAMPLE DATA:**

| status | Result |
|---|---|
| active | PASS (not in blocklist) |
| inactive | PASS (not in blocklist) |
| DELETED | FAIL (forbidden value found!) |
| BANNED | FAIL (forbidden value found!) |

---

## SECTION 11: TEST SELECTION & COMMANDS

```bash
# Run ALL tests
dbt test

# Run tests for a specific model
dbt test --select orders

# Run tests for a model and its upstream dependencies
dbt test --select +orders

# Run only generic tests
dbt test --select test_type:generic

# Run only singular tests
dbt test --select test_type:singular

# Run tests + models together
dbt build

# Run tests with a specific tag
dbt test --select tag:critical

# Store test failures in a table for analysis
dbt test --store-failures
```

---

## SECTION 12: STORE FAILURES

When `--store-failures` is used (or configured in `dbt_project.yml`), dbt writes failing rows to a schema (default: `<target_schema>_dbt_test__audit`). This is invaluable for debugging which rows failed.

**dbt_project.yml config:**
```yaml
tests:
  +store_failures: true
  +schema: dbt_test__audit
```

---

## SECTION 13: TEST TAGS & ORGANIZATION

Tag tests to organize and run them selectively.

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - not_null:
              tags: ['critical', 'daily']
          - unique:
              tags: ['critical', 'daily']
      - name: discount_code
        tests:
          - accepted_values:
              values: ['SUMMER10', 'WINTER20', 'VIP50']
              tags: ['non-critical', 'weekly']
```

---

## SECTION 13A: DEEP DIVE — freshness, tags, and ref() EXPLAINED

You'll see these three concepts used throughout this guide. Here's what each one means, why it exists, and where it appears.

### 13A.1 freshness (source freshness check)

**WHAT IS IT?**

`freshness` is a SOURCE-LEVEL config that checks how RECENTLY data was loaded into a raw table. It answers: "Is my source data stale?"

**HOW IT WORKS:**

dbt looks at the `loaded_at_field` column (a timestamp column that records when each row was loaded) and checks:

`MAX(loaded_at_field)` vs `CURRENT_TIMESTAMP`

```yaml
freshness:
  warn_after: {count: 12, period: hour}     # WARN if data is >12h old
  error_after: {count: 24, period: hour}    # ERROR if data is >24h old
loaded_at_field: _loaded_at                 # the timestamp column to check
```

**TIMELINE EXAMPLE (now = 2026-04-17 10:00 AM):**

| MAX(_loaded_at) | Age | Result |
|---|---|---|
| 2026-04-17 08:00 AM | 2 hours | PASS (fresh!) |
| 2026-04-16 08:00 PM | 14 hours | WARN (>12h, <24h) |
| 2026-04-16 06:00 AM | 28 hours | ERROR (>24h, stale!) |

**HOW TO RUN IT:**
```bash
dbt source freshness
```

**REAL-WORLD ANALOGY:**
Think of a newspaper delivery. You expect today's paper by 7 AM. If it arrives by 7 AM → fresh. If it's 7 PM and no paper → warn. If it's the next day and still no paper → error, something is broken.

### 13A.2 tags (test organization labels)

**WHAT IS IT?**

Tags are LABELS you attach to tests (or models) so you can RUN them selectively. They don't change what the test does — they organize tests.

**WHY USE TAGS?**

In a large project you might have 500+ tests. You don't always want to run ALL of them. Tags let you group tests by importance or schedule:

| Tag | When to run |
|---|---|
| `'critical'` | Every dbt run (must never fail) |
| `'daily'` | Once per day in scheduled job |
| `'weekly'` | Once per week (less urgent tests) |
| `'non-critical'` | Low priority, run occasionally |
| `'freshness'` | Data freshness checks only |

**HOW TO RUN TAGGED TESTS:**
```bash
dbt test --select tag:critical          # run only critical tests
dbt test --select tag:daily             # run only daily tests
dbt test --exclude tag:non-critical     # run everything EXCEPT non-critical
```

**YAML SYNTAX:**
```yaml
tests:
  - not_null:
      tags: ['critical', 'daily']       # this test has 2 tags
  - unique:
      tags: ['critical']                # this test has 1 tag
```

### 13A.3 ref() and source() — dbt's reference functions

**WHAT IS ref()?**

`{{ ref('model_name') }}` is dbt's way of referencing ANOTHER dbt model. It does TWO things:
1. Resolves the model name to the actual `database.schema.table`
2. Tells dbt about the DEPENDENCY (so it runs models in correct order)

**EXAMPLE:**
```
{{ ref('stg_orders') }}
Compiles to → RAW_DB.DEV_SCHEMA.STG_ORDERS (or whatever your target is)
```

**WHAT IS source()?**

`{{ source('source_name', 'table_name') }}` references a RAW table defined in your `sources.yml`. It's for tables NOT built by dbt.

**EXAMPLE:**
```
{{ source('ecommerce_raw', 'raw_orders') }}
Compiles to → RAW_DB.PUBLIC.RAW_ORDERS
```

**WHY NOT JUST WRITE THE TABLE NAME DIRECTLY?**

| Hardcoded (BAD) | Using ref() (GOOD) |
|---|---|
| `FROM raw_db.public.orders` | `FROM {{ ref('stg_orders') }}` |
| Breaks if schema changes | Auto-resolves to correct schema |
| No dependency tracking | dbt knows the execution order |
| Can't switch environments | Works in dev, staging, AND prod |

**VISUAL: How ref() builds the dependency chain (DAG):**

```
source('ecommerce_raw','raw_orders')
       │
       ▼
  stg_orders  ──────────┐
       │                │
       ▼                ▼
int_order_payments    stg_customers
       │                │
       └────┬───────────┘
            ▼
       fct_orders
```

dbt uses `ref()` to build this DAG automatically. It runs models top-to-bottom so dependencies are always ready.

---

## SECTION 14: REAL-LIFE EXAMPLE — E-COMMERCE DATA PIPELINE

**Scenario:** You run an e-commerce business. Raw data lands in Snowflake from your app database. You build dbt models and need tests at every layer.

**Pipeline:** `RAW → STAGING → INTERMEDIATE → MARTS`

### STEP 1: Define Sources & Test Raw Data

**File: `models/staging/sources.yml`**

```yaml
version: 2
sources:
  - name: ecommerce_raw
    database: RAW_DB
    schema: PUBLIC
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _loaded_at
    tables:
      - name: raw_orders
        columns:
          - name: order_id
            tests:
              - not_null
              - unique
          - name: customer_id
            tests:
              - not_null
          - name: order_date
            tests:
              - not_null

      - name: raw_customers
        columns:
          - name: customer_id
            tests:
              - not_null
              - unique
          - name: email
            tests:
              - not_null
              - unique

      - name: raw_payments
        columns:
          - name: payment_id
            tests:
              - not_null
              - unique
          - name: order_id
            tests:
              - not_null
```

### STEP 2: Staging Models with Tests

**File: `models/staging/stg_orders.sql`**

```sql
SELECT
    order_id,
    customer_id,
    order_date,
    status,
    total_amount
FROM {{ source('ecommerce_raw', 'raw_orders') }}
WHERE order_id IS NOT NULL
```

**File: `models/staging/schema.yml`**

```yaml
version: 2
models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: status
        tests:
          - accepted_values:
              values: ['pending', 'processing', 'shipped', 'delivered', 'cancelled', 'returned']
      - name: total_amount
        tests:
          - is_positive
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id

  - name: stg_customers
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
      - name: email
        tests:
          - not_null
          - unique
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$"

  - name: stg_payments
    columns:
      - name: payment_id
        tests:
          - not_null
          - unique
      - name: amount
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 999999
      - name: payment_method
        tests:
          - accepted_values:
              values: ['credit_card', 'debit_card', 'paypal', 'bank_transfer', 'gift_card']
      - name: order_id
        tests:
          - relationships:
              to: ref('stg_orders')
              field: order_id
```

### STEP 3: Intermediate Model with Tests

**File: `models/intermediate/int_order_payments.sql`**

```sql
SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    o.total_amount AS order_amount,
    COALESCE(SUM(p.amount), 0) AS payment_total,
    COUNT(p.payment_id) AS payment_count
FROM {{ ref('stg_orders') }} o
LEFT JOIN {{ ref('stg_payments') }} p ON o.order_id = p.order_id
GROUP BY o.order_id, o.customer_id, o.order_date, o.status, o.total_amount
```

**File: `models/intermediate/schema.yml`**

```yaml
version: 2
models:
  - name: int_order_payments
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - order_id
      - dbt_utils.expression_is_true:
          expression: "payment_total >= 0"
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: payment_count
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 50
```

### STEP 4: Mart Model with Tests

**File: `models/marts/fct_orders.sql`**

```sql
SELECT
    op.order_id,
    op.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    op.order_date,
    op.status,
    op.order_amount,
    op.payment_total,
    op.payment_count,
    op.order_amount - op.payment_total AS outstanding_balance
FROM {{ ref('int_order_payments') }} op
JOIN {{ ref('stg_customers') }} c ON op.customer_id = c.customer_id
```

**File: `models/marts/schema.yml`**

```yaml
version: 2
models:
  - name: fct_orders
    tests:
      - dbt_utils.recency:
          datepart: day
          field: order_date
          interval: 7
          tags: ['freshness']
      - dbt_utils.expression_is_true:
          expression: "outstanding_balance = order_amount - payment_total"
          tags: ['critical']
    columns:
      - name: order_id
        tests:
          - not_null:
              tags: ['critical']
          - unique:
              tags: ['critical']
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
      - name: order_amount
        tests:
          - is_positive
      - name: outstanding_balance
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: -1000
              max_value: 999999
              config:
                severity: warn
```

### STEP 5: Singular Tests for Complex Business Rules

**File: `tests/assert_no_orphan_payments.sql`**

```sql
SELECT p.payment_id, p.order_id
FROM {{ ref('stg_payments') }} p
LEFT JOIN {{ ref('stg_orders') }} o ON p.order_id = o.order_id
WHERE o.order_id IS NULL
```

**File: `tests/assert_delivered_orders_have_payments.sql`**

```sql
SELECT o.order_id
FROM {{ ref('fct_orders') }} o
WHERE o.status = 'delivered'
  AND o.payment_count = 0
```

**File: `tests/assert_revenue_is_consistent.sql`**

```sql
SELECT 1
FROM (
    SELECT
        SUM(order_amount) AS total_orders,
        SUM(payment_total) AS total_payments
    FROM {{ ref('fct_orders') }}
    WHERE status NOT IN ('cancelled', 'returned')
) sub
WHERE ABS(total_orders - total_payments) > 1000
```

### STEP 6: Custom Generic Tests (Reusable)

**File: `macros/test_is_positive.sql`**

```sql
{% test is_positive(model, column_name) %}

SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

**File: `macros/test_no_future_dates.sql`**

```sql
{% test no_future_dates(model, column_name) %}

SELECT *
FROM {{ model }}
WHERE {{ column_name }} > CURRENT_DATE()

{% endtest %}
```

**Usage:**
```yaml
columns:
  - name: order_date
    tests:
      - no_future_dates
```

---

## SECTION 15: QUICK REFERENCE CHEAT SHEET

### Test Types & Where to Define Them

| TEST TYPE | WHERE TO DEFINE |
|---|---|
| `not_null` | YAML (schema.yml) — built-in generic |
| `unique` | YAML (schema.yml) — built-in generic |
| `accepted_values` | YAML (schema.yml) — built-in generic |
| `relationships` | YAML (schema.yml) — built-in generic |
| Custom generic test | `macros/` folder + YAML reference |
| Singular test | `tests/` folder as standalone .sql files |
| dbt_utils tests | YAML — install package first |
| dbt_expectations tests | YAML — install package first |
| Source tests | YAML (sources.yml) — same syntax as models |

### Commands

| COMMAND | WHAT IT DOES |
|---|---|
| `dbt test` | Run all tests |
| `dbt test --select X` | Run tests for model X |
| `dbt test --select +X` | Run tests for X and its upstream deps |
| `dbt build` | Run models + tests together |
| `dbt test --store-failures` | Save failing rows to audit schema |
| `dbt test -s tag:critical` | Run tests tagged 'critical' |
| `dbt test -s test_type:generic` | Run only generic tests |
| `dbt test -s test_type:singular` | Run only singular tests |

### KEY PRINCIPLE

Test early, test often, test at every layer.

| Layer | What to Test |
|---|---|
| **RAW** | Source data integrity |
| **STAGING** | Cleaning & type casting |
| **INTERMEDIATE** | Joins & aggregations |
| **MARTS** | Business logic & freshness |
