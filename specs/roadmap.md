# Roadmap

Phases are intentionally small — each one is a shippable slice of work, independently reviewable and testable.

---

## Phase 1 — Environment & Scaffold

- Python 3.11+ venv with `pip install "dbt==2.0.*"` (standard dbt v2 distribution; or Homebrew/curl installer)
- One-time setup: system DuckDB driver via `dbc` (required for Parquet sources in dbt v2)
- Initialize dbt project with the built-in DuckDB adapter; all models write into a single `transactpype.duckdb` file
- Git repo with initial commit
- First `dbt build` runs green on the empty project

## Phase 2 — Source Registration

- Register last month's Parquet file as a dbt external source
- Declare source columns and types explicitly (v2 static analysis does not infer Parquet schemas)
- Verify declared columns and types against the file's embedded Parquet schema
- Sanity check: query the source and confirm row count matches the upstream file
- No load step — dbt reads what the E/L project already produced

## Phase 3 — Staging: Clean Transactions

- `stg_transactions`: rename columns, cast types, parse dates and amounts (Polish formats)
- Handle messy free-text fields (trim, normalize payee/description)
- Built-in tests: `not_null` on key fields, `unique` on transaction ID
- Result: one month of clean, typed, tested transaction data

## Phase 4 — Categorization

- Import the existing category list as a dbt seed
- Rule-based mapping from payee/description to category
- Fallback "uncategorized" bucket for unmatched transactions
- Built-in test: `accepted_values` on category

## Phase 5 — Marts: Monthly Spending

- Dimensional models: `dim_category`, `dim_account`, `dim_month`
- `fct_monthly_spending`: monthly aggregation, built as an incremental model with month-based filter
- Built-in tests on all marts, including `relationships` from facts to dimension models
- First monthly spending report for last month — first look with the partner

## Phase 6 — Full History Backfill

- Load all 3 years of Parquet files from the upstream project
- Incremental run processes historical months without redesign
- Reconcile totals against source files (row counts, sum checks per month)
- Result: the trusted monthly spending report across all history

## Phase 7 — Documentation & Runbook

- Model and column descriptions; `dbt docs generate`
- Local column-level lineage: `dbt parse --generate-info-schema` (queryable `dbt.column_lineage`, no platform account)
- README with the monthly run procedure (`dbt build` after each monthly load)
- Final review with the partner: numbers never questioned

---

Later phases (not yet planned): dashboards on top of the DuckDB file, cashflow prediction, ML.