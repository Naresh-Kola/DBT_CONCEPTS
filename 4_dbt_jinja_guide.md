# Jinja in dbt - Complete Guide with Simple Examples

---

## Section 1: What is Jinja?

**Jinja** is a templating language that lets you write **PROGRAMMING LOGIC** inside your SQL files. It makes your SQL dynamic and reusable.

### Simple Analogy
- Normal SQL = a printed letter (same content every time)
- SQL with Jinja = a mail merge template (changes based on inputs)

Think of it like this:
- SQL alone: `SELECT * FROM orders WHERE status = 'active'`
- SQL + Jinja: `SELECT * FROM orders WHERE status = '{{ variable }}'`
  - The `{{ variable }}` gets REPLACED with a real value before SQL runs.

### Why Do We Need Jinja in dbt?
- Write SQL once, reuse it for many tables
- Generate repetitive SQL automatically (loops)
- Add if/else logic (different SQL for dev vs prod)
- Create functions (macros) that generate SQL
- Reference other models dynamically

### Jinja Syntax — ONLY 3 Things to Remember

| Syntax | Purpose | Example |
|--------|---------|---------|
| `{{ ... }}` | OUTPUT something (print a value) | `{{ ref('stg_orders') }}` |
| `{% ... %}` | DO something (logic: if, for, set) | `{% if target.name == 'dev' %}` |
| `{# ... #}` | COMMENT (ignored, not executed) | `{# TODO: fix later #}` |

That's it. Everything in Jinja uses one of these three.

---

## Section 2: `{{ }}` — Expressions (Output/Print Values)

`{{ }}` = "Put this value here". It REPLACES itself with whatever is inside.

### Example 1: Simple variable output

**Jinja:**
```sql
{% set my_database = 'ANALYTICS' %}
SELECT * FROM {{ my_database }}.PUBLIC.ORDERS;
```

**Compiles to:**
```sql
SELECT * FROM ANALYTICS.PUBLIC.ORDERS;
```

### Example 2: ref() is actually Jinja!

**Jinja:**
```sql
SELECT * FROM {{ ref('stg_orders') }};
```

**Compiles to:**
```sql
SELECT * FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

### Example 3: source() is also Jinja!

**Jinja:**
```sql
SELECT * FROM {{ source('raw', 'customers') }};
```

**Compiles to:**
```sql
SELECT * FROM DBT_DEV_DB.DEV_RAW.CUSTOMERS;
```

### Example 4: Math inside `{{ }}`

**Jinja:**
```sql
{% set tax_rate = 0.18 %}
SELECT
    ORDER_ID,
    AMOUNT,
    AMOUNT * {{ tax_rate }} AS TAX,
    AMOUNT * (1 + {{ tax_rate }}) AS TOTAL_WITH_TAX
FROM {{ ref('stg_orders') }};
```

**Compiles to:**
```sql
SELECT
    ORDER_ID,
    AMOUNT,
    AMOUNT * 0.18 AS TAX,
    AMOUNT * (1 + 0.18) AS TOTAL_WITH_TAX
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

### Example 5: String concatenation

**Jinja:**
```sql
{% set schema_name = 'STAGING' %}
{% set table_name = 'ORDERS' %}
SELECT * FROM ANALYTICS.{{ schema_name }}.{{ table_name }};
```

**Compiles to:**
```sql
SELECT * FROM ANALYTICS.STAGING.ORDERS;
```

---

## Section 3: `{% %}` — Statements (Do Logic)

`{% %}` = "Do this logic but don't print anything"

Used for: setting variables, if/else, loops, macros

### 3.1 SET — Create Variables

**Jinja:**
```sql
{% set payment_methods = ['credit_card', 'debit_card', 'upi', 'cash'] %}
{% set discount_pct = 10 %}

SELECT
    ORDER_ID,
    AMOUNT,
    AMOUNT * {{ discount_pct }} / 100 AS DISCOUNT
FROM {{ ref('stg_orders') }};
```

