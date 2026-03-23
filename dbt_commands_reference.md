# dbt Command Reference Guide

### Using dbt with DuckDB on Windows 11

> A comprehensive reference for all essential dbt CLI commands, with examples and tips for learning dbt locally with the `dbt-duckdb` adapter.

---

## Table of Contents

1. [Setup & Verification](#1-setup--verification)
2. [Core Workflow — Build & Test](#2-core-workflow--build--test)
3. [Seed Data](#3-seed-data)
4. [Snapshots & Incremental History](#4-snapshots--incremental-history)
5. [Documentation](#5-documentation)
6. [Compile & Inspect](#6-compile--inspect)
7. [Utilities](#7-utilities)
8. [Selector Syntax Cheat Sheet](#8-selector-syntax-cheat-sheet)
9. [Recommended Learning Workflow](#9-recommended-learning-workflow)

---

## 1. Setup & Verification

### `dbt debug`

**What it does:** Performs a full health check of your dbt project. It validates your `profiles.yml` connection settings, checks that the target database is reachable, confirms the dbt project file (`dbt_project.yml`) is valid, and verifies that the installed dbt version is compatible with your project.

**When to use it:**

- First thing when starting a brand new project
- After editing your `profiles.yml` or `dbt_project.yml`
- Any time you get cryptic connection or configuration errors

**With DuckDB specifically:** This will confirm that your `.duckdb` file path is correct and that the `dbt-duckdb` adapter is properly installed. All checks should show a green `OK` — if any show `ERROR`, fix those before proceeding.

```bash
dbt debug
```

**Example output (success):**

```
All checks passed!
```

---

### `dbt deps`

**What it does:** Reads your `packages.yml` file and downloads all listed dbt packages into the `dbt_packages/` directory. Works similarly to `npm install` in Node.js or `pip install -r requirements.txt` in Python.

**When to use it:**

- Once after creating or cloning a project that has a `packages.yml`
- After adding a new package entry to `packages.yml`
- After running `dbt clean` (which deletes the `dbt_packages/` folder)

**Common packages to know:**

- `dbt-utils` — utility macros for common SQL patterns
- `dbt-expectations` — extra data quality test types
- `dbt-audit-helper` — compare model outputs between runs

```bash
dbt deps
```

**Example `packages.yml`:**

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

---

## 2. Core Workflow — Build & Test

### `dbt run`

**What it does:** The primary command in dbt. It reads all your `.sql` model files, renders any Jinja templating, and executes the SQL against DuckDB. Each model is materialized in the database according to its configured type — `view`, `table`, `incremental`, or `ephemeral`.

**When to use it:**

- During active development when iterating on a model
- When you want to build only specific models without running tests
- When you need fine-grained control over exactly what gets executed

**Important flags:**

| Flag               | Purpose                                              |
| ------------------ | ---------------------------------------------------- |
| `--select <name>`  | Run a specific model or set of models                |
| `--exclude <name>` | Skip specific models                                 |
| `--full-refresh`   | Force a full rebuild of incremental models           |
| `--target <name>`  | Use a specific profile target (e.g. `dev` vs `prod`) |
| `--vars`           | Pass runtime variables into your project             |

```bash
# Run all models
dbt run

# Run a single model
dbt run --select my_model

# Run a model and all of its upstream dependencies
dbt run --select +my_model

# Run a model and all models downstream of it
dbt run --select my_model+

# Run a model plus everything upstream AND downstream
dbt run --select +my_model+

# Run all models in a folder
dbt run --select models/staging/

# Run all models with a specific tag
dbt run --select tag:finance

# Force-rebuild an incremental model from scratch
dbt run --select my_model --full-refresh
```

> **Tip for DuckDB:** Runs are near-instant locally since DuckDB is an in-process database. Take advantage of this to iterate quickly.

---

### `dbt test`

**What it does:** Executes all data quality tests defined in your schema `.yml` files and any standalone test SQL files in the `tests/` folder. Tests do **not** modify any data — they only query the database and raise a `WARN` or `ERROR` if expectations are not met.

**Built-in generic tests:**

- `not_null` — ensures a column has no null values
- `unique` — ensures all values in a column are distinct
- `accepted_values` — checks that column values are within a defined list
- `relationships` — checks referential integrity between models (like a foreign key check)

**When to use it:**

- After `dbt run` to validate the quality of the output data
- During CI/CD pipelines to gate deployments on data quality
- When debugging data issues to isolate which model introduced a problem

```bash
# Run all tests
dbt test

# Run tests only for a specific model
dbt test --select my_model

# Run only tests of a specific type
dbt test --select test_type:generic
dbt test --select test_type:singular

# Run tests for a model and all its upstream models
dbt test --select +my_model
```

**Example schema test definition in `schema.yml`:**

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ["placed", "shipped", "delivered", "cancelled"]
```

---

### `dbt build`

**What it does:** A single command that runs `dbt seed` + `dbt run` + `dbt snapshot` + `dbt test` together, in correct DAG (dependency) order. Critically, it runs tests _as it goes_ — if a model's tests fail, downstream models that depend on it are **not** built. This makes it much safer than running `dbt run` and `dbt test` separately.

**When to use it:**

- For full pipeline runs in production or CI/CD
- When you want the safety guarantee that downstream models only build if upstream tests pass
- As a daily scheduled job

**When to prefer `dbt run` instead:**

- During development when you're iterating quickly and don't want to wait for every test on every run

```bash
# Full build of everything
dbt build

# Build a model and everything downstream
dbt build --select my_model+

# Build with full refresh for incrementals
dbt build --full-refresh
```

> **Key difference from `dbt run`:** With `dbt run`, even if a model produces bad data, downstream models will still be built on top of that bad data. With `dbt build`, the tests act as a gate.

---

## 3. Seed Data

### `dbt seed`

**What it does:** Reads CSV files from your project's `seeds/` directory and loads them as tables into DuckDB. The table name matches the CSV filename. Seeds are version-controlled alongside your project, making them great for small reference or lookup tables.

**When to use it:**

- When you have small static lookup tables (country codes, product categories, fiscal calendars, mapping tables)
- When you want reference data that's consistent across all environments (dev, staging, prod)
- At the start of a project setup, before running models that depend on seed data

**When NOT to use seeds:**

- For large datasets (anything over a few thousand rows should be loaded via proper ETL)
- For data that changes frequently

```bash
# Load all seeds
dbt seed

# Load a specific seed file
dbt seed --select country_codes

# Force a full reload (drops and recreates the table)
dbt seed --full-refresh
```

**Example:** A file at `seeds/status_codes.csv` will be loaded as a table called `status_codes` in DuckDB, and can then be referenced in models with `{{ ref('status_codes') }}`.

---

## 4. Snapshots & Incremental History

### `dbt snapshot`

**What it does:** Executes snapshot definitions stored in your `snapshots/` folder. Snapshots implement **Slowly Changing Dimension Type 2 (SCD2)** — they track how rows in a source table change over time by adding `dbt_valid_from` and `dbt_valid_to` timestamp columns. Each time you run `dbt snapshot`, dbt detects changed rows, closes the old record (sets `dbt_valid_to`), and inserts a new current record.

**When to use it:**

- When you need a full history of how a record changed (e.g. a customer's address, an order's status)
- When your source system only keeps the current state and you need to track changes

**Snapshot strategies:**

- `timestamp` — detects changes by comparing an `updated_at` timestamp column (preferred)
- `check` — detects changes by comparing specific column values (slower but works without timestamps)

```bash
dbt snapshot
```

**Example snapshot definition:**

```sql
{% snapshot orders_snapshot %}
  {{
    config(
      target_schema='snapshots',
      unique_key='order_id',
      strategy='timestamp',
      updated_at='updated_at'
    )
  }}
  select * from {{ source('raw', 'orders') }}
{% endsnapshot %}
```

> **Note for beginners:** Snapshots are an advanced topic. You don't need them for basic dbt learning — come back to this once you're comfortable with `dbt run` and `dbt test`.

---

## 5. Documentation

### `dbt docs generate`

**What it does:** Compiles your entire project into a documentation website. It reads all your model `.sql` files, schema `.yml` files (descriptions, column definitions, test definitions), and the project graph, and writes a static site to the `target/` folder. Also generates a `manifest.json` and `catalog.json` that describe the full project structure.

**When to use it:**

- Before running `dbt docs serve` (you must generate before serving)
- After adding descriptions to your models or columns
- As part of a CI pipeline to publish docs automatically

```bash
dbt docs generate
```

---

### `dbt docs serve`

**What it does:** Starts a local HTTP server that serves the generated documentation website. Opens in your browser at `http://localhost:8080` by default. The site includes a searchable model catalog, column-level documentation, and — most importantly — an **interactive lineage DAG** that visually shows how all your models relate to each other.

**When to use it:**

- After `dbt docs generate`, whenever you want to explore your project visually
- When onboarding a new team member to the project
- When debugging complex model dependencies

```bash
# Serve on default port 8080
dbt docs serve

# Use a different port if 8080 is occupied
dbt docs serve --port 8001
```

> **Pro tip:** The lineage graph in the docs UI is one of dbt's best features. You can click on any node to see its upstream sources and downstream dependents, and filter the graph to focus on specific parts of your project.

---

## 6. Compile & Inspect

### `dbt compile`

**What it does:** Renders all Jinja templating in your SQL files (resolving `{{ ref() }}`, `{{ source() }}`, `{{ config() }}`, macros, etc.) and writes the resulting plain SQL to `target/compiled/<project_name>/models/`. It does **not** execute anything against the database.

**When to use it:**

- To see the exact SQL that dbt will run, before actually running it
- When debugging complex Jinja expressions or macros
- To understand what a `{{ ref('model') }}` call resolves to in context

```bash
# Compile all models
dbt compile

# Compile a specific model
dbt compile --select my_model
```

**With VS Code:** After compiling, open `target/compiled/<project>/models/my_model.sql` to inspect the output. This is invaluable when learning how Jinja templating works in dbt.

---

### `dbt show`

**What it does:** Compiles a model and immediately executes it, printing the first few rows of results directly in your terminal. Gives you a fast inline data preview without needing to open DuckDB or run a full `dbt run`.

**Requires:** dbt Core 1.5 or later.

**When to use it:**

- Quick sanity checks during development ("does this model return what I expect?")
- Verifying a transformation looks correct before materializing it
- Faster than running the full model when you just want to see a sample

```bash
# Show first 5 rows (default)
dbt show --select my_model

# Show more rows
dbt show --select my_model --limit 20
```

---

### `dbt parse`

**What it does:** Parses the entire project — reads all `.sql`, `.yml`, and config files — and checks for syntax errors, invalid references, and misconfigured settings. Produces a `manifest.json` in the `target/` folder. It is much faster than `dbt compile` or `dbt run` because it doesn't execute any SQL.

**When to use it:**

- As a quick project validation step (e.g. in CI before running the full pipeline)
- To check for errors after bulk-editing YAML config files
- To pre-generate a `manifest.json` for use with other tooling

```bash
dbt parse
```

---

## 7. Utilities

### `dbt clean`

**What it does:** Deletes the folders listed under `clean-targets` in your `dbt_project.yml` — by default this is `target/` (compiled SQL, artifacts) and `dbt_packages/` (installed packages). Nothing in these folders is source-of-truth; they are always regenerated from your source files.

**When to use it:**

- When you suspect stale compiled artifacts are causing confusing behaviour
- Before a fresh deployment to ensure a clean slate
- After switching branches with very different project structures

```bash
dbt clean
```

> **Remember:** After `dbt clean`, you'll need to run `dbt deps` again to reinstall packages before running other commands.

---

### `dbt ls` (list)

**What it does:** Lists all resources in your project that match a given selector, without running or building anything. It supports the same `--select` syntax as `dbt run`. You can filter by resource type (models, tests, sources, seeds, snapshots, exposures) and by selector expressions.

**When to use it:**

- **Before running** a complex selector expression, to verify it targets what you expect
- To explore which models belong to a tag, a folder, or a specific part of the DAG
- To audit your project structure

```bash
# List all resources
dbt ls

# List all models only
dbt ls --resource-type model

# List models matching a selector (check before running!)
dbt ls --select +my_model
dbt ls --select tag:marketing
dbt ls --select models/staging/

# List all tests
dbt ls --resource-type test

# List all sources
dbt ls --resource-type source
```

> **Best practice:** Always run `dbt ls --select <your_selector>` before `dbt run --select <your_selector>` to confirm you're targeting the right models.

---

### `dbt source freshness`

**What it does:** Queries your source tables and checks whether the data is "fresh" based on thresholds you define in your `sources.yml` config. For each source, you specify how old the most recent row is allowed to be before raising a `warn` or `error`. Helps catch upstream pipeline failures before they silently propagate through your dbt models.

**When to use it:**

- At the start of a pipeline run, before building any models
- In monitoring/alerting setups to detect stale source data
- Any time you suspect an upstream feed has stopped updating

```bash
dbt source freshness
```

**Example freshness config in `sources.yml`:**

```yaml
sources:
  - name: raw
    tables:
      - name: orders
        freshness:
          warn_after: { count: 12, period: hour }
          error_after: { count: 24, period: hour }
        loaded_at_field: _loaded_at
```

---

## 8. Selector Syntax Cheat Sheet

The `--select` flag is the most powerful part of the dbt CLI. It uses a graph-based syntax that lets you precisely control which models are included.

| Selector                    | Meaning                                                         |
| --------------------------- | --------------------------------------------------------------- |
| `my_model`                  | Only `my_model`                                                 |
| `+my_model`                 | `my_model` and all upstream models (parents, grandparents…)     |
| `my_model+`                 | `my_model` and all downstream models (children, grandchildren…) |
| `+my_model+`                | `my_model`, all upstream, and all downstream                    |
| `my_model+1`                | `my_model` and one level of children only                       |
| `tag:finance`               | All models tagged with `finance`                                |
| `models/staging/`           | All models in the `staging/` subdirectory                       |
| `source:raw`                | All models that read from the `raw` source                      |
| `config.materialized:table` | All models materialised as tables                               |
| `path:models/marts/`        | All models under the `marts/` path                              |

**Combining selectors:**

```bash
# Run staging models AND their downstream dependents
dbt run --select models/staging/+

# Run everything EXCEPT a specific model
dbt run --exclude my_broken_model

# Run models matching multiple selectors (union)
dbt run --select tag:finance tag:hr
```

---

## 9. Recommended Learning Workflow

Here is a practical sequence to follow when learning dbt with DuckDB:

### First-time project setup

```bash
dbt debug          # 1. Verify config & DuckDB connection
dbt deps           # 2. Install packages from packages.yml
dbt seed           # 3. Load any CSV seed files
dbt run            # 4. Build all models
dbt test           # 5. Validate data quality
dbt docs generate  # 6. Generate documentation
dbt docs serve     # 7. Explore in the browser
```

### Daily development loop

```bash
# Iterate on a specific model
dbt compile --select my_model   # Check the rendered SQL first
dbt run --select my_model       # Build the model
dbt test --select my_model      # Run its tests
dbt show --select my_model      # Preview output rows

# When ready to validate the full pipeline
dbt build                       # Run everything with integrated testing
```

### Useful VS Code tip

Install the **dbt Power User** extension for VS Code. It provides:

- Run/test the currently open model with a keyboard shortcut
- Inline column previews
- Lineage graph view directly inside VS Code
- Auto-completion for `ref()` and `source()` calls

This pairs especially well with DuckDB since local runs are near-instant.

---

_Generated for dbt-duckdb on Windows 11 — dbt Core 1.5+_
