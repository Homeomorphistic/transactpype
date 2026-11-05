# Phase 2 — E/L Script & Source Registration: Requirements

Source: `specs/roadmap.md`, Phase 2. The constitution amendment (same change series) establishes the boundary: **dbt never extracts or loads**; the thin DuckDB SQL script `scripts/extract.sql` in this repo is the only E/L step. Hard rules: `pip install "dbt==2.0.*"` standard distribution only (never `dbt-oss`); DuckDB adapter is built into dbt v2; built-in generic tests only; no Docker, no orchestrator, no scheduler, no Python for E/L.

## R1 — Extraction script (`scripts/extract.sql`)

- Run with the DuckDB CLI from the repo root: `duckdb -f scripts/extract.sql`. Any error → non-zero exit (fail fast).
- Picks the **latest** `*.xlsx` in `data/incoming/` by the timestamp parsed from the filename. Filename timestamp format: `<<TIMESTAMP_FORMAT>>` (e.g. `YYYY-MM-DD`, `YYYYMMDD_HHMMSS`).
- Reads the 3 workbook sheets `<<SHEET_1_NAME>>`, `<<SHEET_2_NAME>>`, `<<SHEET_3_NAME>>`. Every sheet starts with one junk row → skip it via `read_xlsx(file, sheet = ..., range = 'A2:Z')` (header is the first row within the range; force `header = true` if needed).
- Writes each sheet to a month-partitioned Parquet file via `COPY (...) TO '...' (FORMAT PARQUET)`:
  - `data/raw/<<TABLE_1>>_YYYY-MM.parquet`
  - `data/raw/<<TABLE_2>>_YYYY-MM.parquet`
  - `data/raw/<<TABLE_3>>_YYYY-MM.parquet`
- Each monthly export contains **only that month's rows**; the month (`YYYY-MM`) is derived from the export's parsed filename timestamp.
- **Idempotent:** re-running extraction on the same export overwrites that month's file — no duplicates, no extra files.
- `data/incoming/` and `data/raw/` are gitignored.

## R2 — Makefile

- `make extract` → runs the DuckDB CLI on `scripts/extract.sql`; non-zero exit propagates (fail fast).
- `make monthly` → `extract` then `dbt build` (venv dbt); non-zero exit propagates.
- No scheduler, no orchestrator, no Docker.

## R3 — Source registration (`models/sources.yml`)

- One external source per table, with `external_location: "read_parquet('data/raw/{name}_*.parquet')"` — the glob reads **all months**.
- Explicit column names and types for every source table (dbt v2 static analysis does not infer file schemas). Declarations: `<<TABLE_1_COLUMNS>>`, `<<TABLE_2_COLUMNS>>`, `<<TABLE_3_COLUMNS>>` — derived from a sample export and verified against the Parquet embedded schema.

## R4 — Sanity checks

- Row-count check per source vs its month file(s).
- Idempotency check: re-run extraction on the same export → identical file set and row counts (no duplicated rows).

## Out of scope (Phase 3+)

- `staging` (`stg_*`) and `marts` models, seeds, docs — not this phase.

## Placeholders to fill (owner: user — never guessed)

- `<<SHEET_1_NAME>>`, `<<SHEET_2_NAME>>`, `<<SHEET_3_NAME>>` — exact sheet names in the workbook
- `<<TIMESTAMP_FORMAT>>` — timestamp format in the export filename
- `<<TABLE_1>>`, `<<TABLE_2>>`, `<<TABLE_3>>` — target Parquet/source table names (proposed defaults pending confirmation of sheet contents: `transactions`, `accounts`, `categories`)
- `<<TABLE_1_COLUMNS>>`, `<<TABLE_2_COLUMNS>>`, `<<TABLE_3_COLUMNS>>` — column names and types per table, derived from a sample export the user provides
