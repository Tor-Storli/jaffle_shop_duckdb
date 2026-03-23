# dbt Concepts — Code Snippets & Examples

### Using the Jaffle Shop DuckDB Dataset

> All examples in this guide are written specifically for the `jaffle_shop_duckdb` project. Every code snippet is ready to drop into your project files. Each section explains the **what**, **why**, and **how** of each dbt concept with real Jaffle Shop tables and columns.

---

## Table of Contents

1. [Package Setup — packages.yml](#1-package-setup--packagesyml)
2. [Sources — Declaring Raw Data](#2-sources--declaring-raw-data)
3. [Tests — Built-in Generic Tests](#3-tests--built-in-generic-tests)
4. [Tests — Singular (Custom SQL) Tests](#4-tests--singular-custom-sql-tests)
5. [Jinja — Model References & Templating](#5-jinja--model-references--templating)
6. [Jinja — Variables & var()](#6-jinja--variables--var)
7. [Macros — Writing & Using Your Own](#7-macros--writing--using-your-own)
8. [Documentation — Descriptions & Doc Blocks](#8-documentation--descriptions--doc-blocks)
9. [Snapshots — Tracking Historical Changes](#9-snapshots--tracking-historical-changes)
10. [Incremental Models](#10-incremental-models)
11. [Data Contracts — Schema Enforcement](#11-data-contracts--schema-enforcement)
12. [dbt-utils — Utility Macros & Tests](#12-dbt-utils--utility-macros--tests)
13. [dbt-expectations — Advanced Data Quality Tests](#13-dbt-expectations--advanced-data-quality-tests)
14. [Putting It All Together — The Full Pipeline](#14-putting-it-all-together--the-full-pipeline)

---

## 1. Package Setup — `packages.yml`

Before using `dbt-utils` or `dbt-expectations`, declare them as dependencies in `packages.yml` at the root of your project. After editing this file, always run `dbt deps` to install.

**File:** `packages.yml`

```yaml
packages:
  # dbt-utils: utility macros, SQL generators, and generic tests from dbt Labs
  - package: dbt-labs/dbt_utils
    version: 1.3.0

  # dbt-expectations: Great Expectations-inspired advanced data quality tests
  # Note: as of late 2024, maintained by Metaplane as a community fork
  - package: metaplane/dbt_expectations
    version: 0.10.10
```

Then install:

```bash
dbt deps
```

> **Why pin versions?** Pinning to exact versions ensures that everyone on your team (and your CI/CD pipeline) uses the same package behaviour. Unpinned packages can silently break when a package author releases a new version.

---

## 2. Sources — Declaring Raw Data

A `source` in dbt is how you formally declare a raw table that lives outside your dbt project (typically loaded by an ELT tool or, in Jaffle Shop's case, by `dbt seed`). Using `{{ source() }}` instead of hard-coded table names gives you lineage tracking, freshness monitoring, and a single place to rename tables.

**File:** `models/staging/sources.yml`

```yaml
version: 2

sources:
  - name: jaffle_shop # Logical source name — used in {{ source('jaffle_shop', ...) }}
    description: >
      Raw Jaffle Shop café data loaded via dbt seed. Contains customers,
      orders, and payment records as they arrive from the transactional system.
    schema: main # The DuckDB schema where these tables live
    # Freshness config: warn if data is older than 12 hours, error after 24.
    # Requires a loaded_at_field column — adjust if your seeds don't have one.
    freshness:
      warn_after: { count: 12, period: hour }
      error_after: { count: 24, period: hour }

    tables:
      - name: raw_customers
        description: "One record per customer who has ever placed an order."
        columns:
          - name: id
            description: "Unique customer identifier."
            tests:
              - unique
              - not_null
          - name: first_name
            description: "Customer's first name."
          - name: last_name
            description: "Customer's last name."

      - name: raw_orders
        description: "One record per order placed at the Jaffle Shop."
        columns:
          - name: id
            description: "Unique order identifier."
            tests:
              - unique
              - not_null
          - name: user_id
            description: "Foreign key to raw_customers."
            tests:
              - not_null
              - relationships:
                  to: source('jaffle_shop', 'raw_customers')
                  field: id
          - name: status
            tests:
              - accepted_values:
                  values:
                    [
                      "placed",
                      "shipped",
                      "completed",
                      "return_pending",
                      "returned",
                    ]

      - name: raw_payments
        description: "One record per payment attempt against an order."
        columns:
          - name: id
            tests:
              - unique
              - not_null
          - name: payment_method
            tests:
              - accepted_values:
                  values:
                    ["credit_card", "coupon", "bank_transfer", "gift_card"]
```

Now reference these sources in your staging models:

```sql
-- Instead of: FROM main.raw_orders
-- Use:
SELECT * FROM {{ source('jaffle_shop', 'raw_orders') }}
```

---

## 3. Tests — Built-in Generic Tests

dbt ships with four generic tests out of the box: `unique`, `not_null`, `accepted_values`, and `relationships`. These are declared in `schema.yml` files alongside your model definitions.

**File:** `models/staging/schema.yml`

```yaml
version: 2

models:
  - name: stg_customers
    description: "Staged and lightly cleaned customer records."
    columns:
      - name: customer_id
        description: "Primary key — unique per customer."
        tests:
          - unique # Fails if any two rows share the same customer_id
          - not_null # Fails if any customer_id is NULL

      - name: first_name
        description: "Customer's first name."
        tests:
          - not_null # Every customer must have a first name

  - name: stg_orders
    description: "Staged order records with renamed columns and cleaned status."
    columns:
      - name: order_id
        tests:
          - unique
          - not_null

      - name: customer_id
        description: "Foreign key to stg_customers."
        tests:
          - not_null
          - relationships:
              # Referential integrity: every order must have a matching customer
              to: ref('stg_customers')
              field: customer_id

      - name: status
        tests:
          - accepted_values:
              values:
                - placed
                - shipped
                - completed
                - return_pending
                - returned
              # quote: true  ← add this if your DB is case-sensitive

      - name: order_date
        tests:
          - not_null

  - name: stg_payments
    description: "Staged payment records with amounts converted from cents to dollars."
    columns:
      - name: payment_id
        tests:
          - unique
          - not_null

      - name: order_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_orders')
              field: order_id

      - name: payment_method
        tests:
          - accepted_values:
              values: ["credit_card", "coupon", "bank_transfer", "gift_card"]

      - name: amount
        description: "Payment amount in dollars (converted from cents in raw data)."
        tests:
          - not_null
```

Run all tests:

```bash
dbt test

# Run tests for just one model
dbt test --select stg_orders
```

---

## 4. Tests — Singular (Custom SQL) Tests

Singular tests are plain SQL files in your `tests/` folder. A test **passes** when the query returns **zero rows** — any returned row is treated as a failure. Use these for complex business-rule checks that can't be expressed as generic tests.

**File:** `tests/assert_positive_payment_amounts.sql`

```sql
-- Test: No payment should have a negative or zero amount.
-- A refund would be a separate record, not a negative payment.
-- Returns all rows that violate the rule — zero rows = test passes.

SELECT
    payment_id,
    order_id,
    amount,
    payment_method
FROM {{ ref('stg_payments') }}
WHERE amount <= 0
```

**File:** `tests/assert_orders_have_payments.sql`

```sql
-- Test: Every 'completed' order must have at least one successful payment.
-- An order with status 'completed' but no payment record indicates a data pipeline issue.

SELECT
    o.order_id,
    o.status,
    o.customer_id
FROM {{ ref('stg_orders') }}    AS o
LEFT JOIN {{ ref('stg_payments') }} AS p
    ON o.order_id = p.order_id
WHERE
    o.status = 'completed'
    AND p.payment_id IS NULL
```

**File:** `tests/assert_customer_lifetime_value_is_positive.sql`

```sql
-- Test: Any customer who has placed at least one completed order must have
-- a positive total lifetime value in the customers mart model.

SELECT
    customer_id,
    number_of_orders,
    customer_lifetime_value
FROM {{ ref('customers') }}
WHERE
    number_of_orders > 0
    AND customer_lifetime_value <= 0
```

Run your singular tests:

```bash
dbt test --select test_type:singular
```

---

## 5. Jinja — Model References & Templating

Jinja is the templating language built into dbt. It lets you write dynamic SQL using `{{ }}` expressions and `{% %}` control blocks. The two most important Jinja functions are `ref()` and `source()`.

### `ref()` — Reference another dbt model

`{{ ref('model_name') }}` is the cornerstone of dbt. It resolves to the correct schema-qualified table name for whatever environment you're running in, and it tells dbt about model dependencies so the DAG is built correctly.

**File:** `models/staging/stg_orders.sql`

```sql
-- ref() is used to reference other models.
-- dbt resolves this to the actual table name at runtime (e.g. main.raw_orders).

WITH source AS (
    -- Reference the raw seed table via source()
    SELECT * FROM {{ source('jaffle_shop', 'raw_orders') }}
),

renamed AS (
    SELECT
        id           AS order_id,
        user_id      AS customer_id,
        order_date,
        status
    FROM source
)

SELECT * FROM renamed
```

**File:** `models/marts/orders.sql`

```sql
-- ref() builds the dependency graph automatically.
-- dbt knows stg_orders and stg_payments must be built before this model.

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}       -- depends on stg_orders
),

payments AS (
    SELECT * FROM {{ ref('stg_payments') }}     -- depends on stg_payments
),

order_payments AS (
    SELECT
        order_id,
        SUM(CASE WHEN payment_method = 'bank_transfer'
            THEN amount ELSE 0 END)             AS bank_transfer_amount,
        SUM(CASE WHEN payment_method = 'credit_card'
            THEN amount ELSE 0 END)             AS credit_card_amount,
        SUM(CASE WHEN payment_method = 'coupon'
            THEN amount ELSE 0 END)             AS coupon_amount,
        SUM(CASE WHEN payment_method = 'gift_card'
            THEN amount ELSE 0 END)             AS gift_card_amount,
        SUM(amount)                             AS total_amount
    FROM payments
    GROUP BY order_id
),

final AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        op.bank_transfer_amount,
        op.credit_card_amount,
        op.coupon_amount,
        op.gift_card_amount,
        op.total_amount
    FROM orders AS o
    LEFT JOIN order_payments AS op USING (order_id)
)

SELECT * FROM final
```

### Jinja `if` / `for` control blocks

**File:** `models/staging/stg_payments.sql`

```sql
-- Jinja for loops let you avoid repetitive SQL.
-- Here we use a loop to pivot payment methods dynamically.

WITH payments AS (
    SELECT * FROM {{ source('jaffle_shop', 'raw_payments') }}
),

{# Define the list of payment methods as a Jinja variable #}
{% set payment_methods = ['credit_card', 'coupon', 'bank_transfer', 'gift_card'] %}

renamed AS (
    SELECT
        id              AS payment_id,
        order_id,
        payment_method,
        -- Convert from cents to dollars
        amount / 100.0  AS amount

    FROM payments
)

SELECT * FROM renamed
```

**File:** `models/marts/payment_method_summary.sql`

```sql
-- Using a Jinja for loop to generate multiple CASE WHEN columns
-- without writing each one manually. Great for demonstrating Jinja power!

{% set payment_methods = ['credit_card', 'coupon', 'bank_transfer', 'gift_card'] %}

SELECT
    order_id,

    {% for method in payment_methods %}
    SUM(CASE WHEN payment_method = '{{ method }}'
        THEN amount ELSE 0 END)  AS {{ method }}_amount
    {%- if not loop.last %},{%  endif %}   {# Add comma after every line except the last #}
    {% endfor %}

FROM {{ ref('stg_payments') }}
GROUP BY order_id
```

> **What `loop.last` does:** Jinja's `loop` object has a `.last` property that is `true` on the final iteration of a `for` loop. This lets you cleanly add commas between columns without a trailing comma on the last one — which would break SQL.

---

## 6. Jinja — Variables & `var()`

dbt variables let you make your models configurable. You can define default values in `dbt_project.yml` and override them at runtime with the `--vars` flag. This is useful for parameterising date ranges, feature flags, or environment-specific settings.

### Define variables in `dbt_project.yml`

**File:** `dbt_project.yml`

```yaml
name: "jaffle_shop"
version: "1.0.0"
profile: "jaffle_shop"

# Project-level variable defaults
vars:
  # Controls how far back to look at orders data.
  # Can be overridden at runtime: dbt run --vars '{"order_start_date": "2018-01-01"}'
  order_start_date: "2017-01-01"

  # Only include completed and shipped orders in mart models by default
  # Override: dbt run --vars '{"include_all_statuses": true}'
  include_all_statuses: false

  # Minimum order value to include in reports (in dollars)
  minimum_order_amount: 0

models:
  jaffle_shop:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

### Use `var()` in a model

**File:** `models/marts/orders_filtered.sql`

```sql
-- var() retrieves a project variable. The second argument is the default
-- value if the variable is not set — this makes the model resilient.

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

payments AS (
    SELECT * FROM {{ ref('stg_payments') }}
),

order_totals AS (
    SELECT
        order_id,
        SUM(amount) AS total_amount
    FROM payments
    GROUP BY order_id
),

final AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        ot.total_amount

    FROM orders AS o
    LEFT JOIN order_totals AS ot USING (order_id)

    WHERE
        -- Filter by configurable start date
        o.order_date >= '{{ var("order_start_date", "2017-01-01") }}'

        -- Optionally filter to only completed/shipped orders
        {% if not var("include_all_statuses", false) %}
        AND o.status IN ('completed', 'shipped')
        {% endif %}

        -- Filter by minimum order value (var with a default of 0 = no filter)
        AND COALESCE(ot.total_amount, 0) >= {{ var("minimum_order_amount", 0) }}
)

SELECT * FROM final
```

### Override variables at runtime

```bash
# Run with a different start date
dbt run --select orders_filtered --vars '{"order_start_date": "2018-06-01"}'

# Include all order statuses (e.g. for debugging)
dbt run --select orders_filtered --vars '{"include_all_statuses": true}'

# Chain multiple variable overrides
dbt run --vars '{"order_start_date": "2018-01-01", "minimum_order_amount": 10}'
```

---

## 7. Macros — Writing & Using Your Own

Macros are reusable Jinja functions stored in the `macros/` folder. They work like functions: you define them once and call them in any model. They're great for eliminating repetitive SQL, enforcing consistent patterns, and abstracting business logic.

### A simple utility macro

**File:** `macros/cents_to_dollars.sql`

```sql
{#
  Macro: cents_to_dollars
  Purpose: Convert an integer amount in cents to a decimal amount in dollars.
  Usage: {{ cents_to_dollars('amount_cents') }}
         {{ cents_to_dollars('amount_cents', scale=4) }}
  Arguments:
    - column_name: the name of the column containing the cent value
    - scale: number of decimal places to round to (default: 2)
#}

{% macro cents_to_dollars(column_name, scale=2) %}
    ROUND({{ column_name }} / 100.0, {{ scale }})
{% endmacro %}
```

Use the macro in a model:

**File:** `models/staging/stg_payments.sql`

```sql
WITH source AS (
    SELECT * FROM {{ source('jaffle_shop', 'raw_payments') }}
)

SELECT
    id              AS payment_id,
    order_id,
    payment_method,
    -- Call the macro instead of writing the division logic inline
    {{ cents_to_dollars('amount') }}   AS amount

FROM source
```

---

### A macro that generates a payment method pivot

**File:** `macros/generate_payment_method_columns.sql`

```sql
{#
  Macro: generate_payment_method_columns
  Purpose: Dynamically generate one SUM(CASE WHEN...) column per payment method.
           Call this in a model to avoid repeating the same pattern for every method.
  Usage: {{ generate_payment_method_columns(['credit_card', 'coupon', 'bank_transfer']) }}
  Arguments:
    - payment_methods: a list of payment method strings
#}

{% macro generate_payment_method_columns(payment_methods) %}
    {% for method in payment_methods %}
    SUM(CASE WHEN payment_method = '{{ method }}'
        THEN amount
        ELSE 0
    END) AS {{ method }}_amount
    {%- if not loop.last %},{% endif %}
    {% endfor %}
{% endmacro %}
```

Use it in your mart model:

**File:** `models/marts/orders.sql`

```sql
{% set payment_methods = ['credit_card', 'coupon', 'bank_transfer', 'gift_card'] %}

WITH payments AS (
    SELECT * FROM {{ ref('stg_payments') }}
),

payment_method_totals AS (
    SELECT
        order_id,
        -- The macro generates all four SUM(CASE WHEN...) columns for us
        {{ generate_payment_method_columns(payment_methods) }},
        SUM(amount) AS total_amount
    FROM payments
    GROUP BY order_id
)

SELECT * FROM payment_method_totals
```

---

### A macro for a reusable date spine / fiscal year helper

**File:** `macros/is_weekend.sql`

```sql
{#
  Macro: is_weekend
  Purpose: Returns a boolean expression that is TRUE if the given date column
           falls on a Saturday or Sunday. Works with DuckDB's DAYOFWEEK function
           (1=Sunday, 7=Saturday).
  Usage: {{ is_weekend('order_date') }} AS is_weekend_order
#}

{% macro is_weekend(date_column) %}
    DAYOFWEEK({{ date_column }}) IN (1, 7)
{% endmacro %}
```

Use it:

```sql
SELECT
    order_id,
    order_date,
    {{ is_weekend('order_date') }}  AS is_weekend_order
FROM {{ ref('stg_orders') }}
```

---

### A generic test written as a macro

**File:** `macros/test_is_non_negative.sql`

```sql
{#
  Generic test: is_non_negative
  Purpose: Assert that all values in a column are >= 0.
           Used in schema.yml like any built-in generic test.
  Usage in schema.yml:
    columns:
      - name: amount
        tests:
          - is_non_negative
#}

{% test is_non_negative(model, column_name) %}

SELECT
    {{ column_name }},
    COUNT(*) AS failing_rows
FROM {{ model }}
WHERE {{ column_name }} < 0
GROUP BY {{ column_name }}

{% endtest %}
```

Register it in `schema.yml`:

```yaml
models:
  - name: stg_payments
    columns:
      - name: amount
        tests:
          - not_null
          - is_non_negative # Custom generic test defined above
```

---

## 8. Documentation — Descriptions & Doc Blocks

dbt documentation lives in your `.yml` files as `description:` fields. For longer descriptions, `doc blocks` let you write documentation in separate `.md` files and reference them, keeping your YAML clean.

### Inline descriptions in schema.yml

**File:** `models/marts/schema.yml`

```yaml
version: 2

models:
  - name: customers
    description: >
      One record per customer. Aggregates all order history to produce
      lifetime stats. This is the primary model used by the business
      intelligence team for customer-level analysis.

    columns:
      - name: customer_id
        description: "Primary key. Unique identifier per customer."
        tests:
          - unique
          - not_null

      - name: first_order_date
        description: "Date of the customer's very first order. NULL if they have never ordered."

      - name: most_recent_order_date
        description: "Date of the customer's most recent order. NULL if they have never ordered."

      - name: number_of_orders
        description: "Total count of orders placed by this customer across all time."

      - name: customer_lifetime_value
        description: >
          Total dollar value of all completed payments made by this customer.
          Calculated as the sum of all payment amounts linked to this customer's orders.
          Does not include pending or failed payments.

  - name: orders
    description: >
      One record per order. Joins order details with payment breakdowns.
      The primary model for order-level reporting and revenue analysis.

    columns:
      - name: order_id
        description: "Primary key."
        tests:
          - unique
          - not_null

      - name: status
        description: "Current status of the order in the fulfilment lifecycle."
        tests:
          - accepted_values:
              values:
                ["placed", "shipped", "completed", "return_pending", "returned"]

      - name: total_amount
        description: "Total payment amount for the order in dollars."
```

### Doc blocks — for longer, reusable documentation

**File:** `models/marts/docs.md`

```markdown
{% docs customers_model %}

## customers model

This model is the **single source of truth** for customer-level analysis at the Jaffle Shop.

### Grain

One row per unique customer (`customer_id`).

### Key metrics included

- **number_of_orders** — total lifetime order count
- **customer_lifetime_value** — total spend in dollars across all completed payments

### Important notes

- Customers who have registered but never placed an order **are included** in this model with NULL date fields and zero order counts.
- `customer_lifetime_value` only counts completed payments. Pending or returned orders are excluded.

### Downstream usage

This model feeds the **Customer Analytics** dashboard and the **Monthly Cohort Report**.
{% enddocs %}

{% docs order_status %}
The lifecycle of a Jaffle Shop order:

| Status           | Meaning                               |
| ---------------- | ------------------------------------- |
| `placed`         | Order submitted but not yet fulfilled |
| `shipped`        | Order dispatched to the customer      |
| `completed`      | Order delivered and payment confirmed |
| `return_pending` | Customer has initiated a return       |
| `returned`       | Return processed and refund issued    |

{% enddocs %}
```

Reference doc blocks in your schema YAML:

**File:** `models/marts/schema.yml` (updated)

```yaml
version: 2

models:
  - name: customers
    # Reference the doc block — keeps YAML clean, docs stay in .md
    description: "{{ doc('customers_model') }}"
    columns:
      - name: status
        description: "{{ doc('order_status') }}"
```

View your docs:

```bash
dbt docs generate
dbt docs serve
# Open http://localhost:8080 and click on the customers model
```

---

## 9. Snapshots — Tracking Historical Changes

Snapshots capture how a row changes over time, implementing **Slowly Changing Dimension Type 2 (SCD2)**. Each changed row gets a `dbt_valid_from` and `dbt_valid_to` timestamp so you can query "what did this record look like on a specific date?"

Snapshots live in the `snapshots/` folder and use `{% snapshot %}` blocks.

### Snapshot using timestamp strategy

Use this when your source table has an `updated_at` column (preferred — faster and more reliable).

**File:** `snapshots/orders_snapshot.sql`

```sql
{% snapshot orders_snapshot %}

{{
    config(
        -- Where to store the snapshot table in DuckDB
        target_schema = 'snapshots',

        -- The column that uniquely identifies each row
        unique_key     = 'order_id',

        -- Strategy: detect changes by comparing the updated_at timestamp
        -- Use 'check' strategy if there's no updated_at column
        strategy       = 'timestamp',
        updated_at     = 'updated_at'
    )
}}

SELECT
    id          AS order_id,
    user_id     AS customer_id,
    order_date,
    status,
    -- Add a synthetic updated_at since the raw Jaffle Shop data doesn't have one
    -- In a real project this would come from the source system
    CURRENT_TIMESTAMP   AS updated_at
FROM {{ source('jaffle_shop', 'raw_orders') }}

{% endsnapshot %}
```

### Snapshot using check strategy

Use this when there is no `updated_at` column. dbt compares specific column values between runs to detect changes.

**File:** `snapshots/customers_snapshot.sql`

```sql
{% snapshot customers_snapshot %}

{{
    config(
        target_schema = 'snapshots',
        unique_key     = 'customer_id',

        -- 'check' strategy: dbt hashes the specified columns and detects any change
        strategy       = 'check',
        check_cols     = ['first_name', 'last_name']  -- columns to watch for changes
    )
}}

SELECT
    id          AS customer_id,
    first_name,
    last_name
FROM {{ source('jaffle_shop', 'raw_customers') }}

{% endsnapshot %}
```

### Run the snapshot

```bash
dbt snapshot
```

### Query the snapshot — historical lookups

Once populated, the snapshot table includes four dbt-managed columns:

| Column           | Meaning                                                  |
| ---------------- | -------------------------------------------------------- |
| `dbt_scd_id`     | Unique identifier for each version of a row              |
| `dbt_updated_at` | When dbt detected this version                           |
| `dbt_valid_from` | When this version became active                          |
| `dbt_valid_to`   | When this version was superseded (NULL = current record) |

```sql
-- Get the CURRENT state of all orders (dbt_valid_to IS NULL = latest version)
SELECT *
FROM snapshots.orders_snapshot
WHERE dbt_valid_to IS NULL;

-- Get the state of order 42 as it was on a specific date
SELECT *
FROM snapshots.orders_snapshot
WHERE order_id = 42
  AND dbt_valid_from  <= '2018-03-01'
  AND (dbt_valid_to    > '2018-03-01' OR dbt_valid_to IS NULL);

-- See the full history of status changes for order 42
SELECT
    order_id,
    status,
    dbt_valid_from,
    dbt_valid_to
FROM snapshots.orders_snapshot
WHERE order_id = 42
ORDER BY dbt_valid_from;
```

---

## 10. Incremental Models

Incremental models only process **new or changed rows** on each run instead of rebuilding the entire table. Essential for large tables where a full rebuild would be too slow or expensive.

**File:** `models/marts/orders_incremental.sql`

```sql
{{
    config(
        -- Materialize as an incremental table
        materialized = 'incremental',

        -- The column used to identify new rows (must be monotonically increasing)
        unique_key   = 'order_id',

        -- on_schema_change: what to do if the model's columns change
        -- 'fail' is safest — forces you to explicitly handle schema changes
        on_schema_change = 'fail'
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

payments AS (
    SELECT * FROM {{ ref('stg_payments') }}
),

order_totals AS (
    SELECT
        order_id,
        SUM(amount)     AS total_amount
    FROM payments
    GROUP BY order_id
),

final AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        ot.total_amount
    FROM orders AS o
    LEFT JOIN order_totals AS ot USING (order_id)
)

SELECT * FROM final

-- The is_incremental() block ONLY runs when doing an incremental update,
-- not on the first full build. It filters to only new rows since the last run.
{% if is_incremental() %}

WHERE o.order_date > (
    SELECT MAX(order_date) FROM {{ this }}  -- {{ this }} refers to the existing table
)

{% endif %}
```

Run incremental:

```bash
# Normal run — only processes new rows since last build
dbt run --select orders_incremental

# Force a full rebuild (useful after schema changes or data corrections)
dbt run --select orders_incremental --full-refresh
```

---

## 11. Data Contracts — Schema Enforcement

Data contracts (introduced in dbt Core 1.5) enforce that a model's output columns match exactly what you've declared — the right names, data types, and constraints. If the model's SQL returns a different schema, **dbt refuses to build it**. This prevents silent breaking changes from propagating downstream.

Contracts only work on `table` and `incremental` materializations — not on views.

### Applying a contract to the customers mart

**File:** `models/marts/schema.yml`

```yaml
version: 2

models:
  - name: customers
    description: "{{ doc('customers_model') }}"

    config:
      # Materialize as a table (required for contracts)
      materialized: table

      # Enforce the contract — dbt will fail the build if columns don't match
      contract:
        enforced: true

    columns:
      # Every column MUST be listed with a name and data_type when enforced: true
      - name: customer_id
        data_type: integer
        description: "Primary key."
        constraints:
          - type: not_null # Platform-level constraint (enforced by DuckDB)
          - type: primary_key # Metadata only in DuckDB — documents intent
        tests:
          - unique # dbt test (runs after build, separate from constraints)
          - not_null

      - name: first_name
        data_type: varchar
        description: "Customer's first name."

      - name: last_name
        data_type: varchar
        description: "Customer's last name."

      - name: first_order_date
        data_type: date
        description: "Date of customer's first order. NULL if no orders placed."

      - name: most_recent_order_date
        data_type: date
        description: "Date of customer's most recent order."

      - name: number_of_orders
        data_type: integer
        description: "Count of all orders placed by this customer."
        constraints:
          - type: not_null

      - name: customer_lifetime_value
        data_type: double
        description: "Total spend in dollars."
```

### Applying a contract to the orders mart

**File:** `models/marts/schema.yml` (orders section)

```yaml
- name: orders
  config:
    materialized: table
    contract:
      enforced: true

  columns:
    - name: order_id
      data_type: integer
      constraints:
        - type: not_null
        - type: primary_key
      tests:
        - unique
        - not_null

    - name: customer_id
      data_type: integer
      constraints:
        - type: not_null
      tests:
        - relationships:
            to: ref('customers')
            field: customer_id

    - name: order_date
      data_type: date
      constraints:
        - type: not_null

    - name: status
      data_type: varchar
      tests:
        - accepted_values:
            values:
              ["placed", "shipped", "completed", "return_pending", "returned"]

    - name: total_amount
      data_type: double

    - name: credit_card_amount
      data_type: double

    - name: bank_transfer_amount
      data_type: double

    - name: gift_card_amount
      data_type: double

    - name: coupon_amount
      data_type: double
```

### Enforce contracts project-wide

To enforce contracts across all models in a folder (e.g. your entire `marts/` layer):

**File:** `dbt_project.yml`

```yaml
models:
  jaffle_shop:
    marts:
      +materialized: table
      +contract:
        enforced: true # All mart models must declare full column contracts
```

> **What happens when a contract is violated?** dbt runs a "preflight" check before materializing. If your SQL returns a column named `total_amt` but your contract declares `total_amount`, the build fails immediately with a clear error message — no silent data drift.

---

## 12. dbt-utils — Utility Macros & Tests

`dbt-utils` is the official utility package from dbt Labs. It provides macros for common SQL patterns and additional generic tests that aren't built into dbt core.

Make sure it's in `packages.yml` and run `dbt deps` first.

### Generic tests from dbt-utils

**File:** `models/marts/schema.yml`

```yaml
version: 2

models:
  - name: orders
    tests:
      # Test that the combination of customer_id + order_date is unique
      # (a customer should only have one order per day in this dataset)
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - customer_id
            - order_date

    columns:
      - name: total_amount
        tests:
          # Verify total_amount is between 0 and 5000 dollars
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 5000
              inclusive: true # 0 is allowed (free orders via coupon)

      - name: order_id
        tests:
          # Verify no NULL proportion exceeds 5% (allows for some tolerance)
          - dbt_utils.not_null_proportion:
              at_least: 0.95

  - name: customers
    columns:
      - name: number_of_orders
        tests:
          # Verify order count is non-negative and reasonable
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 1000
              inclusive: true

      - name: customer_lifetime_value
        tests:
          # Assert that CLV = 0 is only valid when number_of_orders = 0
          - dbt_utils.expression_is_true:
              expression: ">= 0" # CLV should never be negative
```

### Macro: `dbt_utils.generate_surrogate_key`

Generates a surrogate key by hashing multiple columns. Useful when your data has no natural primary key.

**File:** `models/staging/stg_payments.sql`

```sql
WITH source AS (
    SELECT * FROM {{ source('jaffle_shop', 'raw_payments') }}
)

SELECT
    -- Generate a surrogate key from the combination of order_id + payment_method
    -- This ensures idempotent inserts in incremental models
    {{ dbt_utils.generate_surrogate_key(['id', 'order_id', 'payment_method']) }}
        AS payment_surrogate_key,

    id              AS payment_id,
    order_id,
    payment_method,
    {{ cents_to_dollars('amount') }} AS amount

FROM source
```

### Macro: `dbt_utils.date_spine`

Generates a complete series of dates between two endpoints. Extremely useful for reporting models where you need to show all dates even when there are no orders.

**File:** `models/marts/date_spine.sql`

```sql
-- Generate one row per day from the first Jaffle Shop order to today.
-- Use this to LEFT JOIN against orders data so that days with no orders
-- still appear in your charts (instead of being missing entirely).

WITH date_spine AS (
    {{ dbt_utils.date_spine(
        datepart  = "day",
        start_date = "cast('2017-01-01' as date)",
        end_date   = "cast(current_date as date)"
    ) }}
)

SELECT
    date_day,
    EXTRACT(YEAR  FROM date_day)  AS year,
    EXTRACT(MONTH FROM date_day)  AS month,
    EXTRACT(DOW   FROM date_day)  AS day_of_week,
    {{ is_weekend('date_day') }}  AS is_weekend
FROM date_spine
```

### Macro: `dbt_utils.star`

Selects all columns from a relation except the ones you want to exclude. Useful for adding computed columns without listing every original column.

**File:** `models/staging/stg_orders_enriched.sql`

```sql
-- Select all columns from stg_orders EXCEPT status, then add a cleaned version
SELECT
    {{ dbt_utils.star(from=ref('stg_orders'), except=['status']) }},

    -- Replace raw status with a cleaned/standardised version
    LOWER(TRIM(status)) AS status

FROM {{ ref('stg_orders') }}
```

### Macro: `dbt_utils.pivot`

Dynamically pivots rows into columns based on a list of values.

**File:** `models/marts/orders_pivoted.sql`

```sql
-- Pivot order counts by status into separate columns
-- Result: one row per customer with columns like: placed_count, shipped_count, etc.

SELECT
    customer_id,
    {{ dbt_utils.pivot(
        column       = 'status',
        values       = ['placed', 'shipped', 'completed', 'return_pending', 'returned'],
        agg          = 'COUNT',
        then_value   = 'order_id',
        suffix       = '_count'
    ) }}
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

---

## 13. dbt-expectations — Advanced Data Quality Tests

`dbt-expectations` brings Great Expectations-style assertions to dbt. It covers complex scenarios that built-in tests can't handle — statistical ranges, regex patterns, row counts, column type validation, and cross-column comparisons.

> **Package note:** As of late 2024, the original `calogica/dbt_expectations` package is no longer actively maintained. Use the community fork: `metaplane/dbt_expectations` (version `0.10.10+`). Update your `packages.yml` accordingly.

### Row-count and table-level tests

**File:** `models/staging/schema.yml`

```yaml
version: 2

models:
  - name: stg_orders
    description: "Staged orders — must always have meaningful row counts."
    tests:
      # The orders table should have at least 50 rows at all times.
      # Catches silent truncation or failed seed loads.
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 50
          max_value: 10000

      # The row count of stg_orders should match raw_orders (no rows lost in staging)
      - dbt_expectations.expect_table_row_count_to_equal_other_table:
          compare_model: source('jaffle_shop', 'raw_orders')

      # The orders table must have exactly these columns — no more, no fewer.
      # Great for catching accidental column additions or removals.
      - dbt_expectations.expect_table_columns_to_match_set:
          column_list:
            - order_id
            - customer_id
            - order_date
            - status
```

### Column value tests

**File:** `models/marts/schema.yml`

```yaml
version: 2

models:
  - name: orders
    columns:
      - name: total_amount
        tests:
          # Confirm the column is a numeric type at the database level
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: double

          # total_amount should be between 0 and 5000 (catch extreme outliers)
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 5000
              # row_condition filters which rows the test applies to
              row_condition: "status = 'completed'"

          # No more than 5% of total_amount values should be NULL
          - dbt_expectations.expect_column_values_to_not_be_null:
              mostly: 0.95 # 95% of rows must be non-null (allows 5% tolerance)

      - name: status
        tests:
          # Regex match: status must be lowercase letters only, no spaces
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[a-z_]+$"

          # Exactly these values and no others
          - dbt_expectations.expect_column_values_to_be_in_set:
              value_set:
                ["placed", "shipped", "completed", "return_pending", "returned"]

      - name: order_date
        tests:
          # All order dates should be after the Jaffle Shop opening date
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2016-01-01'::date"
              max_value: "current_date"
              row_condition: "order_date IS NOT NULL"

          # Data should be fresh — most recent order within the last 365 days
          - dbt_expectations.expect_row_values_to_have_recent_data:
              datepart: day
              interval: 365

  - name: customers
    columns:
      - name: customer_lifetime_value
        tests:
          # CLV should never be negative
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              row_condition: "number_of_orders > 0"

          # The mean CLV should be between $10 and $500 (statistical sanity check)
          - dbt_expectations.expect_column_mean_to_be_between:
              min_value: 10
              max_value: 500
              row_condition: "number_of_orders > 0"

          # Standard deviation check — catches extreme data skew
          - dbt_expectations.expect_column_stdev_to_be_between:
              min_value: 1
              max_value: 200

      - name: number_of_orders
        tests:
          # Median order count should be reasonable
          - dbt_expectations.expect_column_median_to_be_between:
              min_value: 1
              max_value: 10
```

### Cross-column comparison tests

**File:** `models/marts/schema.yml`

```yaml
- name: orders
  tests:
    # The sum of all payment method columns should equal total_amount
    # (no money is created or lost in the split)
    - dbt_expectations.expect_column_pair_values_A_to_be_greater_than_B:
        column_A: total_amount
        column_B: credit_card_amount
        or_equal: true # total >= credit_card (credit card can't exceed total)
        row_condition: "status = 'completed'"

  columns:
    - name: total_amount
      tests:
        # Validate the values in total_amount match an expression across other columns
        # This is the dbt-expectations equivalent of a cross-column integrity check
        - dbt_expectations.expect_column_values_to_be_between:
            min_value: 0
            max_value: 5000
```

### Distribution and uniqueness tests

**File:** `models/staging/schema.yml`

```yaml
- name: stg_payments
  columns:
    - name: payment_method
      tests:
        # Check the proportion of rows using each payment method.
        # Catches sudden spikes or collapses in payment method usage.
        - dbt_expectations.expect_column_proportion_of_unique_values_to_be_between:
            min_value: 0.01 # At least 1% unique-ish proportion
            max_value: 1.0

    - name: payment_id
      tests:
        # Exactly 100% of payment_ids must be unique (no duplicates allowed)
        - dbt_expectations.expect_column_proportion_of_unique_values_to_be_between:
            min_value: 1.0
            max_value: 1.0
```

Run only dbt-expectations tests:

```bash
# Run all tests with 'expect' in the test name
dbt test --select "test_name:*expect*"
```

---

## 14. Putting It All Together — The Full Pipeline

Here is how all the concepts connect in the Jaffle Shop project, from raw seeds to tested, contracted, documented mart models.

### Complete `packages.yml`

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.0
  - package: metaplane/dbt_expectations
    version: 0.10.10
```

### Complete `dbt_project.yml`

```yaml
name: "jaffle_shop"
version: "1.0.0"
profile: "jaffle_shop"

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

vars:
  order_start_date: "2017-01-01"
  include_all_statuses: false
  minimum_order_amount: 0

models:
  jaffle_shop:
    staging:
      +materialized: view
    marts:
      +materialized: table
      +contract:
        enforced: true
```

### Full pipeline execution sequence

```bash
# 1. Install packages
dbt deps

# 2. Verify the project and connection
dbt debug

# 3. Load seed data (raw CSVs → DuckDB tables)
dbt seed

# 4. Run snapshots (capture slowly-changing history)
dbt snapshot

# 5. Build all models + run all tests (in DAG order, stops on test failure)
dbt build

# 6. Check source freshness
dbt source freshness

# 7. Generate and view documentation
dbt docs generate
dbt docs serve
```

### One-line build with variable overrides

```bash
# Run the full build including a different date range and all statuses
dbt build --vars '{"order_start_date": "2018-01-01", "include_all_statuses": true}'
```

### Selective runs by concept

```bash
# Demonstrate just the staging layer
dbt build --select models/staging/

# Demonstrate the marts layer and all its tests
dbt build --select models/marts/

# Show the snapshot running
dbt snapshot

# Show a specific model's compiled SQL
dbt compile --select customers

# Preview the contracts mart model output
dbt show --select customers --limit 10

# Run only dbt-expectations tests
dbt test --select "test_name:*expect*"

# Run only dbt-utils tests
dbt test --select "test_name:*dbt_utils*"
```

---

_Code snippet guide for dbt-duckdb on Windows 11 — dbt Core 1.8+ / dbt-duckdb 1.8+_
_dbt-utils: https://github.com/dbt-labs/dbt-utils_
_dbt-expectations (Metaplane fork): https://github.com/metaplane/dbt-expectations_
_Official dbt docs: https://docs.getdbt.com_
