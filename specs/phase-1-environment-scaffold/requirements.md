# Phase 1 — Environment & Scaffold: Requirements

Source: `specs/roadmap.md`, Phase 1. Boundary rules from `specs/mission.md` and `specs/tech-stack.md` apply (transform-only; no E/L, no dashboards).

## R1 — Python environment
- Python 3.11+ virtual environment, created inside the repo (`.venv/`), not system Python.
- `pip install "dbt==2.0.*"` (standard dbt v2 distribution) inside that venv. The `dbt` CLI must report version 2.0.x.

## R2 — System DuckDB driver (one-time)
- Install a system DuckDB driver via `dbc` (ADBC driver manager), because the driver bundled with dbt v2 cannot load extensions such as `parquet` (see `specs/tech-stack.md`, "Known v2 caveats").
- `dbc install duckdb` must succeed and `dbc list` must show the duckdb driver.

## R3 — dbt project scaffold
- dbt project initialized at the repo root with the built-in DuckDB adapter (no separate adapter package).
- All models write into a single DuckDB file: `transactpype.duckdb` (repo root, git-ignored).
- Project is an *empty* project at this phase: no sources, no models yet.

## R4 — Version control
- Git repo present with the scaffold committed. (Repo and initial commit already exist; the scaffold is committed as a new commit.)

## R5 — Green build
- `dbt build` exits 0 on the empty project.

## Out of scope (later phases)
- Registering Parquet sources (Phase 2), staging models (Phase 3+), seeds, marts, docs.
