# Phase 2 — E/L Script & Source Registration: Plan

1. **Constitution amendment** (committed first, before any Phase 2 code): `specs/mission.md` and `AGENTS.md` (boundary: dbt never extracts; thin `scripts/extract.sql` in this repo is the only E/L step), `specs/tech-stack.md` (source contract + the two stale `dbt-oss` references fixed to `pip install "dbt==2.0.*"`), `specs/roadmap.md` (Phase 2 = E/L script + source registration as one phase).
2. **This spec** — `requirements.md`, `plan.md`, `validation.md` under `specs/phase-2-el-and-source-registration/`; approved by the user before any code.
3. **Placeholders filled by the user** (sheet names, filename timestamp format, table names, columns/types from a sample export). Until filled, every placeholder stays marked `<<...>>` in the specs and in `extract.sql` — never hardcoded from guesses.
4. **Implementation, committed in small slices:**
   a. `.gitignore` += `data/incoming/`, `data/raw/`
   b. `scripts/extract.sql` — placeholder-marked: latest-xlsx-by-parsed-timestamp selection from `data/incoming/`; 3 × `read_xlsx(..., sheet = ..., range = 'A2:Z')` (junk row skipped); 3 × `COPY (...) TO 'data/raw/<table>_YYYY-MM.parquet' (FORMAT PARQUET)` with the month derived from the parsed timestamp
   c. `Makefile` — `extract` (`duckdb -f scripts/extract.sql`), `monthly` (extract + `dbt build`), fail-fast
   d. `models/sources.yml` — glob `external_location` per table + explicit columns/types
5. **Verify** per `validation.md` (V1–Vn), including the idempotency check.
