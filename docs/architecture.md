# slm — repo contract

Written by `/dev-team:plan-repo` on 2026-09-28 from `docs/brief.md` (copy: `docs/history/brief-contracted.md`).

## Goal

A personal learning project that builds a ~0.5B-parameter summarization language model end to
end: gather and clean a large free corpus of research papers, news and webpages into Postgres;
freeze versioned snapshots; train in stages (pretrain or continued-pretrain → supervised
fine-tuning → RL from judge-model scores) on both a from-scratch model and an open base model;
evaluate every stage with automatic metrics, LLM judges and visual reports; and serve the
result through a local summarize endpoint and a dashboard. The data milestone comes first.
Everything runs as CLI commands; development and smoke tests happen on the Mac, real training
on a rented GPU box, with the database and raw files on the Mac or an external disk.

## Packages

Dependency-ordered, lowest first. `covers` names the brief's **Scope — now** capabilities;
parentheses mark where a **Scope — later** capability will land.

| package | responsibility | path | depends on | covers | candidate skills |
|---|---|---|---|---|---|
| `store` | The one Postgres schema and the `var/` file tree: sources, documents, summary pairs, raw originals, versioned snapshots with fixed splits, catalog queries, and the ledger — runs, metrics, costs, judge-score cache, model registry (D2). Migrations. Ledger spool for a box that cannot reach Postgres. | `packages/store` | — | Document database, Raw originals, Versioned snapshots, Data catalog, Experiment tracking, Cost tracking, Model registry | none |
| `judge` | Every call to an LLM other than our own: provider adapters (self-hosted Qwen via an OpenAI-compatible server, Anthropic, OpenAI), response caching through the store, per-run spend caps, the scoring rubric, penalties and the composite reward; teacher-summary prompting. | `packages/judge` | store | Judge adapters, Scoring rubric, Reward penalties, Score caching and budget caps, Secrets handling; (Judge ensemble, Judge calibration) | none |
| `corpus` | Getting data in and clean: bulk-dataset ingest, page and feed crawling, summary-pair dataset ingest, teacher-pair generation, source registry, extraction, normalization, dedup, quality filtering, split assignment and the contamination guard, snapshot building, and the data-validation classes with their rendered report. | `packages/corpus` | store, judge | Raw-text corpus ingest, Webpage and feed crawling, Summary-pair ingest, Synthetic teacher pairs, Source registry, Text extraction, Normalization, Deduplication, Quality filtering, Contamination guard, Data validation classes; (Resumable downloads, PII scrubbing, Scheduled ingestion) | none |
| `lm` | The model itself, independent of any training loop: tokenizer training, the from-scratch architecture, loading either path from a checkpoint directory, the prompt format with length control, generation on cuda / mps / cpu. Named `lm` because `model` is the variable name every caller wants. | `packages/lm` | store | Tokenizer training, Length-controllable summaries; (Export and quantization) | none |
| `evaluation` | Judging a model version: automatic metrics, held-out judge scoring, baselines (untrained base, lead sentences, a large API model), training diagnostics from the ledger, visual reports, the stage gate, and the sample-viewer tables. Sits *below* `training` so a training run can end with its own gate. | `packages/evaluation` | store, judge, lm | Automatic summary metrics, Held-out judge evaluation, Baselines, Training diagnostics, Visual reports, Stage gate, Sample viewer | none |
| `training` | The stages and their scripts: pretraining from scratch, continued pretraining, SFT, LoRA, judge-reward RL (GRPO via `trl` — `PPOTrainer` no longer exists there), checkpoint/resume, the single-run script and the hyperparameter-search script; records runs, metrics and costs to the ledger and registers model versions. | `packages/training` | store, judge, lm, evaluation | Pretraining from scratch, Continued pretraining, Supervised fine-tuning, Judge-reward RL, LoRA-style fine-tuning, Checkpoint and resume, Single-run training script, Hyperparameter-search script, Reproducibility; (Multi-GPU training, Full-length document summarization) | none |
| `serving` | The user-facing surfaces: the FastAPI summarize endpoint over a registered model, and the Streamlit dashboard over the catalog, ledger, reports and sample viewer (D1). | `packages/serving` | store, lm, evaluation | Summarize endpoint, Dashboard | none |

