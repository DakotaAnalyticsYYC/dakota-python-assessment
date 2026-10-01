# app/: Stage 4 Writeback App

Web app for browsing, filtering, editing and reverting values. You choose the framework (Streamlit, Dash, NiceGUI, React + FastAPI, ...).

**Put here:**
- App source code
- Validation and the data-access layer, which reads source data and writes **only** to your writeback tables (never to bronze or silver)
- `Dockerfile` and pinned dependencies

`run.bat app` must open this app in the browser. See **Stage 4** in the root `README.md`.
