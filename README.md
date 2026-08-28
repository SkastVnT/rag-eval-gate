# rag-eval-gate

A RAG serving stack whose CI **blocks merges on measured answer quality**, not
just on tests passing.

Every pull request runs the retrieval pipeline over an evaluation set and fails
if any of three metrics regresses below its floor:

```yaml
- name: Run evaluation
  run: |
    python -m eval.run_eval \
      --dataset eval/datasets/sample.json \
      --judge heuristic \
      --min-context-relevance 0.3 \
      --min-groundedness     0.3 \
      --min-answer-relevance 0.3
```

The three metrics are implemented in this repo (`libs/ragops/metrics/`), not
pulled from a framework:

| Metric | Question it answers | Source |
|---|---|---|
| Context relevance | Did retrieval fetch passages that bear on the question? | `context_relevance.py` |
| Groundedness | Is every claim in the answer supported by those passages? | `groundedness.py` |
| Answer relevance | Does the answer actually address what was asked? | `answer_relevance.py` |

## Why this is not a tutorial RAG

The common shape is: chunk a PDF, embed it, do a similarity search, stuff the
top-k into a prompt. This repo differs on four axes.

**Retrieval is hybrid, not vector-only.** `libs/retrieval/` combines dense and
lexical search (`lexical_search.py`) through rank fusion (`fusion.py`), then
reorders with `rerankers.py`. Query rewriting lives in `transforms/`. There is
also a graph-based path in `libs/graph_rag/`.

**Untrusted input is treated as untrusted.** `libs/guardrails/` is a pipeline of
eight modules covering prompt injection (`injection.py`), PII (`pii.py`), output
filtering (`output.py`), sanitisation, and a trust model (`trust.py`) — applied
to both user input and retrieved documents, since retrieved text is attacker
-controllable in any RAG system.

**Ingestion is a worker, not a request handler.** `apps/api/` and `apps/worker/`
are separate processes; parsing and chunking happen off the request path.
Schema changes go through Alembic migrations in `alembic/`.

**Providers are swappable.** LLM and embedding backends are selected by env var
(`LLM_PROVIDER`, `EMBEDDING_PROVIDER`) behind an interface in
`libs/core/providers/`, so OpenAI is a default rather than a dependency.

## Scale

| | |
|---|---|
| Python | ~25,800 lines across 144 files |
| Tests | 640 passing across 17 modules |
| Library modules | 9 (`agent`, `auth`, `core`, `embedding`, `graph_rag`, `guardrails`, `ingestion`, `ragops`, `retrieval`) |
| Storage | PostgreSQL 16 + pgvector, Redis 7, MinIO |
| Runtime | FastAPI, Python 3.12, Docker Compose |

## Honest limits

- The CI judge is heuristic, not an LLM judge — deterministic and API-free so
  CI can gate on it, but a weaker signal than an LLM judge. It scores query/
  context term coverage with prefix-and-stem matching; it does not understand
  meaning. The thresholds (0.3) are regression floors, not quality targets.
- Context relevance is scored against context supplied by the dataset, so the
  gate measures the *judge and generator* path end to end but does not exercise
  a live index. Wiring the gate to real retrieval needs Postgres + pgvector as
  a CI service.
- `eval/datasets/sample.json` is a small seed set. Growing it is the main lever
  for making the gate meaningful.
- Extracted from a larger personal workspace, so early history is compressed
  into a handful of bulk commits.

## Quick Start

### Prerequisites

- Python 3.12+
- Docker & Docker Compose
- An OpenAI API key (or other supported LLM provider)

### Option A: Local Development (recommended)

Infrastructure in Docker, Python apps running locally with hot-reload.

**Linux/Mac:**
```bash
cd rag/
make setup        # creates venv + .env
# Edit .env → set LLM_API_KEY and EMBEDDING_API_KEY
make infra-up     # start postgres, redis, minio
make dev          # run API with auto-reload (port 8000)
# In another terminal:
make worker       # run background worker
```

**Windows (PowerShell):**
```powershell
cd rag\
.\tasks.ps1 setup        # creates venv + .env
# Edit .env → set LLM_API_KEY and EMBEDDING_API_KEY
.\tasks.ps1 infra-up     # start postgres, redis, minio
.\tasks.ps1 dev           # run API with auto-reload (port 8000)
# In another terminal:
.\tasks.ps1 worker        # run background worker
```

### Option B: Full Docker

Everything in containers. No local Python needed.

```bash
cp .env.example .env      # edit with your API keys
docker compose up -d --build
# API at http://localhost:8000
# MinIO console at http://localhost:9001
```

### Verify

```bash
# Health check
curl http://localhost:8000/health
# Expected: {"status":"healthy","postgres":true,"redis":true,"minio":true}

# Upload a document
curl -X POST http://localhost:8000/api/v1/documents/ -F "file=@README.md"

# Ask a question
curl -X POST http://localhost:8000/api/v1/query/ \
  -H "Content-Type: application/json" \
  -d '{"query": "What tech stack does this project use?"}'
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Infrastructure health check |
| POST | `/api/v1/documents/` | Upload & ingest a document |
| GET | `/api/v1/documents/` | List all documents |
| DELETE | `/api/v1/documents/{id}` | Delete a document + chunks |
| POST | `/api/v1/query/` | RAG query (retrieve + generate) |

## Common Commands

| Command | Linux/Mac | Windows |
|---------|-----------|---------|
| Run tests | `make test` | `.\tasks.ps1 test` |
| Lint | `make lint` | `.\tasks.ps1 lint` |
| Auto-fix | `make lint-fix` | `.\tasks.ps1 lint-fix` |
| Format | `make format` | `.\tasks.ps1 format` |
| All checks | `make check` | `.\tasks.ps1 check` |
| View logs | `make logs` | `.\tasks.ps1 logs` |
| DB migrate | `make db-upgrade` | `.\tasks.ps1 db-upgrade` |

## Roadmap

- [x] MVP: ingest → embed → search → generate
- [x] Dockerized api + worker services
- [x] Structured logging (text/JSON)
- [x] Task runner (Makefile + PowerShell)
- [ ] Hybrid search (BM25 + vector via tsvector)
- [ ] Reranking (cross-encoder)
- [ ] PDF / DOCX ingestion
- [ ] Embedding cache (Redis)
- [ ] Query transformation (HyDE, decomposition)
- [ ] GraphRAG
- [ ] Agentic RAG
- [ ] ReBAC / multi-tenant access control
- [ ] RAGOps (evaluation, monitoring, drift detection)