Later-scope capability with no row above: **Tests / CI** — the Toolchain section's CI commands
are its home when it arrives.

## Dependency graph

```toml
[tool.importlinter]
root_packages = []          # grown by the scaffold step as packages are first built

[[tool.importlinter.contracts]]
name = "package dependency direction"
type = "layers"
layers = ["serving", "training", "evaluation", "corpus | lm", "judge", "store"]
```

`corpus` and `lm` are independent: neither imports the other. `corpus` has no importing
consumer — its surface is its CLI, and its products reach everyone else through `store`.

## Boundaries

Shapes, not signatures. Every record below is a pydantic model or a typed row exposed by the
provider's top level; the provider's package contract fixes the function signatures.

### store → judge, corpus, lm, evaluation, training, serving

| shape | kind | fields | invariants |
|---|---|---|---|
| `Source` | record | `source_id: str` slug; `name: str`; `kind: bulk-dataset \| crawl \| feed \| pairs-dataset \| teacher`; `license: str`; `license_class: open \| non-commercial \| research-only \| unknown`; `url: str \| None`; `notes: str \| None` | unique on `source_id`; every document and pair references one |
| `Document` | record | `doc_id: str`; `source_id`; `domain: paper \| news \| web`; `url: str \| None`; `title: str \| None`; `raw_ref: str \| None` (path under `var/raw/`); `text: str \| None` (cleaned); `lang: str \| None`; `char_len: int`; `word_len: int`; `content_hash: str \| None`; `quality: float \| None`; `status: raw \| cleaned \| rejected \| duplicate`; `split: train \| val \| test`; `fetched_at: datetime`; `license_class` | unique on `doc_id`; `split` assigned once at ingest and never changed; `text` non-null iff `status = cleaned` |
| `SummaryPair` | record | `pair_id: str`; `doc_id`; `summary: str`; `origin: str` (`dataset:<source_id>` \| `teacher:<judge model>` \| `model:<model_id>`); `length_bucket: short \| medium \| long`; `quality: float \| None`; `created_at` | unique on `pair_id`; inherits `split` and `license_class` from its document |
| `Snapshot` | record + files | `snapshot_id: str` (`<name>-v<n>`); `kind: pretrain \| sft \| eval`; `filter: dict` (the selection that built it); `counts: dict[split, int]`; `created_at`; files `var/snapshots/<snapshot_id>/<split>/part-*.parquet` + `manifest.json` (shard sha256s, counts, filter, git sha) | immutable once written; every row honors the document's `split`; a `test` document never appears in a `train` or `val` shard of any snapshot |
| `Run` | record | `run_id: str`; `kind: ingest \| teacher \| tokenizer \| pretrain \| cpt \| sft \| rl \| hpsearch \| eval`; `config: dict`; `snapshot_id: str \| None`; `base_model_id: str \| None`; `seed: int \| None`; `git_sha: str`; `host: str`; `started_at`; `finished_at: datetime \| None`; `status: running \| done \| failed` | unique on `run_id` |
| `Metric` | record | `run_id`; `step: int`; `name: str`; `value: float`; `ts` | append-only; ordered by `(run_id, name, step)` |
| `Cost` | record | `run_id`; `kind: gpu-hours \| api`; `provider: str`; `units: float`; `usd: float`; `ts` | append-only; summed per run and per calendar month |
| `JudgeScore` | record | `key: str` (sha256 over judge model, rubric version, document hash, summary hash); `judge: str`; `rubric_version: str`; `scores: dict[str, int]`; `reward: float`; `usd: float`; `created_at` | unique on `key`; the cache `judge` reads before it spends |
| `ModelVersion` | record + files | `model_id: str`; `path: lm-scratch \| lm-open`; `stage: base \| pretrain \| cpt \| sft \| rl`; `run_id`; `parent_model_id: str \| None`; `checkpoint_ref: str` (dir under `var/models/`); `eval: dict \| None`; `promoted: bool`; `created_at` | unique on `model_id`; lineage follows `parent_model_id` to a `base` |
| `Catalog` query | typed rows | filter by source, domain, split, status, `lang`, length range, quality range; returns `Document` rows plus aggregate counts and length histograms | read-only; the dashboard and the data-validation classes use only this |
| Ledger spool | file | `var/spool/<run_id>.jsonl`, one `Run` / `Metric` / `Cost` / `ModelVersion` event per line | written wherever Postgres is unreachable; imported by a store command, idempotent on `run_id` and event order |

