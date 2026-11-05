# TransactPype Tech Stack

TransactPype's dbt layer is **transform-only**: dbt never extracts or loads. The single E/L step is a thin DuckDB SQL script (`scripts/extract.sql`) owned in this repo, which converts each monthly Excel export into month-partitioned Parquet files (see Source contract); TransactPype models that data into an analytical DuckDB database using dbt. There is no frontend here.

## Core
   Layer | Choice | Rationale |
 |---|---|---|
 | Transformation | dbt 2.0.x (v2 engine) | Latest major dbt — stricter parse-time correctness checks, native column-level lineage, Parquet artifacts; also a deliberate learning goal of this project |
 | Adapter | DuckDB adapter, **built into dbt v2** | dbt 2.0 is the first release that ships DuckDB support inside the engine itself — no separate adapter package to install |
 | Database | DuckDB (embedded, single file) | OLAP analytics on small data, zero infrastructure |
 | Source data | Month-partitioned **Parquet** in `data/raw/` from `scripts/extract.sql`, registered as dbt external sources via glob (`read_parquet('data/raw/{name}_*.parquet')`) | Clean upstream contract; schema embedded in the files; dbt reads them without a load step; re-running extraction is idempotent |
 | Model strategy | Incremental models | Deliberate dbt practice; matches monthly data arrivals |
 | Testing | dbt built-in generic tests only | `not_null`, `unique`, `accepted_values` are sufficient at this scale |
 | Scheduling | Manual `make monthly` (extract via DuckDB CLI + `dbt build`) | Monthly cadence, single user, one command |
 | Environment | dbt v2 CLI in a Python venv | Installed via `pip install "dbt==2.0.*"` (standard distribution) in a venv — no adapter package, system Python untouched |
 | Version control | Git | Every model change is a commit |

## Source contract (E/L → dbt)

The only E/L step is `scripts/extract.sql`, run with the DuckDB CLI (`make extract`):

- Picks the **latest** Excel export in `data/incoming/` by the timestamp parsed from the filename (`<<TIMESTAMP_FORMAT>>`).
- Reads the 3 workbook sheets (`<<SHEET_1_NAME>>`, `<<SHEET_2_NAME>>`, `<<SHEET_3_NAME>>`), skipping the one junk row at the top of each sheet: `read_xlsx(file, sheet = ..., range = 'A2:Z')` (force `header = true` if needed).
- Writes 3 **month-partitioned** Parquet files — `data/raw/<<TABLE_1>>_YYYY-MM.parquet`, `data/raw/<<TABLE_2>>_YYYY-MM.parquet`, `data/raw/<<TABLE_3>>_YYYY-MM.parquet` — via `COPY (...) TO '...' (FORMAT PARQUET)`. Each monthly export contains only that month's rows, so the month (`YYYY-MM`) comes from the export's parsed timestamp. Re-running extraction on the same export overwrites that month's file: idempotent, no duplicates.
- `data/incoming/` and `data/raw/` are gitignored — raw data is never committed.

dbt side (`models/sources.yml`):

- One external source per table with `external_location: "read_parquet('data/raw/{name}_*.parquet')"` — the glob reads **all months**.
- Columns and types are declared **explicitly** per source (v2 static analysis does not infer file schemas), verified against the Parquet embedded schema.

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

At ~1,000 rows per month with 5–8 mostly string columns, data volume is far below anything that needs partitioning (DuckDB has no partitioned tables, and performance is a non-issue at this scale). Raw files are month-partitioned **by filename** (`data/raw/<table>_YYYY-MM.parquet`) so re-runs are idempotent and months drop in one at a time; the DuckDB warehouse itself stays unpartitioned. "Monthly" remains a **run cadence**, not a table design:

- New source files arrive monthly; incremental models pick them up with month-based filters.
- Moving to weekly later means running `dbt build` more often — **no schema, storage, or modeling change**.

## Data

- **Sources**: month-partitioned Parquet files in `data/raw/` produced by `scripts/extract.sql`, declared in dbt as glob external sources (`read_parquet('data/raw/{name}_*.parquet')`), with columns and types declared explicitly (verifiable against each file's embedded schema). `data/incoming/` and `data/raw/` are gitignored.
- **Warehouse**: single DuckDB file (e.g. `transactpype.duckdb`); all dbt models materialize there.
- **Layers**: `staging` (cleaning/renaming/casting, `stg_*`) → `marts` (dimensional model: `fct_*` / `dim_*`, monthly aggregations).
- **Scale**: ~1,000 rows/month, 5–8 columns — mostly strings (category, note, label) plus date and amount.

## Testing

- Built-in generic tests only: `not_null`, `unique`, `accepted_values`, (and `relationships` if dimensional integrity needs it)
- Tests declared alongside the models in their YAML files

## Tooling

- dbt v2 CLI: `pip install "dbt==2.0.*"` (standard distribution) in a Python venv — no adapter package, no extra Python dependencies
- Git repository; `make monthly` (extract + `dbt build`) run manually each month
- `dbt parse --generate-info-schema` for queryable project metadata and local column-level lineage

## What We Are Not Using

- **No extraction or loading inside dbt** — the only E/L is the thin `scripts/extract.sql` in this repo (see Source contract); no heavier E/L tooling, no Python for E/L
- **No frontend / dashboard** — a separate project reads the DuckDB file
- **No orchestrator or scheduler** (Airflow, cron) and **no dbt Cloud** — manual monthly runs
- **No `dbt-duckdb` Python adapter** — superseded by the built-in v2 adapter (fallback option only if we hit a v2 feature gap)
- **No dbt-utils / dbt-expectations or custom test packages** — revisit if built-ins prove insufficient
- **No Docker** — a self-contained CLI binary is enough
- **No CSV as an interchange format** — Parquet only (CSV's readability isn't worth the type-inference risk)
- **No partitions / clustering** — data volume doesn't justify it