# Running ReguLense

## 1. Quick start (Docker)

```bash
cp backend/.env.example backend/.env    # set OPENAI_API_KEY
docker compose up --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:8080 |
| API | http://localhost:8000 (`/docs` for OpenAPI) |
| Postgres + pgvector | `localhost:5433` (user/db `regulense`) |
| Redis | `localhost:6379` |

The containers start with an empty database. Build the corpus next (step 3).

## 2. Manual setup (no Docker)

Requirements: Python 3.14 (what CI and the Docker image use), Node 22, Postgres 16+ with the `vector` extension, and optionally Redis.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r backend/requirements.txt
createdb regulense && psql regulense -c "CREATE EXTENSION IF NOT EXISTS vector;"
cp backend/.env.example backend/.env    # set OPENAI_API_KEY and PG* vars (or DATABASE_URL)

cd backend && python -m app.main        # API on :8000, terminal 1
cd frontend && npm install && npm run dev   # UI on :5173, terminal 2
```

Redis is optional. If `REDIS_URL` is unset or unreachable, the cache degrades cleanly to Postgres-only.

## 3. Build the corpus

The PDFs listed in `backend/corpus_urls.txt` are public DHA / DoH / MOHAP documents. The pipeline is idempotent, so every step can be re-run safely.

```bash
cd backend
pip install -r requirements-ingestion.txt
python -m ingestion.download             # corpus_urls.txt -> dataset/
python -m ingestion.ingest               # parse printed doc_code / version / effective_date
python -m ingestion.apply_manual_fixes   # OCR-verified metadata for PDFs with no parseable footer
python -m ingestion.load_db              # documents rows + superseded flags (idempotent on sha256)
python -m ingestion.rechunk              # Docling structure-aware chunks + embeddings (billed)
```

Optional follow-up question mining, which uses billed LLM calls:

```bash
python -m ingestion.extract_mining_inputs --all
python -m ingestion.mine_suggestions
python -m ingestion.load_suggestions suggestions/question_suggestions/*.jsonl
```

Bring your own corpus: point `corpus_urls.txt` at your own PDFs. Any document that prints a code, version and effective date gets version tracking for free. Documents without them land in `needs_manual.json` instead of being guessed. See [ARCHITECTURE.md §2](ARCHITECTURE.md#2-ingestion-pipeline).

## 4. CLI

```bash
cd backend
python -m cli.ask "What are the licensing requirements for a telehealth provider in Dubai?"
python -m cli.diff_versions "DHA/HRS/HLD/MA-2"
python -m cli.manage_documents list
```

## 5. Configuration

| Variable | Default | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | (required) | Embeddings and generation |
| `NARAROUTER_API_KEY` | (unset) | Fallback chat provider on rate limit or outage |
| `DATABASE_URL` or `PG*` | (none) | Postgres connection, set one or the other |
| `REDIS_URL` | (unset) | L1 exact-key answer cache |
| `CACHE_HIT_THRESHOLD` | `0.92` | Cosine floor for a paraphrase cache hit |
| `CACHE_REDIS_TTL_SECONDS` | `604800` | Redis key TTL |
| `DAILY_OPENAI_CALL_CAP` | `300` | Hard daily ceiling on paid API calls |
| `LANGSMITH_*` | (unset) | Optional tracing |

### Answer cache

`/ask` and `/diff-followup` answers are cached. Postgres (`answer_cache`, `diff_cache`) is the durable source of truth, and Redis is the fast exact-key layer. On every boot, all cached rows are reloaded into Redis (`Warmed cache: N ask rows, M diff rows`). If you see `0/0` with no error, Redis isn't connected. It doesn't mean the cache is empty.

Two deployment gotchas:
- **Render installs from the repo-root `requirements.txt`**, not `backend/requirements.txt`. Keep the two in sync.
- **Upstash has two credential types.** Use the native-protocol URL with the `rediss://` scheme (TLS). The REST URL/token won't work.

To pre-warm known demo questions, edit `backend/scripts/demo_questions.json`, then run `python -m scripts.seed_answer_cache`. It makes billed calls.

## 6. Tests

```bash
cd backend
pip install -r requirements-dev.txt
TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/test_regulense pytest
```

Tests run against a real Postgres + pgvector, not mocks. Only the two OpenAI-facing calls are stubbed, so CI needs no API key.
