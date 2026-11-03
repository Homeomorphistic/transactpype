# Phase 1 — Environment & Scaffold: Validation

Run from the repo root. All checks must pass for Phase 1 to be done.

| # | Check | Command | Expected |
|---|-------|---------|----------|
| V1 | Python version | `python3 --version` | >= 3.11 |
| V2 | venv exists and dbt installed in it | `.venv/bin/dbt --version` | `dbt 2.0.x` |
| V3 | dbt 2.0.x resolves from the `dbt==2.0.*` pin | `.venv/bin/pip show dbt` | `Version: 2.0.x` |
| V4 | system DuckDB driver installed | `.venv/bin/dbc list` | `duckdb` driver listed |
| V5 | dbt project parses | `.venv/bin/dbt parse` | exit 0, project `transactpype` found |
| V6 | dbt can connect to the DuckDB file | `.venv/bin/dbt debug` | connection OK, adapter `duckdb` |
| V7 | empty project builds green | `.venv/bin/dbt build` | exit 0, 0 models, 0 tests, no errors |
| V8 | single-file warehouse created | `ls transactpype.duckdb` | file exists after build/debug |
| V9 | git | `git status --short` | clean tree after scaffold commit |
| V10 | artifacts ignored | `git check-ignore .venv transactpype.duckdb target logs dbt_packages` | all listed as ignored |
