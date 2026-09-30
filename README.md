# data-pipeline-platform

Batch ETL pipelines that ingest public REST APIs into a PostgreSQL star schema.
Orchestrated with Airflow, run locally with Docker.

**Sources:** Open-Meteo (more planned)
**Stack:** Python 3.12 · Airflow · PostgreSQL · Docker · uv

> **Status: in progress.** See the [Roadmap](#roadmap) for what works today.

---

## Why this project exists

I'm a Java/Spring Boot backend developer moving into data engineering. I built this to practice the core of the job end to end: reliable ingestion, dimensional modeling, data quality checks, and scheduled orchestration. Every design decision is written down in [Design decisions](#design-decisions) so the reasoning is visible, not just the code.

## Architecture

```
            +-----------+     +-----------+     +-----------+     +------------+
 REST API-->|  Extract  |---->| Transform |---->|  Quality  |---->|    Load    |
            | (per src) |     | (per src) |     |  checks   |     | (shared)   |
            +-----+-----+     +-----------+     +-----------+     +------+-----+
                  |                                                      |
                  v                                                      v
          raw.<source>_responses                               PostgreSQL star schema
          (raw JSON, kept for replay)                          (fact + dimension tables)

            Airflow schedules one DAG per source, daily.
```

Only **extract** and **transform** differ per API. Quality checks, loading, and scheduling are shared and written once.

## Data model

<!-- Replace with your real schema once designed. Keep an ER diagram or table list here. -->

| Table | Type | Grain / purpose |
|---|---|---|
| `fact_<name>` | Fact | _one row per ... (define the grain)_ |
| `dim_<name>` | Dimension | _describes ..._ |
| `dim_date` | Dimension | Calendar attributes |

## Quickstart

**Requirements:** Docker, Docker Compose, [uv](https://docs.astral.sh/uv/)

```bash
git clone https://github.com/amiraliverse/data-pipeline-platform.git
cd data-pipeline-platform
cp .env.example .env        # then edit the values
docker compose up -d
```

- Airflow UI: http://localhost:8080
- PostgreSQL: `localhost:5432`

> Only keep this section if `docker compose up` really works from a fresh clone. Test it in an empty folder before you push.

**Local development (without Docker):**

```bash
uv sync
uv run pytest
uv run ruff check .
```

## Project structure

```
data-pipeline-platform/
  dags/                  # Airflow DAGs (one per source, generated from the registry)
  src/data_pipeline_platform/
    ingestion/           # one module per API
    transform/
    quality/             # data quality checks
    load/                # idempotent loading into PostgreSQL
  sql/                   # schema and star-schema DDL
  tests/
  docker-compose.yml
  pyproject.toml
  uv.lock
```

## Design decisions

- **Idempotent loads.** Re-running a day replaces that day's data instead of duplicating it, so retries and backfills are safe.
- **Incremental ingestion.** Each run fetches only the date it is responsible for.
- **Raw data is stored before transformation.** A transform bug can be fixed and replayed without calling the API again.
- **Star schema.** Chosen for analytical queries: facts hold measurements, dimensions hold descriptive attributes.
- **Quality checks run before load.** Nulls, duplicates, and row counts are validated so bad data never reaches the warehouse.
- **Airflow runs from its official image; my code is a separate uv project.** Airflow's pinned dependency tree conflicts with normal application dependencies, so they are kept apart.

## Adding a new source

1. Create `src/data_pipeline_platform/ingestion/<source>.py` with `extract` and `transform`.
2. Register it in the source registry.
3. Add its config (URL, parameters, schedule).

The DAG for the new source is generated automatically.

_(Keep this section only after the second source really works this way.)_

## Data quality checks

| Check | Rule |
|---|---|
| Not null | Key columns must be present |
| No duplicates | Grain columns are unique per load |
| Row count | Load must not be empty or drop sharply versus the previous run |

## Roadmap

- [ ] Ingestion from first API (Open-Meteo)
- [ ] Raw layer stored in PostgreSQL
- [ ] Star schema (facts + dimensions) with indexes
- [ ] Idempotent, incremental load
- [ ] Data quality checks
- [ ] Airflow DAG with daily schedule, retries
- [ ] Tests for transform and quality logic
- [ ] `docker compose up` works from a clean clone
- [ ] Second source added through the registry
- [ ] CI (lint + tests) on every push

Tick a box only when it is true in the repo.

## What I learned / trade-offs

<!-- Fill this in as you go. 3-5 honest bullets: what was hard, what you'd change, what you'd do at larger scale. Interviewers read this. -->

## Author

**Amirali Abbasi** · Java backend developer moving into data engineering
Tehran, Iran · [GitHub](https://github.com/amiraliverse) · [LinkedIn](https://linkedin.com/in/amirali-abbasi) · [Portfolio](https://amiraliabbasi.ir)