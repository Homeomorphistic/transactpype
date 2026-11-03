# TransactPype Tech Stack

TransactPype is a **transform-only project**: the upstream E/L project extracts data and produces monthly expense files; TransactPype models that data into an analytical DuckDB database using dbt. There is no extraction, loading, or frontend here.

## Core
   Layer | Choice | Rationale |
 |---|---|---|
 | Transformation | dbt 2.0.x (v2 engine) | Latest major dbt — stricter parse-time correctness checks, native column-level lineage, Parquet artifacts; also a deliberate learning goal of this project |
 | Adapter | DuckDB adapter, **built into dbt v2** | dbt 2.0 is the first release that ships DuckDB support inside the engine itself — no separate adapter package to install |
 | Database | DuckDB (embedded, single file) | OLAP analytics on small data, zero infrastructure |
 | Source data | Monthly **Parquet** files from the E/L project, registered as dbt external sources | Clean upstream contract; schema embedded in the files; dbt reads them without a load step |
 | Model strategy | Incremental models | Deliberate dbt practice; matches monthly data arrivals |
 | Testing | dbt built-in generic tests only | `not_null`, `unique`, `accepted_values` are sufficient at this scale |
 | Scheduling | Manual `dbt build` after each monthly load | Monthly cadence, single user |
 | Environment | dbt v2 CLI, self-contained binary | Installed via pip (`dbt-oss`) in a venv, or the standalone installer — no Python dependencies to manage |
 | Version control | Git | Every model change is a commit |

## Recommended: dbt v2 + DuckDB

DuckDB over Postgres / SQLite / a cloud warehouse:

- **Embedded and file-based** — no server to run or pay for; the single `transactpype.duckdb` file *is* the deliverable
- **Columnar OLAP** — monthly aggregations are instant on this data
- **The v2 engine reads external files as sources**, so "model what's already there" is literal: source files stay untouched, all dbt models write into the DuckDB file
- **Downstream projects attach to the same DuckDB file** (dashboards, future cashflow prediction) without any API layer

dbt v2 over dbt v1:

- DuckDB support is built into the engine (in v1 it required the separate `dbt-duckdb` Python package)
- Native SQL parsing catches invalid column references and type mismatches **before anything reaches the database**
- Column-level lineage works locally — `dbt compile --generate-info-schema --static-analysis strict` writes a queryable `dbt.column_lineage` file, no platform account needed

### Parquet over CSV for source files

- **Schema is embedded in the file** — types survive the E/L → TransactPype handoff: dates stay dates, amounts stay decimals, strings stay strings. No inference ambiguity, so Polish-format pitfalls (comma decimals like `1 234,56`, `DD.MM.YYYY` dates) cannot corrupt staging.
- **Deterministic parsing of free text** — category/note/label columns contain arbitrary text; Parquet has no delimiter, escaping, or encoding edge cases.
- **The data-engineering standard** — this is a learning project, so the habits should transfer to every future project (including cashflow prediction downstream).
- **Trade-off accepted:** files are binary and not eyeball-readable (query them with DuckDB instead), and Parquet needs a system DuckDB driver (one-time setup, below).

### Known v2 caveats for our setup (and how we handle them)

- **Flat-file schema inference is a named gap**: v2's static analysis may fail to infer schemas for CSV/Parquet/JSON sources. → We declare source columns and types **explicitly** in our source definitions; with Parquet the declarations are verifiable against the file's embedded schema.
- **The bundled DuckDB driver cannot load extensions** (e.g. `parquet`). → One-time setup task: install a system DuckDB driver with `dbc`; dbt v2 prefers a system-installed driver over the bundled one.

### Monthly cadence — no partitions needed

At ~1,000 rows per month with 5–8 mostly string columns, data volume is far below anything that needs partitioning (DuckDB has no partitioned tables, and performance is a non-issue at this scale). "Monthly" is a **run cadence**, not a storage design:

- New source files arrive monthly; incremental models pick them up with month-based filters.
- Moving to weekly later means running `dbt build` more often — **no schema, storage, or modeling change**.

## Data

- **Sources**: monthly Parquet files exported by the upstream E/L project, declared in dbt as external sources, with columns and types declared explicitly (verifiable against each file's embedded schema).
- **Warehouse**: single DuckDB file (e.g. `transactpype.duckdb`); all dbt models materialize there.
- **Layers**: `staging` (cleaning/renaming/casting, `stg_*`) → `marts` (dimensional model: `fct_*` / `dim_*`, monthly aggregations).
- **Scale**: ~1,000 rows/month, 5–8 columns — mostly strings (category, note, label) plus date and amount.

## Testing

- Built-in generic tests only: `not_null`, `unique`, `accepted_values`, (and `relationships` if dimensional integrity needs it)
- Tests declared alongside the models in their YAML files

## Tooling

- dbt v2 CLI: `pip install "dbt-oss==2.0.*"` in a Python venv (open-source distribution), or the standalone binary installer — no adapter package, no Python dependencies
- Git repository; `dbt build` run manually after each monthly load
- `dbt parse --generate-info-schema` for queryable project metadata and local column-level lineage

## What We Are Not Using

- **No extraction or loading** — the upstream E/L project owns that boundary
- **No frontend / dashboard** — a separate project reads the DuckDB file
- **No orchestrator or scheduler** (Airflow, cron) and **no dbt Cloud** — manual monthly runs
- **No `dbt-duckdb` Python adapter** — superseded by the built-in v2 adapter (fallback option only if we hit a v2 feature gap)
- **No dbt-utils / dbt-expectations or custom test packages** — revisit if built-ins prove insufficient
- **No Docker** — a self-contained CLI binary is enough
- **No CSV as an interchange format** — Parquet only (CSV's readability isn't worth the type-inference risk)
- **No partitions / clustering** — data volume doesn't justify it