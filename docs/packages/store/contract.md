# store — package contract

Written by `/dev-team:plan-package store` on 2026-09-28 against `docs/architecture.md` (2026-09-28). Greenfield: intended code until built.

## Purpose

`store` is the lowest package: the one Postgres schema, its migrations, and the `var/` file
tree. Every other package reads and writes documents, pairs, snapshots and the ledger through
its public surface; nothing else opens a connection. It provides every shape in the repo
contract's **store →** boundary: `Source`, `Document`, `SummaryPair`, `Snapshot`, `Run`,
`Metric`, `Cost`, `JudgeScore`, `ModelVersion`, the `Catalog` query and the ledger spool (D3).

Covered brief capabilities (Notes verbatim):

- **Document database** — `schema`, `documents`: "retrieve and store all of this data in a full database" — documents, summary pairs, metadata.
- **Raw originals** — `documents`: Original HTML / PDF kept so cleaning can be re-run without re-downloading.
- **Versioned snapshots** — `snapshots`: Frozen, versioned datasets with fixed train / val / test splits that training reads.
- **Data catalog** — `catalog`: Browse and filter the database by source, domain, length, quality.
- **Experiment tracking** — `ledger`: Every run's config, data version and metrics recorded and comparable.
- **Cost tracking** — `ledger`: GPU hours and API spend reported per run.
- **Model registry** — `ledger`: Versioned trained models, each with its eval results and lineage.

No *later* capability lands in `store`; `upsert_documents` is idempotent on `doc_id` so that
`corpus`'s later resumable and scheduled ingestion are not ruled out.

## Sections

| section | responsibility | path | owner doc | builds with | depends on | source |
|---|---|---|---|---|---|---|
| `schema` | The pydantic records (the repo shapes), the SQLAlchemy 2 table definitions for `sources`, `documents`, `summary_pairs`, `snapshots`, `runs`, `metrics`, `costs`, `judge_scores`, `model_versions`, `spool_events`; alembic migrations; engine and session factory; reachability check; `StoreError` and `StoreUnavailable`. | `packages/store/src/store/schema/` | `docs/packages/store/design/schema.md` | — | — | — |
| `documents` | Sources, documents and summary pairs: id derivation, split assignment, idempotent upsert, cleaning updates, length buckets, raw originals under `var/raw/`. | `packages/store/src/store/documents/` | `docs/packages/store/design/documents.md` | — | `schema` | — |
| `catalog` | Read-only filtered queries over documents and pairs: keyset iteration, counts by field, length histograms. The dashboard and the data-validation classes read only this. | `packages/store/src/store/catalog/` | `docs/packages/store/design/catalog.md` | — | `schema` | — |
| `snapshots` | Build, register, read and verify immutable parquet snapshots under `var/snapshots/`, honoring each document's `split`. | `packages/store/src/store/snapshots/` | `docs/packages/store/design/snapshots.md` | — | `schema`, `catalog` | — |
| `ledger` | Runs, metrics, costs, the judge-score cache and the model registry (D2); the write-through-or-spool recorder and the spool importer (D3). | `packages/store/src/store/ledger/` | `docs/packages/store/design/ledger.md` | — | `schema` | — |
| `surface` | The package's pipelines (§4) and public surface (§5): `__init__.py`, `configs.py`, `cli.py`. | `packages/store/src/store/` | `docs/packages/store/design/surface.md` | — | `schema`, `documents`, `catalog`, `snapshots`, `ledger` | — |

## Section interfaces

Every record named below is the repo-contract shape of that name, a pydantic model defined once
in `schema` beside the `Literal` aliases (`Domain`, `Split`, `DocStatus`, `LengthBucket`,
`RunKind`, `Stage`, `LicenseClass`). Each function opens its own session and transaction.

### schema

