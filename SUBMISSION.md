# Submission Checklist

## Before Submitting

### 1. Implementation Status
- [ ] Stage 1: Mock API (OAuth2, pagination, `updated_since`, 429/5xx, messy data, `/admin/*` endpoints)
- [ ] Stage 2: Ingestion (token refresh, retries, incremental watermark, bronze → silver, idempotent upsert, rejected records, run log)
- [ ] Stage 3: Database design (medallion bronze → silver → gold, source history kept, writeback separated; re-runnable DDL)
- [ ] Stage 4: Writeback app (filter, edit with user and reason, validation, audit trail, revert)
- [ ] Stage 5: dbt reconciliation in the gold layer (`recon_field_diff`, `recon_summary`, `recon_recommendations`) + report
- [ ] `run.bat` with every command from the README
- [ ] Docs completed in `docs/`, and `README.md` updated with setup instructions
- [ ] Tests written (pytest + dbt tests)

### 2. Test on Windows in a Clean Environment

Clone your repository into a **new folder whose path contains a space** (e.g. `C:\Temp\review test\`). Then, in **Command Prompt or PowerShell** with Docker Desktop running:

```bat
run.bat clean
run.bat
```

Check that:
- [ ] All services start, the pipeline runs, and the writeback app opens in your browser
- [ ] You can make 2–3 edits in the app

Then simulate upstream restatements and rerun:

```bat
curl.exe -X POST http://localhost:8000/admin/advance-day
run.bat pipeline
run.bat test
```

- [ ] Your edits are still there after the incremental load
- [ ] The report shows diffs, classifications and recommendations
- [ ] All tests pass
- [ ] Running `run.bat` a second time works and doesn't duplicate data

### 3. Verify Deliverables

Check that your repository includes:
- [ ] `run.bat` with: *(no args)*, `pipeline`, `test`, `stop`, `clean`
- [ ] `docker-compose.yml` with every service (postgres, api, app, plus anything else you need)
- [ ] `.env.example` with every required variable
- [ ] `api/`, `ingestion/`, `database/`, `app/`, `dbt/`, `reports/`, `tests/` populated
- [ ] `database/` contains re-runnable DDL scripts
- [ ] `docs/architecture.md`, `docs/decisions.md` (including the **Database Design** answers), `docs/er_diagram.md`, `docs/data_dictionary.md` completed
- [ ] A sample generated report in `reports/output/`

### 4. Final Checks
- [ ] Repository is **public** (or Dakota reviewers have been given access)
- [ ] No secrets committed. `.env` is in `.gitignore`.
- [ ] Commit history shows how the work progressed (please don't squash it into a single commit)

## How to Submit

Email: **technical-assessment@dakotaanalytics.com**

**Subject:** Python Assessment Submission - [Your Name]

**Body:**
```
Name: [Your Full Name]
GitHub Repository: [URL]
Writeback UI framework: [e.g. Streamlit]
Report format: [e.g. HTML]
Tested on: [e.g. Windows 11, Docker Desktop 4.x]

Brief Summary:
[3–5 sentences on your approach and anything you're particularly proud of]

Stretch goals attempted:
[List, or "none"]

Known limitations / what you'd do next:
[Bullet list]

Time Spent: [X hours]
```

## What We'll Do

1. Clone your repository on a clean Windows laptop (Docker Desktop + Git only)
2. Read `README.md` and `docs/`
3. Run `run.bat`
4. Make edits in your writeback app, including some invalid ones
5. Trigger upstream restatements and targeted conflicts through `/admin/*`, then run `run.bat pipeline`
6. Check that edits survived and that the reconciliation classifies each change correctly
7. Run `run.bat test`, then review code and the report
8. Invite you to a live follow-up session
