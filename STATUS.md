# STATUS.md — Living Progress Tracker

*Update this file at the end of every session. This is the handoff document.*

---

## Current Phase

**Phase 0 — Design complete**  
Next up: Phase 1, Session 1 — local prototype ingestion module

---

## Locked Decisions

These are not up for debate in future sessions. If you want to revisit, open a new ADR in `/docs/`.

| Decision | Choice | Reason |
|---|---|---|
| RAG vs fine-tuning | RAG | Grounded answers, updatable, no retraining |
| Primary embedding model | Azure OpenAI text-embedding-3-small | Azure-native; FedRAMP eligible |
| Local dev embedding | nomic-embed-text via Ollama | Zero cost, runs offline |
| Vector store | pgvector on PostgreSQL | Reuses existing infra, no new managed service |
| Orchestration | LlamaIndex | Document-first RAG abstractions |
| API framework | FastAPI | Lightweight, async, containerizable |
| Local LLM | Llama 3 via Ollama | Dev/test without API cost |
| Production LLM | GPT-4o via Azure OpenAI | Quality + compliance posture |
| Chunk size | 512 tokens, 64 overlap | Starting point; will tune in Phase 3 |
| Deployment target | Azure Container Apps | Less ops than AKS for this workload |
| IaC | Terraform (azurerm) | Consistent with author's existing stack |

---

## Session Log

### Session 0 — 2026-03-29
**Deliverable**: Project scaffolding + design documentation  
**Completed**:
- README.md (hiring manager-facing)
- ARCHITECTURE.md (full pipeline design)
- SESSION_GUIDE.md (working protocol)
- STATUS.md (this file)
- Stack decisions finalized

**Decisions made this session**: All locked decisions above  
**Blockers**: None  
**Next session deliverable**: `ingestion/chunker.py` — load a markdown file, chunk it (512/64), print chunk count and first chunk. No vector store yet, just validate chunking logic.

---

## Blockers

*None currently.*

---

## Open Questions

- Chunking strategy for PDF runbooks vs plain markdown may need to differ — revisit in Phase 3
- Reranker (Cohere vs local cross-encoder) — decide in Phase 2 based on latency budget
- Whether to support Confluence live API vs export-only — defer to Phase 2

---

## Metrics to Track (Phase 1 end)

- Retrieval precision on 10 test queries (manually scored)
- Average query latency (target: <3s local, <1.5s Azure)
- Ingestion throughput (docs per minute)