```python
class StoreError(Exception): ...            # the package's exception base
class StoreUnavailable(StoreError): ...     # Postgres unreachable and the call cannot spool
def get_engine(settings: StoreSettings | None = None) -> Engine       # cached per DSN
def session() -> AbstractContextManager[Session]   # one transaction per with-block; rollback on error
def db_reachable(timeout_s: float | None = None) -> bool
def migrate(revision: str = "head") -> None        # alembic upgrade, programmatic, idempotent
def current_revision() -> str | None
```

### documents

```python
def make_doc_id(source_id: str, source_key: str) -> str           # repo IDs convention
def make_pair_id(doc_id: str, origin: str, summary: str) -> str
def assign_split(doc_id: str, dataset_split: Split | None = None) -> Split   # repo Splits convention
def length_bucket(summary: str) -> LengthBucket                    # repo Length buckets convention
def upsert_source(source: Source) -> Source
def get_source(source_id: str) -> Source | None
def list_sources() -> list[Source]
def put_raw(doc_id: str, payload: bytes, ext: str) -> str         # writes var/raw/…, returns raw_ref
def read_raw(raw_ref: str) -> bytes
def upsert_documents(docs: Iterable[Document], *, batch_size: int = 1000) -> int   # rows inserted
    # on conflict (doc_id): updates url, title, raw_ref, fetched_at, license_class only;
    # split, source_id, domain are never changed after insert
class DocumentUpdate(BaseModel):   # doc_id plus optional text, lang, char_len, word_len,
    ...                            # content_hash, quality, status, title
def update_documents(updates: Iterable[DocumentUpdate]) -> int     # rows updated; enforces
    # the invariant text non-null iff status == "cleaned"
def get_document(doc_id: str) -> Document | None
def get_documents(doc_ids: Iterable[str]) -> list[Document]
def upsert_pairs(pairs: Iterable[SummaryPair], *, batch_size: int = 1000) -> int
    # length_bucket filled from length_bucket(summary); unknown doc_id raises StoreError
def get_pairs(doc_id: str) -> list[SummaryPair]
```

### catalog

```python
class DocumentFilter(BaseModel):   # every field optional; None means no constraint
    source_ids: list[str]; domains: list[Domain]; splits: list[Split]; statuses: list[DocStatus]
    langs: list[str]; license_classes: list[LicenseClass]; min_chars: int; max_chars: int
    min_quality: float; max_quality: float; has_pairs: bool; pair_origins: list[str]
def count_documents(f: DocumentFilter) -> int
def iter_documents(f: DocumentFilter, *, batch_size: int = 1000) -> Iterator[Document]   # keyset on doc_id
def iter_pairs(f: DocumentFilter, *, batch_size: int = 1000) -> Iterator[tuple[Document, SummaryPair]]
def counts_by(f: DocumentFilter, group: Literal["source_id", "domain", "split", "status", "lang", "license_class"]) -> dict[str, int]
def length_histogram(f: DocumentFilter, field: Literal["char_len", "word_len"],
                     edges: Sequence[int]) -> list[int]           # len(edges) - 1 bin counts
def pair_counts_by(f: DocumentFilter, group: Literal["origin", "length_bucket", "split"]) -> dict[str, int]
```

### snapshots

```python
class SnapshotSpec(BaseModel):
    name: str; kind: Literal["pretrain", "sft", "eval"]; filter: DocumentFilter
    max_docs_per_split: dict[Split, int] | None = None; seed: int = 0
    # kind == "eval" forces filter.splits == ["test"]; sft/eval select via iter_pairs
def build_snapshot(spec: SnapshotSpec) -> Snapshot   # next v<n> for name; shards + manifest, then row
def get_snapshot(snapshot_id: str) -> Snapshot | None
def list_snapshots(kind: str | None = None) -> list[Snapshot]
def snapshot_dir(snapshot_id: str) -> Path                       # var/snapshots/<snapshot_id>
def iter_rows(snapshot_id: str, split: Split, *, batch_size: int = 1000) -> Iterator[pyarrow.RecordBatch]
def verify_snapshot(snapshot_id: str) -> list[str]   # problems; [] means manifest, shards and sha256s agree
```

Shard columns — `pretrain`: `doc_id, source_id, domain, lang, text, char_len, word_len`;
`sft` and `eval`: those plus `pair_id, summary, length_bucket, origin`. A document with several
pairs yields one row per pair.

### ledger

```python
def start_run(kind: RunKind, config: dict, *, snapshot_id: str | None = None,
              base_model_id: str | None = None, seed: int | None = None) -> Run   # id, git_sha, host, started_at filled
def finish_run(run_id: str, status: Literal["done", "failed"]) -> None
def get_run(run_id: str) -> Run | None
def list_runs(*, kind: RunKind | None = None, status: str | None = None, since: datetime | None = None) -> list[Run]
def record_metrics(run_id: str, metrics: Iterable[Metric]) -> None    # append-only
def get_metrics(run_id: str, names: Sequence[str] | None = None) -> list[Metric]   # ordered (name, step)
def record_cost(run_id: str, kind: Literal["gpu-hours", "api"], provider: str, units: float, usd: float) -> None
def run_costs(run_id: str) -> dict[str, float]                       # usd by kind
def month_api_spend(month: str | None = None) -> float               # "YYYY-MM", default current, UTC
def judge_score_key(judge: str, rubric_version: str, document_hash: str, summary_hash: str) -> str
def get_judge_score(key: str) -> JudgeScore | None                   # None when unreachable (a miss)
def put_judge_score(score: JudgeScore) -> None                       # insert-or-ignore on key
def register_model(version: ModelVersion) -> ModelVersion            # model_id generated when empty
def get_model(model_id: str) -> ModelVersion | None
def list_models(*, stage: Stage | None = None, path: str | None = None, promoted: bool | None = None) -> list[ModelVersion]
def set_model_eval(model_id: str, eval: dict) -> None
def promote_model(model_id: str) -> None
def lineage(model_id: str) -> list[ModelVersion]                     # model → … → base
class SpoolImport(BaseModel): files: int; events: int; applied: int; skipped: int
def import_spool(spool_dir: Path | None = None) -> SpoolImport       # idempotent on (run_id, seq)
```

Spool semantics: with `ledger_mode = auto` a write goes to Postgres when `db_reachable()`, else
appends `{"seq", "event", "payload"}` to `var/spool/<run_id>.jsonl` (`event` one of `run`,
`metric`, `cost`, `model_version`, and `judge_score` under D5); `db` never spools, `spool`
never connects. A read other than `get_judge_score` raises `StoreUnavailable` when unreachable.
`import_spool` records each applied `(run_id, seq)` in `spool_events` and skips it thereafter.

## Pipelines

- **Snapshot build** — trigger: `corpus-snapshot` calling `build_snapshot(spec)`. `catalog`
  (`iter_documents` / `iter_pairs` over `spec.filter`, per split) → `snapshots` writes
  `var/snapshots/<name>-v<n>/<split>/part-*.parquet` and `manifest.json` → `schema` row last,
  in one transaction with the version allocation. A failure before the row leaves no row and
  removes the partial directory; an existing version is never overwritten.
- **Spool import** — trigger: `store-import-spool [--spool-dir]` after the `rsync` from the
  GPU box. Each `var/spool/*.jsonl` → `ledger.import_spool` applies events in `seq` order to
  the ledger tables → `SpoolImport` printed. A malformed line stops that file and is reported;
  other files proceed. Files are never deleted.

## Public surface (intent)