### judge → corpus, evaluation, training

| shape | kind | fields | invariants |
|---|---|---|---|
| `Completion` | record | `provider: qwen \| anthropic \| openai`; `model: str`; `text: str`; `prompt_tokens: int`; `completion_tokens: int`; `usd: float`; `cached: bool` | `usd = 0` for `qwen`; every uncached call adds a `Cost` row to the caller's run |
| `TeacherSummary` | record | `doc_id`; `summary: str`; `length_bucket`; `judge_model: str`; `usd` | the `origin` of the pair it becomes is `teacher:<judge_model>` |
| `Judgement` | record | `doc_id: str \| None`; `judge: str`; `rubric_version: str`; `scores: {faithfulness, coverage, concision, fluency}: int` each in 1–5; `penalties: {over_length, copying, hallucination}: float` each in 0–1; `reward: float` in 0–1 | deterministic for a given `JudgeScore.key`; `reward = mean(scores − 1) / 4 − weighted penalties`, clamped to `[0, 1]` |
| `BudgetExceeded` | error | `run_id`; `cap_usd`; `spent_usd` | raised before the call that would cross a run's cap; the caller ends the run cleanly |

### lm → evaluation, training, serving

| shape | kind | fields | invariants |
|---|---|---|---|
| Checkpoint directory | files | `var/models/<model_id>/` with `config.json` (`model_type: slm` for the scratch architecture, the HF type otherwise), `*.safetensors`, tokenizer files, `manifest.json` (`model_id`, `parent_model_id`, `stage`, `snapshot_id`, `git_sha`) | loadable on cuda, mps and cpu; the loader dispatches on `model_type`; LoRA adapters are merged before a version is registered |
| Prompt format | convention | `<doc text>` then the length control `Summary (<length_bucket>):` on its own line, in tokenizer-defined roles for the open path and plain text for the scratch path | one format for SFT data, RL rollouts, evaluation and serving; the version string lives in `manifest.json` |
| `SummarizeRequest` | record | `text: str`; `length_bucket: short \| medium \| long` (default `medium`); `max_new_tokens: int \| None` | input longer than the context window is truncated to its leading part and flagged |
| `SummaryOutput` | record | `summary: str`; `input_tokens: int`; `output_tokens: int`; `truncated: bool`; `model_id` | — |
| Token statistics | typed rows | per snapshot and tokenizer: token counts per document, length histogram, coverage of the vocabulary | written to the ledger as `Metric` rows of a `tokenizer` run |

### evaluation → training, serving

| shape | kind | fields | invariants |
|---|---|---|---|
| `EvalResult` | record + files | `model_id`; `snapshot_id` (an `eval` snapshot); `metrics: dict[str, float]` (`rouge1`, `rouge2`, `rougeL`, `semantic_sim`, `factual_consistency`, `judge_<criterion>`, `reward`, `len_ratio`); `per_sample_ref: str` (parquet under `var/reports/<run_id>/`); `report_ref: str` (`index.html` there) | one per (model, eval snapshot, judge); stored on `ModelVersion.eval` |
| `GateVerdict` | record | `candidate_model_id`; `incumbent_model_id: str \| None`; `passed: bool`; `deltas: dict[str, float]`; `reason: str` | `passed` requires `reward` up and `factual_consistency` not down beyond tolerance; a pass sets `promoted` |
| Sample rows | typed rows | `doc_id`, document text, reference summary, one summary per `model_id` compared | the dashboard's side-by-side view reads these |

## Shared conventions

