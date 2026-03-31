# SESSION_GUIDE.md — Project Working Protocol

---

## North Star

> This project builds a tool that ingests organizational documents and answers
> natural language questions about them, grounded only in those documents,
> deployed securely within a GCP cloud perimeter.

If you're unsure whether a session's work belongs here, check against this sentence.

---

## Session Starter Block

Save this. Paste it at the top of every new conversation.
Update phase and locked decisions from STATUS.md before pasting.

```
## Session Context

**Project**: rag-onboarding-tool (github.com/dolgova/rag-onboarding-tool)
**Goal**: Private RAG system for DevOps role onboarding using organizational documents

**Current phase**: [copy from STATUS.md]
**Locked decisions**: [copy the Locked Decisions table from STATUS.md]

**This session's deliverable**: [ONE specific, shippable output]

**Out of scope this session**: [anything explicitly off-limits]
```

---

## Before Starting a Session

1. Open `STATUS.md` — read current phase and last session entry
2. Decide on one specific deliverable (not a topic — a deliverable)
3. Fill in the Session Starter Block
4. Paste it as your first message in the new conversation

---

## Session Scope Rules

| ✅ Good scope | ❌ Too broad |
|---|---|
| Deploy to GCP VM, validate `/health` returns 200 | "Set up the GCP environment" |
| `ingestion/obsidian_preprocessor.py` with frontmatter parsing | "Add Obsidian support" |
| `infra/terraform/modules/cloudsql/` — pgvector on Cloud SQL | "Set up the database" |
| Run 10-query eval, produce `docs/eval/phase1_eval.md` | "Evaluate the pipeline" |
| `query/reranker.py` — Cohere Rerank with fallback | "Improve retrieval" |

---

## After Each Session

Say: "Close this session and generate the STATUS.md update."
Paste the output into STATUS.md. Update the Session Starter Block.

---

## Environment Prerequisites

Before any session that touches a running stack:

- GCP VM running (`make ssh NAME=rag-dev`)
- Docker stack up (`make up` on the VM)
- `/health` returning 200 (`make health`)
- For Phase 2+ sessions: `gcloud auth login` and correct project set

For test-only sessions (no live stack needed):
- Python virtualenv active: `source .venv/bin/activate`
- `pytest tests/` — mocked backends, no services required

---

## Makefile Reference

```bash
make up                                    # start stack
make down                                  # stop stack
make health                                # check API
make logs                                  # tail all services
make ingest FILE=doc.md CATEGORY=runbook  # ingest a document
make query Q="your question here"          # run a query
make test                                  # run all tests
make vm NAME=rag-dev ZONE=us-central1-a   # create GCP VM
make ssh NAME=rag-dev                      # SSH to VM
make firewall                              # open port 8080
```

---

## Token Efficiency Rules

1. Reference existing files by path — never re-paste code that's in the repo
2. Use the Session Starter Block — it replaces full prose re-explanation
3. One file per generation — sequential output is better quality than multi-file dumps
4. Add `"Output only code + inline comments"` to prompts when explanation isn't needed
5. Locked decisions are locked — they're not re-opened in new conversations

---

## Remaining Session Sequence

```
Phase 1 — Local Prototype (GCP)
  ✅ Session 1: Full scaffold
  Session 2: GCP VM deploy + first live query [NEXT]
  Session 3: pytest gate on GCP VM
  Session 4: Phase 1 eval — 10 queries, Pixel gate ≥ 3.5/5

Phase 2 — GCP Production
  Track A: ingestion/store.py abstraction → OpenAI swap → pgvector swap
  Track B: Terraform cloudsql → cloudrun → secrets modules
  Track C: Path traversal fix → JWT auth → RBAC → Cloud Logging
  (See BUILDPLAN.md for full session breakdown — Sessions 5-14)

Phase 3 — Quality & Obsidian
  Session 15: query/reranker.py
  Session 16: Chunk size benchmark
  Session 17: ingestion/obsidian_preprocessor.py
  Session 18: query/graph_retriever.py
  Session 19: Obsidian MCP integration
  Session 20: Phase 3 eval — 20 queries, Pixel gate ≥ 4/5

Phase 4 — Hardening
  Sessions 21-25: Monitoring, load test, DR runbook, SSP docs, final security review
  (See BUILDPLAN.md)
```