**Compiles to:**
```sql
SELECT
    ORDER_ID,
    AMOUNT,
    AMOUNT * 10 / 100 AS DISCOUNT
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

### 3.2 IF / ELIF / ELSE — Conditional Logic

#### Example 1: Different filter based on environment

**Jinja:**
```sql
SELECT
    ORDER_ID,
    CUSTOMER_ID,
    ORDER_DATE,
    AMOUNT
FROM {{ ref('stg_orders') }}
{% if target.name == 'dev' %}
WHERE ORDER_DATE >= DATEADD('DAY', -30, CURRENT_DATE())
{% elif target.name == 'staging' %}
WHERE ORDER_DATE >= DATEADD('DAY', -90, CURRENT_DATE())
{% else %}
-- prod: no filter, all data
{% endif %}
```

**In DEV compiles to:**
```sql
SELECT
    ORDER_ID,
    CUSTOMER_ID,
    ORDER_DATE,
    AMOUNT
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS
WHERE ORDER_DATE >= DATEADD('DAY', -30, CURRENT_DATE());
```

**In PROD compiles to:**
```sql
SELECT
    ORDER_ID,
    CUSTOMER_ID,
    ORDER_DATE,
    AMOUNT
FROM PROD_DB.STAGING.STG_ORDERS;
```

#### Example 2: is_incremental() — the most common if statement in dbt

**Jinja:**
```sql
{{ config(materialized='incremental', unique_key='ORDER_ID') }}

SELECT * FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
    WHERE UPDATED_AT > (SELECT MAX(UPDATED_AT) FROM {{ this }})
{% endif %}
```

**First run (table doesn't exist) compiles to:**
```sql
SELECT * FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

**Subsequent runs (table exists) compiles to:**
```sql
SELECT * FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS
WHERE UPDATED_AT > (SELECT MAX(UPDATED_AT) FROM DBT_DEV_DB.DEV_MARTS.FCT_ORDERS);
```

#### Example 3: Conditionally add a column

**Jinja:**
```sql
SELECT
    ORDER_ID,
    AMOUNT,
    {% if var('include_tax', false) %}
    AMOUNT * 0.18 AS TAX_AMOUNT,
    {% endif %}
    STATUS
FROM {{ ref('stg_orders') }};
```

**With:** `dbt run --vars '{include_tax: true}'` compiles to:
```sql
SELECT
    ORDER_ID,
    AMOUNT,
    AMOUNT * 0.18 AS TAX_AMOUNT,
    STATUS
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

**Without variable (default false) compiles to:**
```sql
SELECT
    ORDER_ID,
    AMOUNT,
    STATUS
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

### 3.3 FOR LOOPS — Generate Repetitive SQL

#### Example 1: Generate SUM for each payment method

**Jinja:**
```sql
{% set payment_methods = ['credit_card', 'debit_card', 'upi', 'cash'] %}

SELECT
    ORDER_DATE,
    {% for method in payment_methods %}
    SUM(CASE WHEN PAYMENT_METHOD = '{{ method }}' THEN AMOUNT ELSE 0 END) AS {{ method }}_total
    {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_payments') }}
GROUP BY ORDER_DATE
```

**Compiles to:**
```sql
SELECT
    ORDER_DATE,
    SUM(CASE WHEN PAYMENT_METHOD = 'credit_card' THEN AMOUNT ELSE 0 END) AS credit_card_total,
    SUM(CASE WHEN PAYMENT_METHOD = 'debit_card' THEN AMOUNT ELSE 0 END) AS debit_card_total,
    SUM(CASE WHEN PAYMENT_METHOD = 'upi' THEN AMOUNT ELSE 0 END) AS upi_total,
    SUM(CASE WHEN PAYMENT_METHOD = 'cash' THEN AMOUNT ELSE 0 END) AS cash_total
FROM DBT_DEV_DB.DEV_STAGING.STG_PAYMENTS
GROUP BY ORDER_DATE;
```

