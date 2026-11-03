# Phase 1 — Environment & Scaffold: Plan

1. **Spec docs** — this file plus `requirements.md` and `validation.md` under `specs/phase-1-environment-scaffold/` (constitution: per-feature specs before code).
2. **Venv + dbt** — `python3 -m venv .venv`; `.venv/bin/pip install "dbt==2.0.*"`; verify `dbt --version` → 2.0.x. Python 3.14.4 on this machine satisfies the 3.11+ requirement.
3. **System DuckDB driver** — install `dbc` (ADBC driver manager) into the same venv as a setup tool; `dbc install duckdb`; verify with `dbc list`.
4. **Scaffold** — hand-written minimal project at repo root (the `dbt init` samples ship demo models, which violates "empty project"):
   - `dbt_project.yml` — project `transactpype`, profile `transactpype`, model-paths `models/`
   - `profiles.yml` — profile `transactpype`, output `dev`, `type: duckdb`, `path: transactpype.duckdb`
   - `models/` directory (empty placeholder)
   - `.gitignore` — `.venv/`, `target/`, `dbt_packages/`, `logs/`, `transactpype.duckdb`
5. **Commit** — scaffold + spec docs committed to git.
6. **Verify** — run `dbt build`; expect exit 0 with zero models.
