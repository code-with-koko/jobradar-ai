# JobRadar AI

**European Tech Market Intelligence Platform**

JobRadar AI is a Python backend and data engineering project that will collect technology job postings from multiple sources, normalize them, and expose search, analytics, hiring trends, and candidate skill-gap insights through FastAPI.

The project has two goals: demonstrate the design and operation of an end-to-end data platform, and provide practical insight into European technology hiring, with Germany as the initial focus. Target role families include Python backend engineering, data engineering, and cloud engineering.

## Current progress

Status as of **2 October 2026**:

| Component | Status |
| --- | --- |
| GitHub repository | Created |
| Windows + Git Bash development environment | Set up |
| Python virtual environment | Created locally |
| Minimal FastAPI application | Created locally in `app/main.py` |
| `GET /health` and interactive `/docs` | Reported working locally |
| Application code published to GitHub | Pending at the time this README was added |
| PostgreSQL with Docker Compose | Configuration planned below; startup not yet confirmed |
| SQLAlchemy connection and Alembic migrations | Planned |
| Job validation, ingestion, storage, and retrieval | Planned |
| S3, Airflow, Kafka, PySpark, OpenSearch, and AWS deployment | Later milestones |

This README is the initial repository documentation. The folder tree, database setup, and later capabilities describe the target implementation; they are not all present or operational yet.

## What the platform will answer

- Which companies are hiring Python backend and data engineers?
- Which skills and experience levels appear most often in collected postings?
- Which countries and cities have the most relevant opportunities?
- What salaries and remote/hybrid arrangements are advertised?
- Which postings explicitly mention visa sponsorship?
- How does observed demand for skills such as Kafka, Spark, and Airflow change over time?
- How does a candidate's skill profile compare with requirements in relevant jobs?

The data model will support Germany, the Netherlands, Austria, Switzerland, Sweden, Ireland, and the UK. Collection will expand gradually.

Market statistics will describe the collected dataset, with its source coverage and observation period. Counts and percentages from early planning examples are illustrative, not actual findings. Trend analysis requires historical observations. Missing salary or sponsorship information must remain unknown rather than being inferred as absent.

## Architecture

### First milestone

```text
Local JSON fixture / one permitted source
                  |
                  v
fetch -> parse -> normalize -> validate -> persist
                                           |
                                           v
                                       PostgreSQL
                                           |
                                           v
                                        FastAPI
```

First, make one complete flow work with at least 20 fictional sample jobs. Repeat ingestion without duplicates, then retrieve stored jobs through the API.

### Target architecture

```text
Official APIs / public feeds / permitted career pages
                         |
                         v
           Python collectors (httpx / Scrapy)
                         |
                         v
                       Kafka
                         |
                         v
                S3 Bronze: raw payloads
                         |
                         v
               PySpark transformations
                         |
                         v
            S3 Silver: normalized job records
                         |
                         v
           S3 Gold: curated analytics datasets
                         |
                         v
             PostgreSQL / OpenSearch
                         |
                         v
                       FastAPI
                         |
                         v
           Search / analytics / dashboard
                         |
                         v
             CV-to-market skill-gap analysis

Airflow schedules and monitors ingestion and batch transformations.
An optional Kafka consumer can support real-time processing later.
```

Airflow is the orchestration layer; it schedules work rather than acting as a data store. Kafka, Spark, and other services will be introduced at their roadmap stages.

| Layer | Responsibility |
| --- | --- |
| Bronze | Preserve immutable source payloads and ingestion metadata in S3 for replay and auditability |
| Silver | Clean, normalize, deduplicate, and enrich jobs, locations, role categories, and skills |
| Gold | Produce salary, skill, location, role, and trend aggregates for analytics |
| Serving | Make curated data available through PostgreSQL, OpenSearch, and FastAPI |

Parquet is planned for analytical datasets. Iceberg can be evaluated later if table management requirements justify it.

## Target folder structure

**This is the target structure; create the folders as we reach each stage:**