#### Example 2: UNION ALL multiple tables using a loop

**Jinja:**
```sql
{% set years = [2022, 2023, 2024] %}

{% for year in years %}
SELECT * FROM RAW_DB.PUBLIC.ORDERS_{{ year }}
{% if not loop.last %}UNION ALL{% endif %}
{% endfor %}
```

**Compiles to:**
```sql
SELECT * FROM RAW_DB.PUBLIC.ORDERS_2022
UNION ALL
SELECT * FROM RAW_DB.PUBLIC.ORDERS_2023
UNION ALL
SELECT * FROM RAW_DB.PUBLIC.ORDERS_2024;
```

#### Example 3: Generate multiple columns dynamically

**Jinja:**
```sql
{% set status_list = ['pending', 'shipped', 'delivered', 'cancelled'] %}

SELECT
    CUSTOMER_ID,
    {% for status in status_list %}
    COUNT(CASE WHEN STATUS = '{{ status }}' THEN 1 END) AS {{ status }}_orders
    {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_orders') }}
GROUP BY CUSTOMER_ID
```

**Compiles to:**
```sql
SELECT
    CUSTOMER_ID,
    COUNT(CASE WHEN STATUS = 'pending' THEN 1 END) AS pending_orders,
    COUNT(CASE WHEN STATUS = 'shipped' THEN 1 END) AS shipped_orders,
    COUNT(CASE WHEN STATUS = 'delivered' THEN 1 END) AS delivered_orders,
    COUNT(CASE WHEN STATUS = 'cancelled' THEN 1 END) AS cancelled_orders
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS
GROUP BY CUSTOMER_ID;
```

#### Example 4: Loop with index (loop.index)

**Jinja:**
```sql
{% set columns = ['NAME', 'EMAIL', 'PHONE', 'ADDRESS'] %}

SELECT
    CUSTOMER_ID,
    {% for col in columns %}
    {{ col }} AS FIELD_{{ loop.index }}
    {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_customers') }}
```

**Compiles to:**
```sql
SELECT
    CUSTOMER_ID,
    NAME AS FIELD_1,
    EMAIL AS FIELD_2,
    PHONE AS FIELD_3,
    ADDRESS AS FIELD_4
FROM DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS;
```

### Loop Special Variables

| Variable | What it gives you |
|----------|-------------------|
| `loop.index` | Current iteration number (1,2,3) |
| `loop.index0` | Zero-based index (0,1,2) |
| `loop.first` | True if first iteration |
| `loop.last` | True if last iteration |
| `loop.length` | Total number of iterations |

---

## Section 4: `{# #}` — Comments

`{# #}` = Jinja comments. They are **REMOVED** during compilation. They do NOT appear in the final SQL sent to Snowflake.

**Jinja:**
```sql
{# This comment will NOT appear in compiled SQL #}

SELECT
    ORDER_ID,
    {# TODO: add discount calculation later #}
    AMOUNT
FROM {{ ref('stg_orders') }};
```

