# Phase 2 — E/L Script & Source Registration: Validation

Run from the repo root, with a sample export placed in `data/incoming/`. All checks must pass for Phase 2 to be done. `<<...>>` marks the placeholders from `requirements.md`.

| # | Check | Command | Expected |
|---|-------|---------|----------|
| V1 | raw dirs ignored | `git check-ignore data/incoming data/raw` | both listed as ignored |
| V2 | extraction runs green | `make extract` (i.e. `duckdb -f scripts/extract.sql`) | exit 0 |
| V3 | latest export picked | put two exports with different filename timestamps into `data/incoming/`; run `make extract` | only the newer export's month is (re)written |
| V4 | month-partitioned outputs | `ls data/raw/` | `<<TABLE_1>>_YYYY-MM.parquet`, `<<TABLE_2>>_YYYY-MM.parquet`, `<<TABLE_3>>_YYYY-MM.parquet` for the export's month (`YYYY-MM` from the parsed filename timestamp) |
| V5 | junk row skipped per sheet | `duckdb -c "SELECT count(*) FROM read_parquet('data/raw/<<TABLE_1>>_*.parquet')"` | row count = sheet data rows (one junk row and the header excluded) |
| V6 | declared schema matches file schema | compare `models/sources.yml` columns/types against `parquet_schema('data/raw/<<TABLE_1>>_*.parquet')` (and the other two tables) | identical names and types |
| V7 | dbt parses the sources | `.venv/bin/dbt parse` | exit 0; 3 external sources found |
| V8 | glob reads all months | place two different months in `data/raw/`; `dbt build` / source query | source row count = sum over both month files |
| V9 | row-count sanity per source | source count vs `SELECT count(*) FROM read_parquet('data/raw/<table>_*.parquet')` | equal |
| V10 | idempotency | run `make extract` twice on the same export; diff `ls data/raw/` and row counts before/after | identical file set and counts — no duplicated rows |
| V11 | full monthly pipeline green | `make monthly` | extract and `dbt build` both exit 0 |
| V12 | fail-fast | `make extract` with an empty `data/incoming/` (or a corrupt xlsx) | non-zero exit, clear error |
| V13 | git | `git status --short` | only intended files changed; `data/` untracked-and-ignored |
