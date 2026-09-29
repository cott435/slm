# Decisions

One sequence for the whole repo. Stubs are appended by the architect and the designer; answers
go in `Decision:` and `Status:` and belong to the user.

## D1 — Dashboard web UI: Streamlit, or a FastAPI-served page (Jinja/HTMX), or Gradio?
Scope: serving
Raised by: /dev-team:plan-repo (interview)
Recommendation: Streamlit for the dashboard (data statistics, training progress, reports, sample viewer, catalog browsing), with the summarize endpoint as a separate FastAPI app. Single user, read-mostly views over Postgres and report files; Streamlit gives tables and plots with no frontend code, which keeps the serving package small. FastAPI stays the endpoint either way — it is the default for a local Python API.
Assumption if unanswered: Streamlit dashboard + FastAPI summarize endpoint, two commands in the `serving` package.
Decision:
Status: open
Applied:

## D2 — Experiment tracking: own Postgres tables, or MLflow?
Scope: repo
Raised by: /dev-team:plan-repo (interview)
Recommendation: Own tables (`runs`, `metrics`, `costs`) in the shared Postgres, owned by the lowest package, written by training / evaluation / judge calls and read by the dashboard. One system instead of two, the schema is the user's to understand (the learning goal), the cost ledger (GPU hours, API spend per run) has nowhere natural to live in MLflow, and the dashboard the brief asks for already covers what MLflow's UI would add. MLflow could still be adopted later as a second sink without moving the source of truth.
Assumption if unanswered: own Postgres tables; no MLflow.
Decision:
Status: open
Applied:
