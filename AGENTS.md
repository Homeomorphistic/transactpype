## Project in one paragraph
dbt modeling layer that turns already-loaded family expenses data
(DuckDB) into a clean, tested analytical model. **Hard boundary:
dbt never extracts or loads.** The only E/L step is the thin DuckDB
SQL script `scripts/extract.sql` in this repo (monthly Excel export
in `data/incoming/` → month-partitioned Parquet in `data/raw/`);
dbt registers those files as external sources and only models.
Dashboards are a separate project — never do them here.

## Constitution (specs/) — read on trigger, not every task
- `specs/mission.md` — read before any scope decision: new layers,
  new dependencies, anything touching the E/L or dashboard boundary.
- `specs/tech-stack.md` — read before changing tools or versions.
- `specs/roadmap.md` — read before starting any new task; work only
  in the current phase.
- Per feature: `specs/<feature>/requirements.md`, `plan.md`,
  `validation.md` must exist and be approved before any code.

specs/ is authoritative, on conflict, update AGENTS.md to match specs/.