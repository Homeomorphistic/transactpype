# Mission
Raw bank transaction data answers no questions on its own. TransactPype turns the family's unshaped expenses data into a source of truth the whole family can trust.

## What We Do
Transform raw, already-loaded data into a clean, tested analytical model using dbt. The boundary is explicit: **dbt never extracts or loads** — the only E/L step is a thin DuckDB SQL script (`scripts/extract.sql`) owned in this repo, which turns each monthly Excel export into month-partitioned Parquet under `data/raw/`; dbt registers those files as external sources and only models from them. Future projects (dashboards, ML) build on top.

## Who We Serve
Bartosz (builder/analyst), partner & family (budget decisions), and future projects (cashflow prediction, ML) as downstream consumers.

## Target Audience
Family budget talks as the primary use case, plus Bartosz-as-learner: a deliberate practice project for dbt and SDD.

## What Success Looks Like
A monthly spending report both partners trust enough for real decisions; numbers never questioned. Plus: a backbone solid enough that cashflow prediction later is a new feature, not a rewrite.