| name | section | consumer |
|---|---|---|
| `Source`, `Document`, `SummaryPair`, `Snapshot`, `Run`, `Metric`, `Cost`, `JudgeScore`, `ModelVersion`, `DocumentUpdate`, `DocumentFilter`, `SnapshotSpec`, the `Literal` aliases | `schema`, `documents`, `catalog`, `snapshots` | every package |
| `StoreError`, `StoreUnavailable` | `schema` | every package |
| `session`, `db_reachable`, `migrate`, `current_revision` | `schema` | `serving` (dashboard health), tests of every package, `store-migrate` |
| `make_doc_id`, `make_pair_id`, `assign_split`, `length_bucket`, `upsert_source`, `get_source`, `list_sources`, `put_raw`, `read_raw`, `upsert_documents`, `update_documents`, `get_document`, `get_documents`, `upsert_pairs`, `get_pairs` | `documents` | `corpus` |
| `count_documents`, `iter_documents`, `iter_pairs`, `counts_by`, `length_histogram`, `pair_counts_by` | `catalog` | `corpus` (validation classes), `serving` (dashboard), `evaluation` (sample viewer) |
| `build_snapshot`, `list_snapshots`, `get_snapshot`, `snapshot_dir`, `iter_rows`, `verify_snapshot` | `snapshots` | `corpus` (build), `lm`, `training`, `evaluation` (read), `store-verify` |
| `start_run`, `finish_run`, `get_run`, `list_runs`, `record_metrics`, `get_metrics`, `record_cost`, `run_costs`, `month_api_spend` | `ledger` | `corpus`, `judge`, `lm`, `training`, `evaluation`, `serving` |
| `judge_score_key`, `get_judge_score`, `put_judge_score` | `ledger` | `judge` |
| `register_model`, `get_model`, `list_models`, `set_model_eval`, `promote_model`, `lineage` | `ledger` | `training`, `evaluation`, `serving` |
| `import_spool` | `ledger` | `store-import-spool` |
| `StoreSettings` | `surface` | tests of every package |

CLI commands (`[project.scripts]`): `store-migrate` (`migrate`), `store-import-spool`
(**Spool import**), `store-verify <snapshot_id>` (`verify_snapshot`, exit 1 on problems).

## Consumes

No table: `store` is the lowest layer and imports nothing from a sibling package.

## Package conventions

- **Settings.** `StoreSettings` (prefix `STORE_`): `dsn` (default
  `postgresql+psycopg://localhost:5433/slm`), `var_dir` (default `./var`), `ledger_mode`
  (`auto | db | spool`, default `auto`), `db_timeout_s` (`2.0`), `shard_rows` (rows per
  parquet shard, `50_000`), `spool_dir` (`<var_dir>/spool`). Every `var/` path derives from
  `var_dir`; `raw/`, `snapshots/`, `spool/` are created on first write.
- **Raw originals.** `raw_ref` is relative to `var_dir`: `raw/<source_id>/<doc_id[:2]>/<doc_id>.<ext>`.
- **Tables.** Snake-case names as listed under `schema`; `timestamptz` for every datetime;
  `jsonb` for `config`, `filter`, `counts`, `scores`, `eval`; `Document.text` is `text`
  (TOAST-compressed, no blobs). Indexes back every `DocumentFilter` field and every
  `(run_id, …)` lookup; the `schema` design lists them.
- **Migrations.** One alembic revision per schema change, under `schema/migrations/`; tests
  build the schema by `migrate()`, never `metadata.create_all`, against a throwaway database
  named by `STORE_DSN`; a test that cannot reach one skips.
- **Snapshot manifest.** `manifest.json`: `snapshot_id`, `kind`, `filter`, `counts`,
  `shards: {<split>/<file>: sha256}`, `git_sha`, `created_at`. `verify_snapshot` is the
  check the GPU box runs after `rsync`.

## Open decisions

- D2 — Experiment tracking: own Postgres tables, or MLflow? (assumption: own tables)
- D3 — Ledger transport from the GPU box: spool files or a tunnel? (assumption: spool)
- D5 — Should the spool also carry `JudgeScore` events so RL on the GPU box fills the cache? (assumption: yes)