```text
jobradar-ai/
├── app/                         # FastAPI application
│   ├── main.py
│   ├── api/
│   │   ├── dependencies.py
│   │   └── v1/
│   │       ├── router.py
│   │       └── endpoints/
│   │           ├── health.py
│   │           ├── jobs.py
│   │           └── ingestion.py
│   ├── core/                    # Settings and logging
│   ├── db/                      # Engine, sessions, model base
│   ├── models/                  # SQLAlchemy database models
│   ├── schemas/                 # API request/response schemas
│   ├── repositories/            # Database queries
│   └── services/                # Application use cases
│
├── collectors/                  # Source-specific ingestion
│   ├── base.py
│   ├── fixtures/
│   ├── apis/
│   └── scrapy/
│
├── contracts/                   # Shared job/event schemas
│                                 # Used by collectors and pipelines
├── pipelines/
│   ├── batch/                   # PySpark transformations
│   │   ├── bronze_to_silver/
│   │   └── silver_to_gold/
│   └── streaming/               # Kafka consumers/processors
│
├── orchestration/
│   └── airflow/
│       └── dags/
│
├── migrations/                  # Alembic database migrations
├── analytics/                   # Metric definitions and SQL
│
├── docker/
│   ├── api/
│   │   └── Dockerfile
│   ├── airflow/
│   │   └── Dockerfile
│   └── spark/
│       └── Dockerfile
│
├── infra/
│   └── terraform/               # AWS infrastructure
│       ├── modules/
│       └── environments/
│
├── config/                      # Non-secret service configuration
├── scripts/                     # Setup and operational helpers
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/                # Small, version-controlled samples
│
├── data/                        # Local generated data; Git-ignored
├── docs/
│   ├── architecture/
│   └── decisions/
│
├── .github/
│   └── workflows/               # CI/CD
├── .env                         # Local secrets; Git-ignored
├── .env.example                 # Documented configuration template
├── .gitignore
├── .dockerignore
├── compose.yaml                 # Base local development services
├── compose.data.yaml            # Later: Kafka, Airflow, Spark
├── alembic.ini
├── pyproject.toml
└── README.md
```

Add `__init__.py` files where Python packages are introduced; they are omitted from the target tree for readability.

### Boundaries and conventions

- `app/api/` handles HTTP routing and dependencies; `app/services/` contains application use cases; `app/repositories/` contains database access.
- `app/models/` defines persisted SQLAlchemy models. `app/schemas/` defines API request and response models.
- `contracts/` defines canonical job and event schemas shared by collectors and processing code. Collectors should not depend on FastAPI routing code.
- `collectors/fixtures/` contains fixture collector code. `tests/fixtures/` contains the actual small JSON/HTML samples.
- `pipelines/` contains transformations and stream processing. `orchestration/airflow/dags/` contains scheduling and workflow definitions.
- `migrations/` tracks database schema changes through Alembic.
- `docker/` contains service Dockerfiles. Root Compose files define how local services run together.
- `infra/terraform/` contains AWS infrastructure definitions. `config/` holds non-secret runtime configuration.
- `data/` is for generated local data; small committed test samples belong in `tests/fixtures/`.
- `pyproject.toml` will centralize Python dependencies and tool settings. Dependency versions and a reproducible installation workflow will be recorded when it is introduced.

## Local development: Windows with Git Bash

Run the following commands in **Git Bash**, not PowerShell. The initial API runs in a Windows virtual environment; PostgreSQL will run in Docker Desktop.

Versions reported during initial setup:

| Tool | Reported version |
| --- | --- |
| Python | 3.12.3 |
| Git | 2.55.0.windows.2 |
| Docker | 26.1.4 |
| Docker Compose | v2.27.1-desktop.1 |

These are the initial development-machine versions, not a tested compatibility matrix for all future services.

### 1. Clone and activate the virtual environment

For a new checkout:

```bash
git clone https://github.com/code-with-koko/jobradar-ai.git
cd jobradar-ai
python -m venv .venv
source .venv/Scripts/activate
```

If the repository and virtual environment already exist, enter that folder and run only the activation command. If `python` is unavailable before activation, use `py -m venv .venv`.

Check the tools:

```bash
python --version
git --version
docker --version
docker compose version
```

### 2. Keep local artifacts out of Git

Create or extend the root `.gitignore` with:

```gitignore
.venv/
.env
__pycache__/
*.py[cod]
.pytest_cache/
.ruff_cache/
/data/
```

Commit `.env.example` with placeholders, never real credentials. When Dockerfiles are introduced, use `.dockerignore` to exclude `.git/`, `.venv/`, `.env`, caches, and generated data from build contexts.

### 3. Reproduce the initial FastAPI application

The initial local application has been verified by its developer. Until that code is committed, a fresh clone must create it using this bootstrap section.

Install the initial dependencies:

```bash
python -m pip install fastapi "uvicorn[standard]"
mkdir -p app
touch app/__init__.py
```

If `app/main.py` does not already exist, create it with:

```python
from fastapi import FastAPI

app = FastAPI(
    title="JobRadar AI",
    description="European tech job market intelligence API",
    version="0.1.0",
)


@app.get("/health")
def health_check():
    return {"status": "ok"}
```

Start the server from the repository root:

```bash
python -m uvicorn app.main:app --reload
```

- [Health endpoint](http://127.0.0.1:8000/health): expected response `{"status":"ok"}`.
- [Interactive API documentation](http://127.0.0.1:8000/docs): try the health endpoint.

This health endpoint checks application responsiveness only; it does not check database connectivity. `--reload` is for local development. See the [FastAPI first-steps guide](https://fastapi.tiangolo.com/tutorial/first-steps/) for routing and interactive documentation.

### 4. Next step: PostgreSQL with Docker Compose

This configuration is proposed and has not yet been confirmed running. Create `compose.yaml` at the repository root:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: jobradar
      POSTGRES_USER: jobradar
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U jobradar -d jobradar"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  postgres_data:
```

Create `.env.example` alongside it:

```dotenv
POSTGRES_PASSWORD=replace_with_a_local_development_password
```

Copy it only if a local `.env` does not already exist:

```bash
test -f .env || cp .env.example .env
```

Set your development password in `.env`. Compose reads this file for variable interpolation; the required-value expression in the YAML rejects a missing or empty password. See [Docker's interpolation documentation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/).

With Docker Desktop running Linux containers, open another Git Bash terminal at the repository root:

```bash
docker compose up -d
docker compose ps
```

Wait for `db` to become healthy, then verify it:

```bash
docker compose exec -T db psql -U jobradar -d jobradar -c "SELECT current_database();"
```

The result should be `jobradar`. The `-T` option avoids pseudo-terminal issues in Git Bash.

Useful commands:

```bash
docker compose logs db
docker compose stop
docker compose start
```

The named volume preserves database files when containers are recreated. The API will connect to `127.0.0.1:5432` while it runs on the Windows host. A future containerized API on the same Compose network will connect to `db:5432`. Starting PostgreSQL alone does not create the SQLAlchemy integration or application tables.

### 5. Docker and Compose file placement

Keep the base `compose.yaml` at the root so `docker compose up -d` works from the repository folder. PostgreSQL uses an official image and needs no custom Dockerfile.

When the API is containerized, its service will use the repository root as the build context:

```yaml
build:
  context: .
  dockerfile: docker/api/Dockerfile
```

This is a future service fragment, not a standalone Compose file. The root context allows the image to copy both `app/` and shared `contracts/`.

Later, after `compose.data.yaml` is implemented, combine it with the base configuration:

```bash
docker compose -f compose.yaml -f compose.data.yaml up -d
```

Use the same file selection for subsequent `ps`, `logs`, and `stop` commands for that stack. Paths in the combined configuration are resolved relative to the first Compose file. See [Docker's guide to merging Compose files](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/).

## Canonical job record

The initial shared job schema will contain:

| Field | Purpose |
| --- | --- |
| `source` | Source identifier |
| `external_id` | Identifier assigned by the source |
| `company` | Employer name |
| `title` | Original job title |
| `country`, `city` | Location |
| `description` | Original description |
| `employment_type` | Full-time, contract, or another employment type |
| `seniority` | Junior, mid-level, senior, lead, or unknown |
| `salary_min`, `salary_max`, `currency` | Advertised compensation when available |
| `remote_type` | On-site, hybrid, remote, or unknown |
| `posted_at` | Source posting timestamp when available |
| `source_url` | Original posting URL |
| `collected_at` | Ingestion timestamp |

Source availability and validation rules will determine which fields are required. Preserve unknown values rather than inventing data. Retain original titles alongside later normalized role categories. Salary comparisons will require an explicit pay period and normalization policy before aggregation.

Start with `jobs` and `ingestion_runs` tables. Enforce uniqueness on `(source, external_id)` for repeatable ingestion. This prevents duplicate records from the same source; detecting the same posting across different sources is a later deduplication task.

Track ingestion outcomes, including failed validation, so runs can be inspected and retried. Preserve the collector contract:

```text
fetch -> parse -> normalize -> validate -> persist
```

The pipeline begins with a local fixture or one permitted API/feed. More adapters can then produce the same canonical records.

## Initial API contract

| Method | Path | Purpose | Status |
| --- | --- | --- | --- |
| GET | `/health` | Application health response | Working locally |
| GET | `/jobs` | Paginated list of stored jobs | Planned |
| GET | `/jobs/{job_id}` | Retrieve one job; return 404 when absent | Planned |
| POST | `/ingestion/run` | Trigger the initial ingestion flow | Planned |

The target `api/v1/` folder organizes router code. Initial URLs remain as shown; an external version prefix can be chosen when versioned routers are introduced.

## Implementation roadmap

| Step | Deliverable | Main tools |
| --- | --- | --- |
| 1 | Working fixture/source ingestion, validation, storage, retrieval, and checks | Python, FastAPI, Pydantic, PostgreSQL, SQLAlchemy, Alembic |
| 2 | Multiple source adapters, retries, rate limiting, and consistent records | httpx, Scrapy, feeds/APIs |
| 3 | Immutable raw payloads and Bronze storage | AWS S3, Parquet where appropriate |
| 4 | Scheduled incremental ingestion and run monitoring | Airflow |
| 5 | Event-driven ingestion and consumer processing | Kafka, Python |
| 6 | Silver cleaning, normalization, deduplication, and enrichment | PySpark |
| 7 | Gold datasets for salaries, skills, cities/countries, and trends | PySpark, SQL |
| 8 | Search and analytics APIs | FastAPI, PostgreSQL, OpenSearch |
| 9 | Skill extraction and CV-to-market gap analysis | Python NLP; optional LLM integration |
| 10 | Cloud deployment, CI/CD, infrastructure management, and observability | AWS, Docker, Terraform, GitHub Actions, CloudWatch |

The initial planning horizon is roughly three months, with scope adjusted as each milestone is validated. AI features are planned for a later stage and are not required for the first backend.

### Step 1 definition of done

The whole first milestone is complete when:

- [x] The repository and local Python environment exist.
- [x] The minimal FastAPI application starts locally.
- [x] `GET /health` returns `{"status":"ok"}` locally.
- [ ] Initial application code and dependency configuration are committed.
- [ ] PostgreSQL runs locally through Docker Compose and responds to a query.
- [ ] SQLAlchemy connects successfully and Alembic manages schema changes.
- [ ] The canonical Pydantic job schema is implemented.
- [ ] `jobs` and `ingestion_runs` tables exist.
- [ ] At least 20 sample jobs can be ingested.
- [ ] Repeating ingestion does not create duplicates.
- [ ] `GET /jobs` returns stored records with pagination.
- [ ] `GET /jobs/{job_id}` returns a job or an appropriate not-found response.
- [ ] Invalid records and ingestion failures are handled and recorded.
- [ ] Relevant unit/integration tests and lint checks pass.
- [ ] Setup instructions reproduce the working milestone.

The next implementation sequence is: start PostgreSQL, connect SQLAlchemy, introduce migrations and the job schema, build the fixture collector, add the remaining endpoints, and verify the complete flow.

## Collection and quality principles

Use official APIs, feeds, and permitted public pages. Scrapy is a source adapter within the ingestion platform. BeautifulSoup/lxml can support parsing; browser rendering can be considered only when a source requires it.

As collection expands, add retries, rate limiting, incremental crawling, change detection, schema validation, failed-message handling, idempotency, and scheduling. Respect source terms and access restrictions.

Use small fictional fixtures for predictable development and testing. Keep source identity and timestamps so curated records can be traced back to their origin. Publish market metrics only from actual collected observations.

## Portfolio outcome

The intended result demonstrates multi-source Python ingestion, API development, data modeling, orchestration, streaming, Spark processing, data-lake design, search, cloud deployment, and observability. Documentation and demos should identify which capabilities are implemented and which remain on the roadmap.
