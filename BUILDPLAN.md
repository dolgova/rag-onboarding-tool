# BUILDPLAN.md — RAG Onboarding Tool

Project build plan. Phases, owners, parallelism, and honest assessment of
where Claude carries the work vs. where human judgment is irreplaceable.

---

## How to Read This

Each phase has:
- **Deliverables** — specific shippable outputs
- **Owner** — which builder leads
- **Claude leverage** — how much Claude can execute vs. advise on
- **Parallel tracks** — what can run simultaneously

---

## The Build Team

| Handle | Role | What they own |
|---|---|---|
| `@joe` | Platform PM & Architect | Specs, ADRs, pipeline design, coordination, post-mortems |
| `@sara` | Pipeline & API Engineer | Python — ingestion, query, FastAPI, pytest |
| `@codeguard` | Security & Compliance Reviewer | Code review + zero-trust/FedRAMP compliance gate |
| `@pixel` | RAG Quality Engineer | Retrieval benchmarks, chunking eval, prompt quality, phase gates |

Full team charter: `skill/rag-dev-team/SKILL.md`

---

## Phase Overview

```
Phase 0 — Foundation          ✅ Complete
Phase 1 — Local Prototype     🔄 Scaffold complete — deploying to GCP
Phase 2 — GCP Production      ⬜ Planned
Phase 3 — Quality & Obsidian  ⬜ Planned
Phase 4 — Hardening           ⬜ Planned
```

---

## Phase 0 — Foundation
**Status**: ✅ Complete

All design decisions locked. Four project files in place. Security document written.
Build plan and dev team skill created.

**Outputs**: README, ARCHITECTURE, SESSION_GUIDE, STATUS, BUILDPLAN,
security_compliance_considerations.docx, rag-dev-team/SKILL.md

---

## Phase 1 — Local Prototype (GCP VM + Docker Compose)

**Goal**: Working end-to-end RAG loop on GCP. Prove the pipeline before cloud costs.

**Claude leverage**: 🟢 High — Python + LlamaIndex + ChromaDB is well within
Claude's reliable generation range. ~85% of code came from Claude in Session 1.
Human work: running it, validating output, catching GCP-specific issues.

### ✅ Session 1 — Full scaffold (COMPLETE)

All files generated and committed. See Session Log in STATUS.md for full list.

**Files produced**:
```
requirements.txt
.env.example
docker-compose.yml
api/Dockerfile
api/main.py             POST /ingest, POST /query, GET /health, GET /sources
ingestion/chunker.py    SentenceSplitter 512/64, metadata preserved
ingestion/embedder.py   Ollama/OpenAI toggle, ChromaDB batch storage
query/retriever.py      Cosine similarity, category filter, score normalization
query/prompt.py         Context assembly, LLM call, confidence heuristic
tests/test_chunker.py   6 tests
tests/test_api.py       7 tests (mocked backends)
tests/sample_docs/      runbook_db_incident.md fixture
infra/gcp/setup_vm.sh   One-time VM provisioning
Makefile                up/down/logs/health/ingest/query/test/vm/ssh/firewall
```

**CodeGuard flags from Session 1 (tracked, not blocking Phase 1)**:
- Path traversal validation missing on ingest endpoint — fix in Phase 2
- Prompt injection: user question injected as string — add sanitization in Phase 2

---

### Session 2 — GCP VM deploy + first live query
**Owner**: Andrey (runs) | @sara (supports if errors)
**Depends on**: Session 1 ✅

```
Deliverable: Stack running on GCP VM, /health returns 200,
runbook_db_incident.md ingested, one live query returning a grounded answer.

Steps:
  make vm NAME=rag-dev ZONE=us-central1-a
  make ssh NAME=rag-dev
  bash infra/gcp/setup_vm.sh     ← one-time, ~10 min
  cp .env.example .env
  make up                         ← models pull automatically, ~10 min first run
  make health
  cp tests/sample_docs/runbook_db_incident.md docs_ingest/
  make ingest FILE=runbook_db_incident.md CATEGORY=runbook
  make query Q="What is the P1 database incident response process?"

Success criteria:
  - /health → 200
  - ingest → chunks_stored > 0
  - query → confidence: high or medium, sources cite runbook_db_incident.md
```

**Parallel**: @pixel drafts the Phase 1 eval query set (10 questions) while VM deploys.

---

### Session 3 — Run tests on GCP VM
**Owner**: @sara | @codeguard (reviews test output)
**Depends on**: Session 2

```
Deliverable: pytest tests/test_chunker.py passes on VM without Ollama dependency.
pytest tests/test_api.py passes (mocked — no live services needed).
Both gates green before Phase 1 eval.
```

---

### Session 4 — Phase 1 evaluation (Pixel gate)
**Owner**: @pixel (leads) | Andrey (scores)
**Depends on**: Session 2 (live stack)

```
Deliverable: docs/eval/phase1_eval.md

10 test queries against runbook_db_incident.md + 2 additional ingested docs.
Scoring rubric (human-scored, not LLM-scored):
  Relevance    0-2  (did it retrieve the right content?)
  Accuracy     0-2  (is the answer correct?)
  Citation     0-1  (did it cite the source?)
  Max score: 5

Target: average ≥ 3.5/5 across all 10 queries.

Gate: Phase 2 does NOT start until this passes.
If score < 3.5: @pixel files findings → @joe specs chunking/retrieval adjustments
                → iterate before moving to Phase 2.
```

---

## Phase 2 — GCP Production

**Goal**: Replace Docker Compose stack with GCP-native services.
Same pipeline, cloud-grade deployment.

