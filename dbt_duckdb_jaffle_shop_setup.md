# Setting Up a dbt + DuckDB Project in VS Code (Windows 11)

### Using the Jaffle Shop DuckDB Dataset — YouTube Tutorial Setup Guide

> This guide walks you through every step to get a fully working dbt + DuckDB project running locally in VS Code on Windows 11, using the official `jaffle_shop_duckdb` demo dataset from dbt Labs.

---

## Table of Contents

1. [What is the Jaffle Shop?](#1-what-is-the-jaffle-shop)
2. [Prerequisites](#2-prerequisites)
3. [Install Required Software](#3-install-required-software)
4. [Set Up the Project — Two Approaches](#4-set-up-the-project--two-approaches)
   - [Option A: Clone the Official Jaffle Shop DuckDB Repo](#option-a-clone-the-official-jaffle-shop-duckdb-repo-recommended-for-beginners)
   - [Option B: Build from Scratch with dbt init](#option-b-build-from-scratch-with-dbt-init)
5. [Install VS Code Extensions](#5-install-vs-code-extensions)
6. [Understand the Project Structure](#6-understand-the-project-structure)
7. [Run the Full Pipeline](#7-run-the-full-pipeline)
8. [Explore Your Data](#8-explore-your-data)
9. [Generate & View Documentation](#9-generate--view-documentation)
10. [Common Errors & Fixes](#10-common-errors--fixes)
11. [What to Show on Your YouTube Tutorial](#11-what-to-show-on-your-youtube-tutorial)

---

## 1. What is the Jaffle Shop?

The **Jaffle Shop** is a fictional café/sandwich shop created by dbt Labs as an official learning dataset. It is the standard demo project used in dbt tutorials worldwide. The dataset simulates a small e-commerce operation and includes:

| Table           | Description                                         |
| --------------- | --------------------------------------------------- |
| `raw_customers` | Customer records (id, first name, last name)        |
| `raw_orders`    | Order records (id, customer, date, status)          |
| `raw_payments`  | Payment records (order id, method, amount in cents) |

From this raw data, dbt builds a set of **staging models** (cleaned, renamed columns) and **mart models** (business-ready aggregations like `customers` and `orders`).

The `jaffle_shop_duckdb` variant is specifically designed for **local development** — no cloud warehouse needed. All data lives in a single `.duckdb` file on your machine.

> A "jaffle" is a toasted sandwich with crimped sealed edges, invented in Australia in 1949. The name is just dbt's way of keeping things fun.

---

## 2. Prerequisites

Before starting, make sure you have the following installed and ready:

### Required

- **Windows 11** (this guide is written for Windows)
- **Python 3.9 or higher** — dbt runs on Python
- **Git** — to clone the repository
- **VS Code** — your editor

### Check your existing installs

Open a terminal in VS Code (`Ctrl + `` ` ``) and run:

```powershell
# Check Python version (must be 3.9+)
python --version

# Check pip is available
pip --version

# Check Git is installed
git --version
```

If any of these fail, install them before continuing:

- **Python**: https://www.python.org/downloads/ — ✅ Check "Add Python to PATH" during install
- **Git for Windows**: https://git-scm.com/download/win
- **VS Code**: https://code.visualstudio.com/

---

## 3. Install Required Software

### Step 3.1 — Create a project folder

Open VS Code and open its integrated terminal (`Ctrl + `` ` ``). Navigate to wherever you keep your projects and create a working directory:

```powershell
# Navigate to your projects folder (adjust path as needed)
cd C:\Users\YourName\Documents\

# Create a folder for your dbt work
mkdir dbt_projects
cd dbt_projects
```

### Step 3.2 — Create a Python virtual environment

A virtual environment keeps your dbt installation isolated from other Python projects on your machine. This is best practice and avoids version conflicts.

```powershell
# Create the virtual environment (named "venv")
python -m venv venv

# Activate the virtual environment (Windows Command Prompt)
venv\Scripts\activate.bat

# OR if you're using PowerShell
venv\Scripts\Activate.ps1
```

> ⚠️ **PowerShell execution policy note:** If you get an error about scripts being disabled in PowerShell, run this first:
>
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

Once activated, your terminal prompt will change to show `(venv)` at the start — this means you're inside the virtual environment. **All pip installs from this point go into the venv, not your system Python.**

### Step 3.3 — Upgrade pip

Always upgrade pip before installing packages to avoid resolver issues:

```powershell
python -m pip install --upgrade pip
```

### Step 3.4 — Install dbt-core and the DuckDB adapter

Since dbt Core 1.8, the core package and adapters must be installed separately:

```powershell
pip install dbt-core dbt-duckdb
```

This installs:

- `dbt-core` — the main dbt engine and CLI
- `dbt-duckdb` — the adapter that lets dbt talk to DuckDB

### Step 3.5 — Verify the installation

```powershell
dbt --version
```

You should see output similar to:

```
Core:
  - installed: 1.8.x
  - latest:    1.8.x - Up to date!

Plugins:
  - duckdb: 1.8.x - Up to date!
```

---

## 4. Set Up the Project — Two Approaches

Choose **Option A** if you want to jump straight into a working project (recommended for a YouTube tutorial demo). Choose **Option B** if you want to show viewers how to build a project from scratch.

---

### Option A: Clone the Official Jaffle Shop DuckDB Repo _(Recommended for beginners)_

This clones the ready-made project from dbt Labs, which already includes all models, seeds, tests, and a `profiles.yml` configured for DuckDB.

#### Step A.1 — Clone the repository

```powershell
git clone https://github.com/dbt-labs/jaffle_shop_duckdb.git
cd jaffle_shop_duckdb
```

#### Step A.2 — Install project dependencies from requirements.txt

The repo includes a `requirements.txt` that pins the exact compatible versions of dbt and duckdb:

```powershell
pip install -r requirements.txt
```

#### Step A.3 — Open the folder in VS Code

```powershell
code .
```

This opens the project in VS Code. You're ready to run dbt commands — skip ahead to [Section 7: Run the Full Pipeline](#7-run-the-full-pipeline).

---

### Option B: Build from Scratch with `dbt init`

This approach shows how to create a brand new dbt project and wire it up to DuckDB manually. Great for teaching viewers the setup process from zero.

#### Step B.1 — Initialise a new dbt project

```powershell
dbt init jaffle_shop
```

dbt will ask you a series of questions:

1. **Which database adapter?** — type the number for `duckdb`
2. That's it for DuckDB — unlike cloud databases, there's no host, port, or credentials needed

This creates a new folder called `jaffle_shop/` with the standard dbt project scaffold.

#### Step B.2 — Navigate into the project

```powershell
cd jaffle_shop
```

#### Step B.3 — Configure `profiles.yml`

By default, dbt looks for `profiles.yml` in `~/.dbt/` (i.e. `C:\Users\YourName\.dbt\profiles.yml`). Open that file and confirm it looks like this — or create it if it doesn't exist:

```yaml
jaffle_shop:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: "jaffle_shop.duckdb" # DuckDB file created in your project root
      threads: 4
      schema: main
```

> **What `path` means:** This is where DuckDB will create (or look for) the `.duckdb` database file. The path is relative to your `profiles.yml` location. If the file doesn't exist yet, DuckDB will create it automatically on first run.

> **Threads:** Setting `threads: 4` is generally optimal for DuckDB. DuckDB parallelises internally, so more threads don't always mean faster results.

Alternatively, you can place `profiles.yml` directly in your project root folder (alongside `dbt_project.yml`). This is useful for self-contained demo projects and means the connection config travels with your project.

#### Step B.4 — Add Jaffle Shop seed data (CSV files)

Create a `seeds/` folder in your project and add the three raw CSV files. You can download them from the official repo:

- `seeds/raw_customers.csv`
- `seeds/raw_orders.csv`
- `seeds/raw_payments.csv`

The CSVs are available at:

```
https://github.com/dbt-labs/jaffle_shop_duckdb/tree/duckdb/seeds
```

#### Step B.5 — Verify the connection

```powershell
dbt debug
```

All checks should pass with green `OK` messages. If you see an error about the profile name, make sure the profile name in `profiles.yml` exactly matches the `profile:` field in `dbt_project.yml`.

---

## 5. Install VS Code Extensions

These extensions dramatically improve the dbt development experience in VS Code:

### dbt Power User _(Most important)_

- **Extension ID:** `innoverio.vscode-dbt-power-user`
- Install from the Extensions panel (`Ctrl+Shift+X`), search "dbt Power User"
- **What it gives you:**
  - Run `dbt run` / `dbt test` on the currently open file with a single click or keyboard shortcut
  - Inline lineage graph showing upstream/downstream models
  - Column-level preview of model output without leaving VS Code
  - Auto-completion for `{{ ref('...') }}` and `{{ source('...', '...') }}` calls
  - Syntax highlighting for Jinja + SQL in `.sql` files

### YAML _(by Red Hat)_

- **Extension ID:** `redhat.vscode-yaml`
- Provides schema validation and auto-complete for `.yml` files, including dbt schema files

### SQLTools + SQLTools DuckDB Driver _(Optional but useful)_

- **Extension ID:** `mtxr.sqltools` + `evidence-dev.sqltools-duckdb-driver`
- Lets you connect to your `.duckdb` file and run queries directly inside VS Code
- Great for browsing tables after `dbt run` without needing a separate DuckDB client

### GitLens _(Optional)_

- **Extension ID:** `eamodio.gitlens`
- Enhances Git tracking inside VS Code — useful if you're demonstrating version control of dbt projects

---

## 6. Understand the Project Structure

After cloning or initialising, your project should look like this:

```
jaffle_shop_duckdb/
│
├── dbt_project.yml          # Main project config — name, profile, model paths, materializations
├── profiles.yml             # Database connection (DuckDB path, schema, threads)
├── packages.yml             # External dbt packages to install (optional)
├── requirements.txt         # Python dependencies (dbt-core, dbt-duckdb, etc.)
├── jaffle_shop.duckdb       # The DuckDB database file (created after first dbt run)
│
├── seeds/                   # CSV files loaded as raw tables into DuckDB
│   ├── raw_customers.csv
│   ├── raw_orders.csv
│   └── raw_payments.csv
│
├── models/
│   ├── staging/             # Staging layer — light cleaning of raw data
│   │   ├── schema.yml       # Column descriptions + tests for staging models
│   │   ├── stg_customers.sql
│   │   ├── stg_orders.sql
│   │   └── stg_payments.sql
│   │
│   └── marts/               # Mart layer — business-ready aggregated tables
│       ├── schema.yml       # Column descriptions + tests for mart models
│       ├── customers.sql    # One row per customer with lifetime order stats
│       └── orders.sql       # One row per order with payment info joined in
│
├── tests/                   # Optional standalone SQL tests
├── macros/                  # Optional reusable Jinja macros
├── snapshots/               # Optional SCD2 snapshot definitions
│
└── target/                  # Auto-generated — compiled SQL, docs, artifacts
    ├── compiled/            # Rendered SQL (Jinja resolved) — read-only reference
    ├── run/                 # SQL that was actually executed
    └── manifest.json        # Full project graph (used by docs and other tools)
```

### Key files explained

**`dbt_project.yml`** — The heart of the project. Defines the project name, which profile to use, where models live, and default materializations per folder:

```yaml
name: "jaffle_shop"
version: "1.0.0"
profile: "jaffle_shop"

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]

models:
  jaffle_shop:
    staging:
      +materialized: view # Staging models are views (fast, no storage cost)
    marts:
      +materialized: table # Mart models are tables (pre-computed for performance)
```

**`profiles.yml`** — Connection config. For DuckDB this is minimal — just a file path:

```yaml
jaffle_shop:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: "jaffle_shop.duckdb"
      threads: 4
      schema: main
```

**`schema.yml` files** — Define documentation and tests for your models:

```yaml
version: 2

models:
  - name: stg_customers
    description: "Staged customer data — cleaned from raw_customers seed"
    columns:
      - name: customer_id
        description: "Primary key for customers"
        tests:
          - unique
          - not_null
```

---

## 7. Run the Full Pipeline

Now for the fun part. Make sure your virtual environment is active (`(venv)` shows in the terminal prompt) and you are inside the project directory.

### Step 7.1 — Install any dbt packages

If the project has a `packages.yml`, install them first:

```powershell
dbt deps
```

### Step 7.2 — Verify the connection

```powershell
dbt debug
```

All checks must be green before proceeding. Common issues:

- Wrong profile name — check it matches in both `profiles.yml` and `dbt_project.yml`
- Python virtual environment not activated — the `dbt` command won't be found

### Step 7.3 — Load the seed data

This reads the CSV files from `seeds/` and creates raw tables in DuckDB:

```powershell
dbt seed
```

**Expected output:**

```
1 of 3 START seed file main.raw_customers ........... [RUN]
1 of 3 OK loaded seed file main.raw_customers ....... [INSERT 100 in 0.10s]
2 of 3 START seed file main.raw_orders .............. [RUN]
2 of 3 OK loaded seed file main.raw_orders .......... [INSERT 99 in 0.05s]
3 of 3 START seed file main.raw_payments ............ [RUN]
3 of 3 OK loaded seed file main.raw_payments ........ [INSERT 113 in 0.05s]
```

### Step 7.4 — Run the models

This builds all SQL models in dependency order — staging first, then marts:

```powershell
dbt run
```

**Expected output:**

```
1 of 5 START sql view model main.stg_customers ...... [RUN]
1 of 5 OK created sql view model main.stg_customers . [OK in 0.08s]
2 of 5 START sql view model main.stg_orders ......... [RUN]
2 of 5 OK created sql view model main.stg_orders .... [OK in 0.06s]
3 of 5 START sql view model main.stg_payments ....... [RUN]
3 of 5 OK created sql view model main.stg_payments .. [OK in 0.05s]
4 of 5 START sql table model main.customers ......... [RUN]
4 of 5 OK created sql table model main.customers .... [OK in 0.15s]
5 of 5 START sql table model main.orders ............ [RUN]
5 of 5 OK created sql table model main.orders ....... [OK in 0.12s]
```

### Step 7.5 — Run the tests

This validates data quality across all models:

```powershell
dbt test
```

**Expected output:**

```
20 of 20 PASS ...
Completed successfully
Done. PASS=20 WARN=0 ERROR=0 SKIP=0 TOTAL=20
```

### Step 7.6 — Run everything at once with dbt build

For production-style runs, `dbt build` combines seed + run + test in one command, stopping if any test fails:

```powershell
dbt build
```

This is the command you'll use most in real projects — it's safer because tests act as gates between models.

---

## 8. Explore Your Data

### Using dbt show (quick terminal preview)

Preview rows from any model directly in the terminal:

```powershell
# See the first 5 rows of the customers mart
dbt show --select customers

# See 10 rows of raw orders
dbt show --select raw_orders --limit 10

# Preview a staging model
dbt show --select stg_payments
```

### Using SQLTools in VS Code (full SQL browser)

If you installed the SQLTools + DuckDB driver extensions:

1. Open the SQLTools panel (database icon in the left sidebar)
2. Add a new connection → choose DuckDB
3. Set the database path to your `jaffle_shop.duckdb` file
4. Browse tables and run queries directly in VS Code

### Using the DuckDB CLI (optional)

You can also query the `.duckdb` file directly from the terminal:

```powershell
# Install DuckDB CLI (if not already available)
pip install duckdb

# Open the database
duckdb jaffle_shop.duckdb

# Then run SQL:
SELECT * FROM customers LIMIT 5;
SHOW TABLES;
.quit
```

---

## 9. Generate & View Documentation

One of dbt's most impressive features for a YouTube demo — a fully auto-generated documentation website with an interactive lineage graph.

### Step 9.1 — Generate the docs

```powershell
dbt docs generate
```

This reads your project files and creates `target/manifest.json` and `target/catalog.json` — the data that powers the docs site.

### Step 9.2 — Serve the docs locally

```powershell
dbt docs serve
```

This starts a local web server and automatically opens `http://localhost:8080` in your browser.

> If port 8080 is already in use: `dbt docs serve --port 8001`

### What to explore in the docs UI

- **Project overview** — all models, seeds, tests listed in the left sidebar
- **Model detail page** — click any model to see its description, columns, tests, and the SQL that was compiled
- **Lineage graph** — click the graph icon (bottom right of any model page) to see the interactive DAG. You can:
  - See the full dependency chain from raw seeds → staging → marts
  - Click nodes to navigate to that model
  - Filter the graph to show only upstream or downstream nodes

The lineage graph for Jaffle Shop looks like this:

```
raw_customers ──┐
raw_orders ─────┤──► stg_customers ──┐
raw_payments ───┘   stg_orders ──────┤──► customers
                    stg_payments ────┘──► orders
```

---

## 10. Common Errors & Fixes

### `dbt: command not found` / `dbt is not recognized`

**Cause:** The virtual environment is not activated, so the `dbt` binary isn't on your PATH.

**Fix:**

```powershell
# Activate the virtual environment first
venv\Scripts\activate.bat      # Command Prompt
venv\Scripts\Activate.ps1      # PowerShell
```

---

### `Could not find profile named 'jaffle_shop'`

**Cause:** The profile name in `dbt_project.yml` doesn't match the top-level key in `profiles.yml`.

**Fix:** Open both files and make sure they match exactly:

```yaml
# dbt_project.yml
profile: 'jaffle_shop'   # ← must match ↓

# profiles.yml
jaffle_shop:             # ← must match ↑
  target: dev
  ...
```

---

### `IO Error: Could not set lock on file "jaffle_shop.duckdb"`

**Cause:** Another process has the DuckDB file open and is holding a write lock. DuckDB only allows one writer at a time.

**Fix:**

- If you have DBeaver open, close it completely (not just disconnect — fully shut down the app)
- If VS Code's SQLTools is connected, disconnect it before running dbt commands
- Check for any other terminal sessions that might have a DuckDB connection open
- As a last resort (you'll lose your data): delete the `.duckdb` file and re-run `dbt build`

---

### `PowerShell: running scripts is disabled on this system`

**Cause:** Windows PowerShell's default execution policy blocks running `.ps1` scripts.

**Fix:** Run this once in PowerShell as your user:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Then try activating the venv again.

---

### Tests fail with `not_null` or `unique` errors

**Cause:** The seed data or a model transformation has introduced unexpected nulls or duplicate values.

**Fix:** Use `dbt show` to inspect the data and `dbt compile --select my_model` to review the compiled SQL and find where the issue was introduced.

---

### `ModuleNotFoundError` when running dbt

**Cause:** dbt or one of its dependencies isn't installed in the active environment.

**Fix:** Make sure your venv is activated and reinstall:

```powershell
venv\Scripts\activate.bat
pip install dbt-core dbt-duckdb
```

---

## 11. What to Show on Your YouTube Tutorial

Here's a suggested filming flow for a clear, engaging tutorial:

### Opening sequence (show this first)

```powershell
dbt --version           # Prove the tools are installed
dbt debug               # Show a clean "All checks passed!" screen
```

### The core demo loop

```powershell
dbt seed                # "This loads our raw CSV data into DuckDB"
dbt run                 # "This builds our transformation models"
dbt test                # "This validates the data quality"
```

### The wow moments (great for screen recording)

```powershell
# Show compiled SQL — reveal the magic behind ref()
dbt compile --select customers

# Open target/compiled/.../customers.sql in VS Code to show the rendered SQL

# Quick data preview without leaving the terminal
dbt show --select customers
dbt show --select orders --limit 10
```

### The documentation reveal

```powershell
dbt docs generate
dbt docs serve
# → Switch to browser, show the lineage graph, click through models
```

### The "one command to rule them all"

```powershell
dbt build               # Show that this runs everything with testing built in
```

### Useful selector examples to demonstrate

```powershell
dbt ls                                    # List all project resources
dbt ls --select +customers                # Show upstream dependencies
dbt run --select staging.*               # Run only staging models
dbt test --select customers              # Test just one model
dbt run --select customers --full-refresh # Force rebuild a table
```

---

_Setup guide for dbt-duckdb on Windows 11 — dbt Core 1.8+ / dbt-duckdb 1.8+_
_Jaffle Shop DuckDB repo: https://github.com/dbt-labs/jaffle_shop_duckdb_
_Official dbt DuckDB quickstart: https://docs.getdbt.com/guides/duckdb_