**Compiles to (comments gone):**
```sql
SELECT
    ORDER_ID,
    AMOUNT
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

> **Note:** SQL comments (`--` or `/* */`) DO appear in compiled SQL. Jinja comments (`{# #}`) do NOT.

---

## Section 5: Macros — Reusable Jinja Functions

A **macro** = a Jinja function that generates SQL. Define it once in `macros/` folder, use it in any model.

### 5.1 Simple Macro (no parameters)

**File: `macros/current_timestamp_ist.sql`**
```sql
{% macro current_timestamp_ist() %}
    CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', CURRENT_TIMESTAMP())
{% endmacro %}
```

**Usage in a model:**
```sql
SELECT
    ORDER_ID,
    {{ current_timestamp_ist() }} AS LOADED_AT_IST
FROM {{ ref('stg_orders') }};
```

**Compiles to:**
```sql
SELECT
    ORDER_ID,
    CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', CURRENT_TIMESTAMP()) AS LOADED_AT_IST
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

### 5.2 Macro with Parameters

**File: `macros/cents_to_dollars.sql`**
```sql
{% macro cents_to_dollars(column_name, decimal_places=2) %}
    ROUND({{ column_name }} / 100.0, {{ decimal_places }})
{% endmacro %}
```

**Usage:**
```sql
SELECT
    ORDER_ID,
    {{ cents_to_dollars('AMOUNT_CENTS') }} AS AMOUNT_DOLLARS,
    {{ cents_to_dollars('TAX_CENTS', 4) }} AS TAX_DOLLARS
FROM {{ ref('stg_orders') }};
```

**Compiles to:**
```sql
SELECT
    ORDER_ID,
    ROUND(AMOUNT_CENTS / 100.0, 2) AS AMOUNT_DOLLARS,
    ROUND(TAX_CENTS / 100.0, 4) AS TAX_DOLLARS
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

### 5.3 Macro that generates a CASE statement

**File: `macros/status_label.sql`**
```sql
{% macro status_label(column_name) %}
    CASE {{ column_name }}
        WHEN 1 THEN 'Active'
        WHEN 2 THEN 'Inactive'
        WHEN 3 THEN 'Suspended'
        ELSE 'Unknown'
    END
{% endmacro %}
```

**Usage:**
```sql
SELECT
    CUSTOMER_ID,
    STATUS_CODE,
    {{ status_label('STATUS_CODE') }} AS STATUS_NAME
FROM {{ ref('stg_customers') }};
```

**Compiles to:**
```sql
SELECT
    CUSTOMER_ID,
    STATUS_CODE,
    CASE STATUS_CODE
        WHEN 1 THEN 'Active'
        WHEN 2 THEN 'Inactive'
        WHEN 3 THEN 'Suspended'
        ELSE 'Unknown'
    END AS STATUS_NAME
FROM DBT_DEV_DB.DEV_STAGING.STG_CUSTOMERS;
```

### 5.4 Macro with a Loop inside

**File: `macros/pivot_column.sql`**
```sql
{% macro pivot_column(column_name, values_list, agg='SUM', agg_column='AMOUNT') %}
    {% for val in values_list %}
    {{ agg }}(CASE WHEN {{ column_name }} = '{{ val }}' THEN {{ agg_column }} END) AS {{ val }}_{{ agg_column | lower }}
    {% if not loop.last %},{% endif %}
    {% endfor %}
{% endmacro %}
```

**Usage:**
```sql
SELECT
    CUSTOMER_ID,
    {{ pivot_column('STATUS', ['pending', 'shipped', 'delivered'], 'COUNT', '1') }}
FROM {{ ref('stg_orders') }}
GROUP BY CUSTOMER_ID
```

**Compiles to:**
```sql
SELECT
    CUSTOMER_ID,
    COUNT(CASE WHEN STATUS = 'pending' THEN 1 END) AS pending_1,
    COUNT(CASE WHEN STATUS = 'shipped' THEN 1 END) AS shipped_1,
    COUNT(CASE WHEN STATUS = 'delivered' THEN 1 END) AS delivered_1
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS
GROUP BY CUSTOMER_ID;
```

---

## Section 6: Built-in Jinja Variables in dbt

dbt gives you these variables automatically (no need to define them):

| Variable | What it contains | Example value |
|----------|-----------------|---------------|
| `target.name` | Current environment name | `'dev'`, `'prod'` |
| `target.database` | Target database | `'DBT_DEV_DB'` |
| `target.schema` | Target schema | `'DEV_STAGING'` |
| `target.type` | Warehouse type | `'snowflake'` |
| `this` | Current model's full table name | `'DB.SCHEMA.TABLE'` |
| `model.name` | Current model's name | `'stg_orders'` |
| `var('key')` | User-defined variable (from CLI or yml) | `dbt run --vars '{}'` |
| `env_var('KEY')` | Environment variable (NOT in Snowflake) | OS-level variable |
| `modules.datetime` | Python datetime module | For date calculations |
| `is_incremental()` | True if model exists and is incremental | True/False |

### Example: Using target variables

**Jinja:**
```sql
SELECT
    '{{ target.name }}' AS ENVIRONMENT,
    '{{ target.database }}' AS DATABASE_NAME,
    '{{ model.name }}' AS MODEL_NAME,
    *
FROM {{ ref('stg_orders') }}
{% if target.name == 'dev' %}
LIMIT 1000
{% endif %}
```

**In DEV compiles to:**
```sql
SELECT
    'dev' AS ENVIRONMENT,
    'DBT_DEV_DB' AS DATABASE_NAME,
    'stg_orders' AS MODEL_NAME,
    *
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS
LIMIT 1000;
```

**In PROD compiles to:**
```sql
SELECT
    'prod' AS ENVIRONMENT,
    'PROD_DB' AS DATABASE_NAME,
    'stg_orders' AS MODEL_NAME,
    *
FROM PROD_DB.STAGING.STG_ORDERS;
```

---

## Section 7: Filters (Jinja String Operations)

Filters modify values using the pipe (`|`) symbol. Think of them as string/value transformations.

| Filter | What it does | Example | Result |
|--------|-------------|---------|--------|
| `upper` | Uppercase | `{{ 'hello' \| upper }}` | HELLO |
| `lower` | Lowercase | `{{ 'HELLO' \| lower }}` | hello |
| `trim` | Remove whitespace | `{{ '  hi  ' \| trim }}` | hi |
| `replace` | Replace text | `{{ 'foo_bar' \| replace('_','-') }}` | foo-bar |
| `default` | Fallback if undefined | `{{ var \| default('N/A') }}` | N/A |
| `length` | Count items | `{{ [1,2,3] \| length }}` | 3 |
| `join` | Join list into string | `{{ ['a','b'] \| join(', ') }}` | a, b |

### Example: Using filters

**Jinja:**
```sql
{% set columns = ['order_id', 'customer_id', 'amount'] %}

SELECT
    {% for col in columns %}
    {{ col | upper }} AS {{ col | upper }}
    {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_orders') }}
```

**Compiles to:**
```sql
SELECT
    ORDER_ID AS ORDER_ID,
    CUSTOMER_ID AS CUSTOMER_ID,
    AMOUNT AS AMOUNT
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS;
```

---

## Section 8: Real-World Examples in dbt

### Example 1: Dynamic Schema Based on Environment

**Jinja:**
```sql
{% set source_schema = 'RAW_PROD' if target.name == 'prod' else 'RAW_DEV' %}

SELECT * FROM DBT_DEV_DB.{{ source_schema }}.ORDERS;
```

**In DEV:** `SELECT * FROM DBT_DEV_DB.RAW_DEV.ORDERS;`

**In PROD:** `SELECT * FROM DBT_DEV_DB.RAW_PROD.ORDERS;`

### Example 2: Generate Staging Models for Multiple Tables

**File: `macros/generate_staging.sql`**
```sql
{% macro generate_staging(source_name, table_name, columns) %}

SELECT
    {% for col in columns %}
    {{ col }}
    {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ source(source_name, table_name) }}
WHERE {{ columns[0] }} IS NOT NULL

{% endmacro %}
```

**Usage in `models/staging/stg_orders.sql`:**
```sql
{{ generate_staging('raw', 'orders', ['ORDER_ID', 'CUSTOMER_ID', 'AMOUNT', 'ORDER_DATE']) }}
```

**Compiles to:**
```sql
SELECT
    ORDER_ID,
    CUSTOMER_ID,
    AMOUNT,
    ORDER_DATE
FROM DBT_DEV_DB.DEV_RAW.ORDERS
WHERE ORDER_ID IS NOT NULL;
```

### Example 3: Configurable Incremental with Variable Lookback

**Jinja:**
```sql
{{ config(materialized='incremental', unique_key='ORDER_ID') }}

{% set lookback_days = var('lookback_days', 3) %}

SELECT
    ORDER_ID,
    CUSTOMER_ID,
    AMOUNT,
    ORDER_DATE,
    UPDATED_AT
FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE UPDATED_AT >= DATEADD('DAY', -{{ lookback_days }}, (SELECT MAX(UPDATED_AT) FROM {{ this }}))
{% endif %}
```

**Compiles to (incremental run):**
```sql
SELECT
    ORDER_ID,
    CUSTOMER_ID,
    AMOUNT,
    ORDER_DATE,
    UPDATED_AT
FROM DBT_DEV_DB.DEV_STAGING.STG_ORDERS
WHERE UPDATED_AT >= DATEADD('DAY', -3, (SELECT MAX(UPDATED_AT) FROM DBT_DEV_DB.DEV_MARTS.FCT_ORDERS));
```

With: `dbt run --vars '{lookback_days: 7}'` → changes -3 to -7

### Example 4: Grant Permissions Dynamically

**File: `macros/grant_select.sql`**
```sql
{% macro grant_select(roles) %}
    {% for role in roles %}
    GRANT SELECT ON {{ this }} TO ROLE {{ role }};
    {% endfor %}
{% endmacro %}
```

**Usage as post-hook:**
```sql
{{ config(
    materialized='table',
    post_hook="{{ grant_select(['ANALYST_ROLE', 'REPORTING_ROLE']) }}"
) }}

SELECT * FROM {{ ref('stg_orders') }};
```

**After table is built, runs:**
```sql
GRANT SELECT ON DBT_DEV_DB.DEV_MARTS.FCT_ORDERS TO ROLE ANALYST_ROLE;
GRANT SELECT ON DBT_DEV_DB.DEV_MARTS.FCT_ORDERS TO ROLE REPORTING_ROLE;
```

---

## Section 9: Whitespace Control

**Problem:** Jinja adds blank lines where `{% %}` blocks were.

**Solution:** Add a hyphen (`-`) to trim whitespace.

**Without whitespace control:**
```
{% for i in [1,2,3] %}
  {{ i }}
{% endfor %}
```
Output has extra blank lines between values.

**With whitespace control:**
```
{%- for i in [1,2,3] -%}
  {{ i }}
{%- endfor -%}
```
Output is compact: `1  2  3`

### Rules:
| Syntax | Effect |
|--------|--------|
| `{%-` | Trim whitespace BEFORE |
| `-%}` | Trim whitespace AFTER |
| `{{-` | Trim whitespace BEFORE output |
| `-}}` | Trim whitespace AFTER output |

---

## Section 10: Common Patterns Cheat Sheet

| What you want | Jinja pattern |
|---------------|---------------|
| Reference another model | `{{ ref('model_name') }}` |
| Reference a source table | `{{ source('source', 'table') }}` |
| Set a variable | `{% set x = 'value' %}` |
| If/else condition | `{% if condition %} ... {% else %} ... {% endif %}` |
| Loop through a list | `{% for item in list %} ... {% endfor %}` |
| Handle last comma in loop | `{% if not loop.last %},{% endif %}` |
| Call a macro | `{{ macro_name(arg1, arg2) }}` |
| User-defined variable | `{{ var('key', 'default') }}` |
| Current environment | `{{ target.name }}` |
| Current model table reference | `{{ this }}` |
| Check if incremental run | `{% if is_incremental() %} ... {% endif %}` |
| Compile but don't execute | `dbt compile --select model_name` |
| Uppercase a value | `{{ value \| upper }}` |
| Default if undefined | `{{ value \| default('fallback') }}` |
| Run SQL and capture result | `{% set results = run_query(sql) %}` |
