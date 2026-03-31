# STATUS.md — Living Progress Tracker

*Update this file at the end of every session. This is the handoff document.*

---

## Current Phase

**Phase 1 — Local Prototype (GCP VM + Docker Compose)**
Core scaffold complete. Stack is runnable. Next: deploy to GCP VM and run end-to-end.

---

## Locked Decisions

These are not up for debate in future sessions. Open a new ADR in `/docs/` to revisit.

| Decision | Choice | Reason |
|---|---|---|
| RAG vs fine-tuning | RAG | Grounded answers, updatable, no retraining |
| Phase 1 embedding | nomic-embed-text via Ollama | Zero cost, no API dependency |
| Phase 2 embedding | text-embedding-3-small via OpenAI API | Quality + cost balance |
| Phase 1 vector store | ChromaDB (Docker) | Zero config, validates pipeline cheaply |
| Phase 2 vector store | Cloud SQL pgvector | GCP-native, no new managed service category |
| Orchestration | LlamaIndex | Document-first RAG abstractions |
| API framework | FastAPI | Lightweight, async, containerizable |
| Phase 1 LLM | Llama 3 via Ollama | Dev/test without API cost |
| Phase 2 LLM | GPT-4o via OpenAI API | Quality + reasonable cost |
| Chunk size | 512 tokens, 64 overlap | Starting point — Pixel benchmarks in Phase 3 |
| Compute | GCP Compute Engine e2-standard-2 | Laptop resource constraint (ADR-001) |
| Phase 2 deployment | Cloud Run or GKE | GCP-native, replaces Docker Compose |
| Phase 2 IaC | Terraform (google provider) | Consistent with infra-as-code practice |
| Backend toggle | USE_LOCAL_MODELS env var | Same code, swappable backends |

---

## Session Log

### Session 0 — 2026-03-29
**Deliverable**: Project scaffolding + design documentation
**Completed**:
- README.md (hiring manager-facing)
- ARCHITECTURE.md (full pipeline design)
- SESSION_GUIDE.md (working protocol)
- STATUS.md (this file)
- security_compliance_considerations.docx
- BUILDPLAN.md + rag-dev-team skill
**Decisions made**: All initial locked decisions (Azure stack)
**Blockers**: None

---

### Session 1 — 2026-03-31
**Deliverable**: Full Phase 1 scaffold — all runnable files
**Completed**:
- `requirements.txt` — pinned dependencies
- `.env.example` — all config variables documented
- `docker-compose.yml` — ollama + ollama-setup + chromadb + api services with healthchecks
- `api/Dockerfile`
- `ingestion/chunker.py` — SentenceSplitter, 512/64, metadata preserved
- `ingestion/embedder.py` — Ollama/OpenAI toggle, ChromaDB storage, batch embedding
- `query/retriever.py` — cosine similarity search, category filter, score normalization
- `query/prompt.py` — context assembly, LLM call, confidence scoring heuristic
- `api/main.py` — /ingest, /query, /health, /sources with input validation
- `tests/test_chunker.py` — 6 tests, chunker gate
- `tests/test_api.py` — 7 tests, all endpoints mocked
- `tests/sample_docs/runbook_db_incident.md` — test fixture
- `infra/gcp/setup_vm.sh` — one-time VM provisioning script
- `Makefile` — up/down/logs/health/ingest/query/test/vm/ssh/firewall targets

**Decisions made this session**:
- GCP over local/Azure (ADR-001) — laptop resource constraint
- ChromaDB for Phase 1 (not pgvector) — zero config, faster to validate
- OpenAI API for Phase 2 (not Azure OpenAI) — simpler auth, GCP-native path

**Blockers**: None
**Next session**: Deploy to GCP VM — run `setup_vm.sh`, `make up`, validate `make health`
returns 200, ingest `runbook_db_incident.md`, run first live query.

---

## Blockers

*None currently.*

---

## Open Questions

| Question | Decision needed by |
|---|---|
| Chunking strategy may differ for PDFs vs markdown | Phase 3 — Pixel benchmarks |
| Reranker choice: Cohere vs local cross-encoder | Phase 2 — decide on latency budget |
| Confluence: live API vs export-only | Phase 2 |
| Obsidian vault as ingestion source — graph-augmented retrieval, frontmatter metadata, MCP write-back | Phase 3 — high value for compliance-heavy environments; requires `obsidian_preprocessor.py` + `graph_retriever.py` |
| Path traversal validation on ingest endpoint | Phase 2 — CodeGuard requirement before cloud deploy |
| Input sanitization layer for prompt injection | Phase 2 — CodeGuard requirement |

---

## Metrics to Track (Phase 1 end — Session 7)

- Retrieval precision: 10 manually scored queries, target ≥ 3.5/5 (Pixel gate)
- Query latency: target < 3s on GCP e2-standard-2
- Ingestion throughput: docs per minute on sample corpus
- Stack startup time: `docker compose up` → `/health` 200 (first run vs warm)