**Claude leverage**: 🟡 Medium-High — Terraform for GCP is well within Claude's range,
but Cloud SQL pgvector setup, Cloud Run config, and IAP auth often need iteration
against real GCP error output. Expect 60-70% from Claude.

### Parallel tracks (can run simultaneously after Phase 1 gate)

**Track A — Backend swap** (@sara)
- Session 5: `ingestion/store.py` abstraction layer — ChromaDB and pgvector implement same interface
- Session 6: Swap Ollama → OpenAI API (`USE_LOCAL_MODELS=false` fully functional end-to-end)
- Session 7: Swap ChromaDB → Cloud SQL pgvector

**Track B — GCP Infrastructure** (@joe specs, @sara implements)
- Session 8: `infra/terraform/modules/cloudsql/` — Cloud SQL PostgreSQL + pgvector extension
- Session 9: `infra/terraform/modules/cloudrun/` — Cloud Run service + env var injection
- Session 10: `infra/terraform/modules/secrets/` — GCP Secret Manager for API keys

**Track C — Auth & Security** (@codeguard reviews, @sara implements)
- Session 11: Path traversal fix + input sanitization layer (CodeGuard Phase 1 flags)
- Session 12: Google IAP or JWT middleware on FastAPI
- Session 13: Role-based access — query/compliance/admin roles enforced at retrieval layer
- Session 14: Cloud Logging — all queries/ingest/delete events logged with full payload

**Gate**: @codeguard full compliance review before any endpoint is public-facing.
Both Phase 1 CodeGuard flags must be resolved. Zero-trust checklist signed off.

---

## Phase 3 — Quality & Obsidian Integration

**Goal**: Make retrieval accurate enough for production use in a compliance environment.
Add Obsidian vault as an ingestion path for graph-augmented retrieval.

**Claude leverage**: 🟡 Medium
- Reranker code: Claude writes it
- Obsidian preprocessor: Claude writes it, human validates wikilink resolution
- Chunk size benchmarks: Pixel runs them, human scores results
- Confidence threshold tuning: requires operational data, human judgment

| Session | Deliverable | Owner | Claude leverage |
|---|---|---|---|
| 15 | `query/reranker.py` — Cohere Rerank, falls back to top-K without API key | @sara | 🟢 High |
| 16 | Chunk size benchmark — 256 vs 512 vs 1024 on eval set | @pixel | 🟡 Medium (scoring is human) |
| 17 | `ingestion/obsidian_preprocessor.py` — frontmatter → metadata, wikilink resolution | @sara | 🟢 High |
| 18 | `query/graph_retriever.py` — graph walk on top-K hits, pull linked notes | @sara + @pixel | 🟡 Medium |
| 19 | Obsidian MCP integration — bidirectional write-back from query sessions | @joe specs | 🟡 Medium |
| 20 | Phase 3 evaluation — re-run Phase 1 query set + 10 new cross-document queries | @pixel | 🔴 Low (human scoring) |

**Phase 3 gate**: Pixel eval ≥ 4/5 average on full 20-query set before Phase 4.

---

## Phase 4 — Hardening

**Goal**: Production-grade. Observable, recoverable, documented.

**Claude leverage**: 🔴 Low-Medium — load testing and DR validation require
a real environment. Alert thresholds need operational data. Claude generates
scaffolding; human judgment sets the parameters.

| Session | Deliverable | Owner | Notes |
|---|---|---|---|
| 21 | GCP Cloud Monitoring dashboards — query latency P50/P95, ingest lag, error rate | @joe + @sara | Claude generates config; thresholds need tuning |
| 22 | Load test — k6 script, 50 concurrent queries, latency under load | @sara | Claude writes k6 script; Andrey runs it |
| 23 | DR runbook — re-ingestion procedure if Cloud SQL data loss | @joe | Claude drafts; Andrey validates |
| 24 | FedRAMP SSP documentation if scope expands to cover this system | @joe + @codeguard | Org decision required first |
| 25 | Final security review — full pass against ARCHITECTURE.md security checklist | @codeguard | No shortcuts |

---

## Parallelism Summary

```
Phase 1:
  Sequential: Sessions 1 → 2 → 3 → 4
  Parallel:   Session 2 (VM deploy) ‖ Pixel drafts eval query set

Phase 2:
  Parallel:   Track A (backend swap) ‖ Track B (Terraform) ‖ Track C auth spec
  Sequential: Track C implementation (Sara) follows Track B deployment

Phase 3:
  Parallel:   Sessions 15-16 can overlap
              Sessions 17-18 (Obsidian) can overlap with Session 16 benchmark
  Sequential: Session 18 (graph retriever) requires Session 17 (preprocessor)
              Session 19 (MCP) requires Session 18 (graph retriever)

Phase 4:
  Mostly sequential — each depends on production system being stable first
```

---

## Where Claude Cannot Replace Human Work

| Area | Why human judgment is required |
|---|---|
| Phase 1 + 3 eval scoring | Relevance/accuracy is a human call — Claude cannot score its own outputs reliably |
| GCP infrastructure debugging | Real error messages from real provisioning needed for diagnosis |
| Alert threshold tuning | Requires baseline data from actual query load |
| Compliance boundary decisions | Whether this system enters a FedRAMP boundary is an org decision |
| DR validation | Destructive testing must be executed and observed by a human |
| Obsidian MCP write-back policy | What the system is allowed to create/modify is a governance decision |

---

## Session Targeting by Phase

| Phase | Sessions | Estimated calendar time |
|---|---|---|
| Phase 1 | 4 (1 complete) | ~1 week remaining |
| Phase 2 | 10 | ~2.5 weeks |
| Phase 3 | 6 | ~1.5 weeks |
| Phase 4 | 5 | ~1 week |
| **Total remaining** | **25** | **~6 weeks** |