- **Database.** One Postgres database `slm`. Only `store` opens a connection; every other
  package reaches the schema through `store`'s public surface. DSN in `STORE_DSN`, default
  `postgresql://localhost:5433/slm` (the port the Mac's Postgres listens on). SQLAlchemy 2 +
  psycopg 3, migrations under `store`.
- **Files.** All large files live under one root, `STORE_VAR_DIR` (default `./var`,
  gitignored): `raw/`, `snapshots/`, `models/`, `reports/`, `spool/`, `cache/`. The root may
  be an external disk. Snapshot and model directories are what `rsync` moves to and from the
  GPU box; Postgres stays on the Mac and the ledger spool covers the box (see `store`).
- **Config.** One `pydantic-settings` class per package, env prefix `STORE_`, `JUDGE_`,
  `CORPUS_`, `LM_`, `TRAINING_`, `EVALUATION_`, `SERVING_`; `.env` at the repo root, never
  committed. Secrets are `SecretStr` fields and are never logged or written to the ledger.
- **External sources.** Credentials: `JUDGE_ANTHROPIC_API_KEY`, `JUDGE_OPENAI_API_KEY` (not
  held yet — the adapter is built and exercised once a key exists), `JUDGE_QWEN_BASE_URL` (an
  OpenAI-compatible server: vLLM on the GPU box, `mlx_lm.server` or Ollama on the Mac),
  `CORPUS_HF_TOKEN` and `LM_HF_TOKEN` for Hugging Face. Datasets are not yet chosen: none is
  probed; `corpus`'s sections probe their sources at their PROBE step, and each probe doc
  under `docs/sources/` is added here by `sync-plan`.
- **Budget.** Every API call is charged to a `Run`; `judge` refuses a call that would take a
  run past `JUDGE_RUN_CAP_USD` (`BudgetExceeded`), and reports month-to-date API spend against
  the $50 ceiling. Qwen self-hosted costs `usd = 0`; GPU hours are `Cost` rows of kind
  `gpu-hours` written by `training`.
- **Splits.** `Document.split` is assigned once at ingest: from the dataset's own split when it
  has one, else `test` if `sha256(doc_id) mod 100 < 2`, `val` if `< 5`, else `train`. Dedup
  runs before snapshots, so a near-duplicate of a `test` document does not survive into
  `train`. A snapshot never re-assigns a split; an `eval` snapshot is a fixed subset of `test`.
- **Language.** `lang` is ISO 639-1 from language detection at cleaning; v1 snapshots select
  `en` only (D4).
- **Text.** Cleaned text is Unicode NFC, `\n` newlines, paragraphs separated by one blank line,
  no HTML. `content_hash` is sha256 of the cleaned text. Token counts are per tokenizer and
  live in `lm`'s statistics, not on `Document`.
- **Length buckets.** By reference-summary word count: `short` ≤ 60, `medium` 61–150,
  `long` > 150. Assigned when a pair is stored; the same labels drive the prompt format.
- **Judge scale.** Each rubric criterion is an integer 1–5; `reward` is a float in `[0, 1]`
  per the `Judgement` invariant. A rubric change bumps `rubric_version`, which invalidates the
  cache by key.
- **IDs.** `doc_id` = first 32 hex chars of sha256 over `source_id`, NUL, the source's own key
  (URL, arXiv id, dataset row id), so re-ingest is idempotent. `pair_id` likewise over
  `doc_id`, NUL, `origin`, NUL, summary hash. `snapshot_id` is `<name>-v<n>`. `run_id` and
  `model_id` are `<kind>-<YYYYMMDD-HHMMSS>-<4 hex>`.
- **Time.** UTC everywhere; tz-aware `datetime` in Python, `timestamptz` in Postgres,
  ISO-8601 in files.
- **Reproducibility.** A `Run` records its full config, `snapshot_id`, `base_model_id`, `seed`
  and `git_sha`; a `ModelVersion` records its `run_id` and parent. Re-running a run's config
  against the same snapshot and base is the reproduction.
- **Errors.** Each package defines one exception base at its top level (`StoreError`,
  `JudgeError`, …); cross-package callers catch the provider's base or a named subclass
  (`BudgetExceeded`). No shared base class.
- **Logging.** `loguru`; structured keys `run_id`, `pkg`, `section`, `model_id`, `snapshot_id`
  where they apply; no secrets, no document text at `INFO`.
- **Devices.** `cuda` if available, else `mps`, else `cpu`, chosen in `lm`. CUDA-only
  dependencies (`vllm`, `flash-attn`, `bitsandbytes`) live in an optional extra `gpu` of the
  package that needs them; `uv sync --all-packages` on the Mac installs none, and every
  package's tests run on the Mac.
- **Stage order.** pretrain or cpt → sft → rl. `training` ends every stage run with
  `evaluation`'s gate; only a promoted version is the next stage's parent by default.
- **Data on disk vs. in the table.** Postgres holds text and metadata; raw HTML/PDF, snapshot
  shards, checkpoints and reports are files referenced by path. No blobs in Postgres.

## Toolchain

- **Workspace.** `uv`, root `pyproject.toml` per `workspace-scaffold` §1 with
  `name = "slm"`, `requires-python = ">=3.12"` (the venv is CPython 3.13), members
  `packages/*`, one `[tool.uv.sources]` line per package (`store = { workspace = true }`, …).
  `[tool.pytest.ini_options] addopts = "--import-mode=importlib"`. Lint thresholds merged from
  `/Users/connorott/.claude/plugins/cache/cott-plugins/dev-team/1.0.0/pyproject-lint-config.toml`.
- **Package `pyproject.toml`** per `workspace-scaffold` §2: hatchling, `pydantic>=2`,
  `pydantic-settings>=2`, `loguru`, upstream packages by name. Shared runtime libraries:
  `sqlalchemy>=2` + `psycopg[binary]>=3` + `alembic` (store only), `pyarrow` (snapshot
  shards), `torch` + `transformers` + `tokenizers` + `safetensors` (lm, training, evaluation),
  `httpx` (judge). The rest are each package's to choose.
- **Commands** (`workspace-scaffold` §5), from the root:
  `uv sync --all-packages`; `uv run ruff check && uv run ruff format --check`;
  `uv run pylint --disable=all --enable=C0302,R0904 packages/*/src`; `uv run lint-imports`;
  `uv run --package <pkg> pytest packages/<pkg>/tests`; `uv run pytest packages/*/tests`;
  `uv run mkdocs build --strict`.
- **import-linter.** The block under Dependency graph is contract 1; each package's `surface`
  section adds contract 2 and each first section contract 3, per `workspace-scaffold` §3.
- **Docs.** `mkdocs-material` + `mkdocstrings[python]`, `mkdocs.yml` per `workspace-scaffold`
  §4 with `site_name: slm`; Google-style docstrings; cross-references as
  `` [`name`][pkg.module.name] ``.
- **Entry points.** `[project.scripts]` as `<pkg>-<verb> = "<pkg>.cli:<function>"`, e.g.
  `corpus-ingest`, `corpus-snapshot`, `training-run`, `training-search`, `evaluation-run`,
  `serving-api`, `serving-dashboard`, `store-migrate`, `store-import-spool`. No `scripts/`.

## Non-goals

- Multi-user, auth, or deployment beyond the user's Mac and one rented GPU box.
- Paid data sources; commercial use of non-commercial or research-only data.
- Full-length (beyond context window) summarization, multi-GPU training, judge ensembles and
  calibration, export/quantization, PII scrubbing, scheduled ingestion, CI — all **later**.
- MLflow or any second experiment tracker (D2), and any web frontend beyond Streamlit (D1).
- Carrying code forward from the predecessor project.

## Open decisions

- D1 — Dashboard web UI: Streamlit, or a FastAPI-served page, or Gradio?
- D2 — Experiment tracking: own Postgres tables, or MLflow?
- D3 — Ledger transport from the GPU box: spool files imported later, or a tunnel to the Mac's Postgres?
- D4 — v1 language scope: English only, or keep other languages in snapshots?
