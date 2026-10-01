# api/: Stage 1 Mock Source API

FastAPI service that acts as the upstream operations system.

**Put here:**
- The FastAPI app (e.g. `main.py`), Pydantic models, and the seeded data generator
- OAuth2 client-credentials token endpoint (`POST /oauth/token`)
- Paginated `wells` and `production` endpoints with `updated_since`
- Configurable 429 / 5xx failures and messy data
- `/admin/advance-day`, `/admin/restate`, `/admin/reset`, `/health`
- `Dockerfile` and pinned dependencies (`pyproject.toml` / `requirements.txt`)

See **Stage 1** in the root `README.md` for the full requirements.
