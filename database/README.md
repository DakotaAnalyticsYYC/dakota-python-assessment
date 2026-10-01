# database/: Stage 2 Database Design

Your DDL for a **medallion architecture** (bronze → silver → gold). The design is up to you.

**Put here:**
- Re-runnable SQL scripts (e.g. numbered `001_...sql`, `002_...sql`) that create your schemas, tables, keys, constraints and indexes
- They run automatically when `run.bat` starts the stack (e.g. mounted into the Postgres container's `/docker-entrypoint-initdb.d/`, or applied by a migration step)

Document the design in `docs/er_diagram.md`, `docs/data_dictionary.md` and the **Database Design** section of `docs/decisions.md`. See **Stage 2** in the root `README.md`.
