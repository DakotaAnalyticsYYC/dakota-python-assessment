# dbt/: Stage 5 Reconciliation Models

dbt project (`dbt-postgres`) that builds the **gold** layer, comparing silver source data (and its history) with writeback data.

**Suggested layout:**
```
dbt/
├── dbt_project.yml
├── profiles.yml          # reads connection details from env vars
└── models/
    ├── staging/          # stg_ models over silver and writeback tables (+ sources.yml, schema.yml tests)
    ├── intermediate/     # effective (golden) view: source + active edits
    └── marts/            # recon_field_diff, recon_summary, recon_recommendations (+ tests)
```

dbt runs inside a container (`run.bat` / `run.bat pipeline`). See **Stage 5** in the root `README.md`.
