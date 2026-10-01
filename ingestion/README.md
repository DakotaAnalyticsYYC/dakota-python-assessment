# ingestion/: Stage 3 Python Ingestion

Python client and loader that pull from the mock API into the database. Runs in a container started by `run.bat pipeline`.

**Put here:**
- API client: token handling and refresh, retries with backoff, pagination
- Incremental watermark handling per entity
- Landing every received record unchanged in **bronze**
- Cleaning into **silver** (dedupe, type coercion, flattening, date normalisation), maintaining source history, and quarantining rejected records
- Idempotent upserts, plus the ingestion run log
- `Dockerfile` and pinned dependencies

If you build silver with dbt instead, the ingestion stops at bronze. Explain your choice in `docs/decisions.md`. See **Stage 3** in the root `README.md`.
