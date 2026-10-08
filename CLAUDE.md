---
created: 2026-10-08T21:48
updated: 2026-10-08T22:27
---
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Learning pet project for a Claude Code course: a config-driven Python ETL that ingests pharma distributors' Excel files (sell-in and stock), cleans and maps them in a DuckLake "workshop", publishes fully clean facts and master data to a StarRocks "showcase", and serves self-service dashboards from it.

- [README.md](README.md) (Ukrainian) — project goal, target architecture, roles, roadmap.
- [docs/speckit-prompts.md](docs/speckit-prompts.md) — target architecture, constitution text, roadmap (5 stages, features 001–018), the `/speckit.specify` prompt for feature 001. Until `.specify/memory/constitution.md` exists, treat this file as the source of the project principles.

## Data privacy — read before touching data

The project uses **real distributor files**, so:

- Real files live only in `data/private/` (git-ignored). **Never open, read, print or sample anything in `data/private/`** — whatever Claude reads leaves the machine. The user runs scripts against it themselves.
- Everything else works on anonymized copies in `data/samples/`, produced by `scripts/anonymize.py` (feature 001). Anonymization replaces distributor/client/address/product names with stable pseudonyms (same real value → same pseudonym across all files), distorts quantities while keeping integers, signs and total rows consistent, and preserves file structure exactly (sheets, headers, junk rows, totals, historical versions, region-name language).
- The pseudonym mapping and distortion coefficients stay in `data/private/`.
- Before any commit, check that no real data or real names are staged. Region names are kept as-is (public, needed for the RU→UA drift case).

## Architecture (planned)

Two stores with different roles:

- **Workshop — DuckLake.** Files are ingested, normalized and mapped here. Everything dirty or disputed stays here.
- **Showcase — StarRocks.** Star schema of facts + master data (products/SKU, regions, distributors, clients, calendar). A distributor period is published only after its mapped share of volume reaches the readiness threshold and a data steward confirms it; unmapped remainder never goes there. Publishing is idempotent per distributor + period (Parquet → StarRocks `FILES()`). Business rollups for dashboards live here, not in the workshop.
- **Around them:** a Streamlit mapping UI where the data steward maps `unmapped` values to dictionaries, and Apache Superset on top of StarRocks. Postgres, the mapping UI, StarRocks and Superset run in `docker compose`; config in git, data in volumes or git-ignored folders.

Roadmap stages: 1 — ETL into the workshop (001–009); 2 — master data, mapping UI, Postgres catalog (010–012); 3 — StarRocks showcase and publish (013–014); 4 — Superset (015–016); 5 — whole stack in one `docker compose up`, optional orchestrator (017–018).

ETL pipeline: `data/inbox/` → detect distributor by file-name pattern → find sheet by required columns (not by name) → find header row by max matches in first N rows → map columns via RU/UA synonym lists (case-, whitespace- and punctuation-insensitive) → normalize types and pack quantities → map regions and products through alias dictionaries → validate → load into DuckLake → report and move the source file to `data/archive/`.

Invariants that shape the code:

- **Config, not code.** A new distributor is added only with a YAML file in `configs/distributors/` (pydantic-validated). Code must not contain distributor, sheet or header names.
- **Column rules.** Missing required column → stop with a clear error; missing optional → warning; extra column → log only.
- **Nothing is dropped silently.** Unknown regions/products/clients go to the `unmapped` table; rejected rows are counted in `load_log`.
- **Drift tracking.** Each load stores a structure fingerprint (sheet names + headers) in `structure_snapshots` and reports differences from the previous load of the same distributor.
- **Idempotent loads.** Reloading the same distributor + period replaces data in one transaction; each load is a DuckLake snapshot.

Workshop layers:

| Layer | Content | Location |
|---|---|---|
| `raw` | each input file unchanged, one Parquet per file, append-only; `normalized` must be rebuildable from it | `data/lake/raw/` |
| `normalized` | `fact_sell_in`, `fact_stock`, `dim_region`/`region_aliases`, `dim_product`/`product_mapping`, `unmapped`, `load_log`, `structure_snapshots` | DuckLake schema `normalized` |
| `aggregates` | service aggregates for mapping, not reporting: unmapped values ranked by volume, mapped share of volume per distributor + period, period readiness (feature 009) | DuckLake schema `aggregates` |

DuckLake catalog: stage 1 — file `data/lake/catalog.ducklake`; from stage 2 — Postgres in Docker (a file catalog allows a single writer, while ETL and the mapping UI write concurrently). Table data as Parquet in `data/lake/tables/`. All of `data/` except `data/samples/` is local-only.

Stack: Python 3.12 managed by **uv** (system Python is 3.9 — always use `uv run` / `uv add`, never bare `python`/`pip`), Polars, fastexcel (calamine) to read Excel, openpyxl to write anonymized copies, pydantic v2, DuckDB + `ducklake` extension, typer, pytest, ruff. Stages 2–5 add Docker Desktop + docker compose, Postgres, Streamlit, StarRocks, Apache Superset.

## Workflow

- Development follows spec-kit: `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement`. Specs live in `specs/NNN-*/`.
- One feature = one spec-kit branch = one Pull Request (one course homework). The first submission goes to `main`.
- Each feature includes a learning note `docs/learning/NNN-*.md`: which features of the stack's tools were used and why, with a small runnable example. Prefer the clear solution over the clever one.
- The user is learning these technologies: talk to them in Russian.
