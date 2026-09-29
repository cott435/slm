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

## D3 — Ledger transport from the GPU box: spool files imported later, or a tunnel to the Mac's Postgres?
Scope: store, training
Raised by: /dev-team:plan-repo (interview)
Recommendation: a file spool. Training on the rented box writes `Run` / `Metric` / `Cost` / `ModelVersion` events to `var/spool/<run_id>.jsonl` through `store`'s API, and `store-import-spool` loads them into Postgres after the `rsync` back. No network path from a rented box to a Mac behind NAT, the run survives the Mac being asleep, and the spool doubles as an offline log. A reverse SSH tunnel can be added later without changing the API: the same `store` calls would simply write directly.
Assumption if unanswered: spool files; `store` writes directly when Postgres is reachable and spools otherwise; the import command is idempotent.
Decision:
Status: open
Applied:

## D4 — v1 language scope: English only, or keep other languages in snapshots?
Scope: corpus
Raised by: /dev-team:plan-repo (interview)
Recommendation: English only in v1 snapshots. A 0.5B model has no capacity to spare, the summary-pair datasets are English, and the judges score English most reliably. Other languages are still stored with their detected `lang` so the filter can be widened later without re-ingesting.
Assumption if unanswered: `lang = en` is the default snapshot filter; other languages are kept in the database and excluded from snapshots.
Decision:
Status: open
Applied:

## D5 — Ledger spool: carry `JudgeScore` events too, so RL on the GPU box fills the judge-score cache?
Scope: repo
Raised by: /dev-team:plan-package store (interview)
Recommendation: yes. The repo contract's spool shape lists Run / Metric / Cost / ModelVersion events only, but RL rollouts are scored on the GPU box, where Postgres is unreachable; without a `judge_score` event every score the box produces is lost to the cache and would be paid for again at evaluation time. On the box `get_judge_score` returns a miss and `put_judge_score` spools; `store-import-spool` upserts them on key. Adds one line to the repo contract's Ledger spool row at the next `/dev-team:plan-repo` edit.
Assumption if unanswered: yes — the spool carries `judge_score` events and the importer applies them insert-or-ignore on `key`.
Decision:
Status: open
Applied:
