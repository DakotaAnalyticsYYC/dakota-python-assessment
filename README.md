# Dakota Analytics: Python Data Engineering Assessment

Build an end-to-end data platform: generate data from a mock API, ingest it into a database, let users correct it in a writeback app, and then reconcile the corrected data against the source.

---

## Read This First: What You Need to Do

1. **Use this repository as your starting point.** The folders already exist (`api/`, `ingestion/`, `app/`, `dbt/` and so on), and each one has a `README.md` describing what goes in it.
2. **Build the five stages** described in [What You Will Build](#what-you-will-build):
   1. A mock source API (FastAPI)
   2. **Database design**: your own PostgreSQL design, following a **medallion architecture** (bronze → silver → gold)
   3. Python ingestion into that database
   4. A writeback app for editing the data
   5. A dbt reconciliation model with recommendations
3. **Write a `run.bat` script** at the repository root. It must start and run the **whole application on a Windows laptop with one command**. This is the most important deliverable: if `run.bat` doesn't work, we can't test your submission. See [The `run.bat` Script](#the-runbat-script-required).
4. **Fill in the docs** in `docs/` (architecture, decisions, ER diagram, data dictionary) and update this README's [Your Notes](#your-notes-fill-in) section.
5. **Submit** by following [SUBMISSION.md](SUBMISSION.md).

> **How we will test it:** on a clean Windows laptop that has only **Docker Desktop** and **Git** installed, we'll run:
>
> ```bat
> git clone <your-repo-url>
> cd <your-repo>
> run.bat
> ```
>
> We then expect the API, the database and the writeback app to be running, the pipeline to have loaded data, and the app to open in the browser. We won't install Python, WSL, Git Bash, `make`, or anything else.

---

## Scenario

A field operations team gets daily well production data from an upstream operations system. The data isn't always right. Meters drift, allocations get keyed in wrong, and wells get the wrong status. Production engineers fix these values by hand.

Leadership wants to know three things:

1. **What has been changed** compared with what the source system says?
2. **Is each change still valid** now that the source system has sent new or restated data?
3. **What should we do about it?** Keep the edit, retire it, escalate it, or fix the source?

You'll build the platform that answers those questions.

## Architecture

![Architecture](python-assessment-architecture.png)

```
┌──────────────────────────── Your code repository ──────────────────────────────┐
│                                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──── 2. PostgreSQL (you design) ────┐ │
│  │  1. Mock API │───►│ 3. Python    │───►│  Bronze   data as received         │ │
│  │  (FastAPI)   │    │  Ingestion   │    │    ▼                               │ │
│  │  OAuth2,     │    │  incremental,│    │  Silver   cleaned & conformed      │ │
│  │  pagination, │    │  retries,    │    │    ▼                               │ │
│  │  messy data  │    │  idempotent  │    │  Gold     business-ready (dbt)     │ │
│  └──────────────┘    └──────────────┘    │  + writeback & metadata: your call │ │
│                                          └──────▲───────────────┬─────────────┘ │
│                                                 │ edits         │ dbt           │
│                                       ┌─────────┴──────┐  ┌─────▼──────────┐    │
│                                       │ 4. Writeback   │  │ 5. Reconcile   │    │
│                                       │    App (UI)    │  │  diff, classify│    │
│                                       └────────────────┘  │  recommend     │    │
│                                                           └─────┬──────────┘    │
│                                                                 ▼               │
│                                                       Report (your choice)      │
│   run.bat: one command starts and runs everything on Windows                    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                │ you share the repository
                                ▼
       Dakota clones the repo on a Windows laptop, runs run.bat, edits data,
       triggers restatements, reruns the pipeline, and reviews the output
```

Everything runs in **Docker containers** (Docker Compose). The only thing on the host is `run.bat`.

---

## Repository Structure

This is the structure we provide. Keep it so reviewers know where to look. You can add files and subfolders inside it.

```
dakota-python-assessment/
├── README.md               # This brief; complete the "Your Notes" section at the end
├── SUBMISSION.md           # Submission checklist
├── .env.example            # Environment variable template (add your own variables)
├── .gitignore
├── python-assessment-architecture.png
│
├── run.bat                 # YOU CREATE: one-command Windows runner (required)
├── run.sh                  # YOU CREATE: optional macOS/Linux equivalent
├── docker-compose.yml      # YOU CREATE: all services
│
├── api/                    # Stage 1: FastAPI mock source API (+ Dockerfile)
├── ingestion/              # Stage 3: Python ingestion client + loader (+ Dockerfile)
├── database/               # Stage 2: DDL scripts for your medallion design
├── app/                    # Stage 4: writeback app (+ Dockerfile)
├── dbt/                    # Stage 5: dbt project (builds the gold layer)
│   └── models/
│       ├── staging/        # stg_ models over silver and writeback data
│       ├── intermediate/   # effective (golden) view: source + active edits
│       └── marts/          # recon_field_diff, recon_summary, recon_recommendations
├── reports/                # Report generator
│   └── output/             # Generated reports land here
├── tests/                  # pytest tests
└── docs/
    ├── architecture.md     # Components, data flow, how edits and restatements interact
    ├── decisions.md        # Choices, alternatives, trade-offs, assumptions, AI usage
    ├── er_diagram.md       # Mermaid ER diagram of your whole design
    └── data_dictionary.md  # Every table and column: grain, type, constraints, meaning
```

---

## What You Will Build

Each stage has **Must have** requirements, which are needed to pass, and **Stretch** ideas, which are optional. Get every must-have working end to end through `run.bat` before you start on any stretch goal. A complete, solid submission beats a half-finished ambitious one.

### Stage 1: Mock Source API (FastAPI) → `api/`

Build a FastAPI service that acts as the upstream operations system. It generates **synthetic, seeded (reproducible)** oil & gas data and behaves like a real-world API, including its flaws.

**Minimum data domain** (you may add more entities and fields):

| Entity | Key | Minimum fields |
|---|---|---|
| `wells` | `well_id` | `well_name`, `operator`, `basin`, `state`, `well_type` (Oil/Gas/Dual), `status` (Active/Inactive/Shut-in/Plugged), `updated_at` |
| `production` | (`well_id`, `production_date`) | `oil_bbl`, `gas_mcf`, `water_bbl`, `uptime_hours`, `updated_at` |

Generate at least **50 wells** and **90 days** of daily production.

**Must have**

- [ ] **OAuth2 client-credentials auth.** `POST /oauth/token` exchanges a `client_id` and `client_secret` for a bearer token. Tokens expire after `TOKEN_TTL_SECONDS` (configurable; default 300). Data endpoints return `401` for a missing or expired token.
- [ ] **Pagination** on every list endpoint (page/offset or cursor, your choice). Responses include whatever the client needs to fetch the next page.
- [ ] **Incremental support.** List endpoints accept `updated_since` (ISO-8601) and return only records created or changed after that timestamp.
- [ ] **Realistic failures**, configurable through environment variables:
  - Rate limiting that returns `429` with a `Retry-After` header
  - Random transient `500`/`503` responses at a configurable rate (e.g. `API_ERROR_RATE=0.05`)
- [ ] **Messy data.** At minimum, the payload includes:
  - Duplicate records within and across pages
  - Nulls in non-key fields
  - Numbers sometimes sent as strings (e.g. `"1,234.5"`)
  - At least one nested object (e.g. `measurements: {oil: …, gas: …}`)
  - Occasional out-of-range values (negative volumes, `uptime_hours > 24`)
  - Mixed date or timestamp formats
- [ ] **Upstream restatements.** The source system changes historical data over time. Provide these admin endpoints (no auth needed; reviewers will call them):
  - `POST /admin/advance-day` adds the next day of production for every well and **restates a random sample** of past production records (changes values and bumps `updated_at`). Returns what changed.
  - `POST /admin/restate`, with body `{ "well_id": ..., "production_date": ..., "field": ..., "value": ... }`, restates one specific value. Reviewers use this to create conflicts on purpose.
  - `POST /admin/reset` restores the initial seeded state.
- [ ] OpenAPI docs available at `/docs`, plus a `GET /health` endpoint.

**Stretch:** soft deletes (records removed upstream, exposed via `is_deleted` or a deletes feed), ETag or `If-Modified-Since` support, schema versioning, contract tests.

### Stage 2: Database Design → `database/`, `docs/`

Design the PostgreSQL database using a **medallion architecture** (bronze → silver → gold). **The design is yours.** You decide the schemas, tables, keys, data types, constraints, indexes, how source history is kept, and where writeback data and pipeline metadata live. We assess the choices you make and how well you justify them, not whether you match a template.

**Must have**

- [ ] **Medallion architecture** with clearly separated **bronze**, **silver** and **gold** layers. You decide exactly what each layer holds and document it.
- [ ] Your design must support the rest of the assignment:
  - the data **as received** from the API is kept
  - **user edits** are stored separately from source data, with a full audit trail
  - **restated source values are not lost**: you can tell what the source said at the time an edit was made, and whether it has changed since (Stage 5 depends on this)
  - ingestion metadata (run log, watermarks, rejected records) has a home
- [ ] **DDL scripts** in `database/`, re-runnable, and applied automatically when `run.bat` starts the stack.
- [ ] **Documentation:**
  - `docs/er_diagram.md`: a Mermaid ER diagram of your design, with PKs and FKs
  - `docs/data_dictionary.md`: each table's layer, grain and purpose, plus each column's type, constraints and meaning
  - `docs/decisions.md`: a **Database Design** section answering the questions below

**Design questions** (answer them in `docs/decisions.md`; we'll discuss them in the live session):

1. What goes in each layer, and why did you draw the boundaries there?
2. Natural or surrogate keys: which did you choose, and why?
3. How is a restatement represented, and how do you find the source value at the time of an edit?
4. Where does validation live (database constraints, Python, or dbt tests), and what happens when a value passes one and fails another?
5. Where do writeback data and pipeline metadata sit relative to the medallion layers, and why?
6. What would you change in the design at 100× the data volume?

**Stretch:** least-privilege database roles, audit immutability enforced in the database, partitioning, a schema migration tool (Alembic, Flyway, sqitch).

### Stage 3: Python Ingestion → PostgreSQL → `ingestion/`

Write a Python ingestion process that pulls from **your API** and loads it into the database you designed in Stage 2: land in **bronze**, then clean into **silver**. It runs inside a container, started by `run.bat`.

**Must have**

- [ ] **Auth handling.** Gets a token, and **refreshes it automatically** when it expires mid-run. Never hardcode credentials; read them from the environment.
- [ ] **Resilience.** Retries with exponential backoff (and jitter) on `429`/`5xx`, honours `Retry-After`, and fails loudly with a clear error once retries run out. Don't retry other `4xx` errors.
- [ ] **Full pagination**, with no silently skipped pages.
- [ ] **Incremental loads.** Persist a high-water mark per entity and request only `updated_since` on later runs. The first run is a full load.
- [ ] **Idempotent loads.** Running the same load twice produces the same silver state: no duplicate current rows and no spurious history versions (upsert on business keys; only create a new version when values actually change).
- [ ] **Bronze then silver.** Land every received record in bronze unchanged first. Then clean it into silver: deduplicate, coerce types (e.g. `"1,234.5"` → `1234.5`), flatten nested fields, normalise dates, and maintain the source history. Quarantine invalid records (with the reason) in a rejected-records table, instead of dropping them silently or crashing.
- [ ] **Run logging.** An ingestion log table records the run id, entity, start and end time, rows fetched, inserted, updated and rejected, watermark before and after, and status.
- [ ] Structured logging to stdout. Use `logging`, not `print`.

**Stretch:** async or concurrent page fetching, a pluggable auth strategy (API key / OAuth2 / mTLS behind one interface), checkpoint and resume mid-run, a dead-letter replay command.

### Stage 4: Writeback App → `app/`

Build a web app that lets a production engineer **review and correct** the ingested data. You choose the UI framework (Streamlit, Dash, NiceGUI, React + FastAPI, etc.) and explain the choice in `docs/decisions.md`. It runs in a container and opens in the browser.

**Must have**

- [ ] **Browse and filter** production records (at least by well, operator or basin, and date range) and wells.
- [ ] **Edit** these fields at minimum: `oil_bbl`, `gas_mcf`, `water_bbl`, `uptime_hours` on production, and `status` on wells. Each edit also captures:
  - **who** made it (a simple username prompt or selector is fine; real auth isn't required)
  - a **reason** (required free text, or a reason code plus a comment)
- [ ] **Validation** before save: non-negative volumes, `0 ≤ uptime_hours ≤ 24`, status from the allowed list. Show clear error messages.
- [ ] **Edits are stored separately** from the source copy, in the writeback tables you designed. **The app must never modify bronze or silver data.**
- [ ] **Append-only audit trail.** Every save records the entity, key, field, the **source value at the time of the edit**, the old value, the new value, the user, the reason and the timestamp. Editing the same field twice keeps both history rows.
- [ ] **Edits survive re-ingestion.** Running the pipeline again must never lose or overwrite a user edit.
- [ ] Wherever an edited value is shown, show the **current source value next to it**.
- [ ] **Revert.** A user can withdraw an edit so the record goes back to the source value. The withdrawal is also audited.
- [ ] A **"Run pipeline" or "Refresh" action** in the app (or a clear instruction to use `run.bat pipeline`) so a reviewer can re-ingest and see the reconciliation update.

**Stretch:** an approval workflow (draft → approved), optimistic concurrency (detect when two users edit the same record), bulk edit via CSV upload, row-level change history view.

### Stage 5: Reconciliation Model and Recommendations (dbt) → `dbt/`, `reports/`

Use **dbt** (`dbt-postgres`) to build models in the **gold** layer that compare the **source data** (silver, including its history) with the **writeback data**, explain what changed, and recommend actions.

**Must have**

- [ ] **Staging models** over your silver and writeback tables, with dbt tests (`not_null`, `unique`, `accepted_values`, relationships).
- [ ] An **effective (golden) view**: the source data with the active edits applied on top. Downstream consumers read this view.
- [ ] **`recon_field_diff`**: one row per active edited field, with at least:
  - entity, business key, field
  - `source_value_at_edit`, `current_source_value`, `writeback_value`
  - absolute and percentage delta between writeback and current source
  - edited by, edited at, reason, days since edit
  - **`classification`**, using the definitions below
- [ ] **`recon_summary`**: aggregate counts and deltas by classification, well, operator, field and user.
- [ ] **`recon_recommendations`**: one row per recommendation, with a `severity`, a `recommendation`, a plain-English `rationale` and the affected keys. Implement **at least four** rules. Examples:
  - `source_caught_up` → *retire the edit; the source now agrees*
  - `conflict` → *needs review: the source changed after the edit was made*
  - An edit with a delta above X% (configurable) → *high-impact change: require a second approver*
  - A well with N or more overrides in a period → *probable source or meter issue: raise with the upstream system owner*
  - An edit open for more than N days without the source agreeing → *stale override: confirm it is still valid*
- [ ] **A report** in a format you choose (HTML, notebook, Excel, PDF, or a page in the writeback app). It summarises the reconciliation and lists the recommendations, and is written to `reports/output/` (or shown in the app).

**Classification definitions** (apply them exactly; add more if you like):

| Classification | Meaning |
|---|---|
| `override` | Writeback value differs from the source, and the source has **not changed** since the edit was made |
| `source_caught_up` | The source changed after the edit and now **equals** the writeback value |
| `conflict` | The source changed after the edit to a value **different** from both the original and the writeback value |
| `orphaned` | The edited record no longer exists in the source (only if you implemented deletes) |

**Stretch:** an anomaly score that suggests records users *should* review, dbt snapshots of source history, dbt docs, a generated natural-language summary of the recommendations.

---

## The `run.bat` Script (Required)

`run.bat` is how we'll run and test your work. It must run from **Command Prompt or PowerShell** on Windows with only **Docker Desktop** installed. That means no WSL, Git Bash, `make` or host Python. Run everything (ingestion, dbt, tests, reports) **inside containers**, e.g. `docker compose run --rm ingestion ...`.

| Command | Must do |
|---|---|
| `run.bat` *(no arguments)* | **One-command demo:** check that Docker is running, create `.env` from `.env.example` if missing, build and start all services, wait until healthy, run the full pipeline, then **open the writeback app in the default browser** and print every URL |
| `run.bat start` | Build and start all services and wait until healthy |
| `run.bat pipeline` | Run ingestion (incremental after the first run) → `dbt build` → generate the report |
| `run.bat app` | Open the writeback app in the browser (start it if it isn't running) |
| `run.bat reconcile` | Run only the dbt reconciliation models and the report |
| `run.bat test` | Run pytest and dbt tests, and exit non-zero if anything fails |
| `run.bat status` | Show service status and URLs (API docs, app, report location) |
| `run.bat stop` | Stop services and keep the data |
| `run.bat clean` | Remove containers, volumes and generated output |

Requirements for the script:

- [ ] Prints clear progress messages and a clear error (e.g. *"Docker Desktop is not running"*) instead of failing silently
- [ ] Stops at the first failed step and exits with a non-zero code
- [ ] Works from a path that contains spaces (e.g. `C:\Users\Jane Doe\Projects\...`)
- [ ] Is safe to run repeatedly: `run.bat` twice in a row must not break anything or duplicate data
- [ ] Ports are configurable in `.env` in case a reviewer's port is already in use

A `run.sh` for macOS/Linux is welcome but optional. **`run.bat` is what we grade.**

---

## Technical Requirements

| Area | Requirement |
|---|---|
| Language | Python 3.11+, with type hints on public functions |
| API | FastAPI + Pydantic |
| Database | PostgreSQL 15+ (in Docker) |
| Transformation | dbt-core + dbt-postgres |
| Writeback UI | Your choice; justify it in `docs/decisions.md` |
| Runtime | Docker Compose, started through `run.bat` on Windows |
| Config | All config through environment variables / `.env`. Commit `.env.example`; **never** commit `.env` or secrets. |
| Tests | `pytest` unit tests for the ingestion logic (token refresh, retry/backoff, pagination, cleaning, idempotency) and at least one API test. dbt tests on models. |
| Dependencies | Pinned (uv, Poetry or pip-tools, your choice) |

Orchestration tools (Airflow, Dagster, etc.) and CI pipelines are **not required**. `run.bat` is the orchestrator.

## How You Will Be Evaluated

| Area | What we look for |
|---|---|
| **It runs** | `run.bat` brings everything up on a clean Windows laptop the first time |
| **Correctness** | Edits, restatements and reconciliation behave as specified |
| **Reliability** | Token refresh, retries, incremental and idempotent loads, nothing lost silently |
| **Data integrity** | Bronze is never mutated, source history is kept, the audit trail is complete, and edits survive re-ingestion |
| **Database design** | A sound medallion design: clear layer boundaries, sensible keys, types, constraints, history and indexes, with the reasoning documented |
| **Data modelling** | Clean dbt layering, correct classifications, useful tests |
| **Insight** | Recommendations that a production engineer could actually act on |
| **Code quality** | Readable, modular, typed and tested; config kept separate from code |
| **Documentation** | A reviewer can understand *why* you made each choice, not just *what* you built |

After you submit, we'll hold a **live follow-up session** (about 45 minutes). You'll walk us through your solution and we'll discuss changes to the requirements.

### Use of AI tools

You may use AI assistants. In the live session, we'll expect you to explain and extend any part of your code without help. Note in `docs/decisions.md` how you used AI tools.

### Questions

Email **technical-assessment@dakotaanalytics.com**. Where something is ambiguous, make a reasonable assumption and write it down in `docs/decisions.md`. We value that as much as asking.

---

## Your Notes (fill in)

> Replace this section before submitting.

**Quick start:** confirm that `run.bat` is all a reviewer needs, or list anything extra.

**URLs once running:**

| Service | URL |
|---|---|
| Source API docs | http://localhost:8000/docs |
| Writeback app | http://localhost:8501 |
| Report | `reports/output/...` |

**How to use the writeback app:** a short walkthrough of making an edit, reverting it, and viewing reconciliation.

**Assumptions and known limitations:**

**What I'd do next with more time:**
