# SESSION_GUIDE.md — Working on This Project with AI Assistance

This document is the single most important file for maintaining momentum on this project. Read it before every session.

---

## The Core Problem

Long AI conversations degrade in three ways:
1. **Context drift** — the model loses track of earlier decisions and starts re-litigating them
2. **Token bloat** — re-explaining context wastes tokens and slows responses
3. **Scope creep** — long conversations invite tangents that pull you away from the current objective

This guide solves all three with one discipline: **treat every session as a focused contract with a defined deliverable.**

---

## Session Protocol

### Before Starting a New Session

1. Open `STATUS.md` and read the current state
2. Decide on **one specific deliverable** for this session (not a topic — a deliverable)
3. Copy the **Session Starter Block** below, fill it in, paste it as your first message

### Session Starter Block

```
## Session Context

**Project**: RAG Onboarding Tool (github.com/dolgova/rag-onboarding-tool)
**Goal**: Private AI knowledge system using RAG for DevOps role onboarding

**Current phase**: [copy from STATUS.md]
**Decisions locked in**: [copy the "Locked Decisions" section from STATUS.md]

**This session's deliverable**: [ONE specific thing — e.g., "working ingestion script that chunks a markdown file and stores embeddings in pgvector"]

**Do not**: [anything explicitly out of scope for this session]
```

Paste this at the start of every new conversation. It takes 60 seconds and saves 20 minutes of re-alignment.

---

## What Belongs in Each Session

Keep sessions **narrow and output-oriented**. Good session scopes:

| ✅ Good scope | ❌ Too broad |
|---|---|
| "Write the chunking module with tests" | "Build the ingestion pipeline" |
| "Terraform for pgvector on Azure" | "Set up the Azure infrastructure" |
| "Design the /query API contract" | "Work on the API" |
| "Benchmark chunk sizes 256 vs 512 vs 1024" | "Improve retrieval quality" |

If a session runs long and you haven't hit the deliverable, **stop and update STATUS.md** before closing. Never carry unresolved work across sessions silently.

---

## After Each Session

Update `STATUS.md` with:
- What was built / decided
- Any new locked decisions
- Blockers encountered
- The next session's recommended deliverable

This takes 5 minutes and is the entire system. Without it, the next session starts from zero.

---

## Token Efficiency Rules

1. **Don't re-paste code that already exists in the repo** — reference the file path instead
2. **Don't ask open-ended questions** — the more specific your prompt, the less the model has to explore
3. **Use the Session Starter Block** — it compresses full context into ~100 tokens
4. **One file or module per session** — avoid multi-file generation in one prompt; it produces worse output and costs more tokens
5. **If you get a long answer you didn't need** — add "Be concise. Output only code + brief inline comments" to your next prompt

---

## Project North Star

If you ever feel uncertain about whether a session's work belongs in this project, check against this:

> **This project builds a tool that ingests organizational documents and answers natural language questions about them, grounded only in those documents, deployed securely within a cloud perimeter.**

If the work doesn't serve that sentence, it's out of scope.

---

## Suggested Session Sequence

```
Phase 1 — Local prototype (Ollama + ChromaDB)
  Session 1: ingestion/chunker.py — load MD file, chunk, store in ChromaDB
  Session 2: query/retriever.py — embed question, similarity search, return chunks
  Session 3: query/prompt.py — assemble context + prompt, call Ollama, return answer
  Session 4: api/main.py — wrap sessions 1-3 in FastAPI /ingest and /query endpoints
  Session 5: docker-compose.yml — containerize the full local stack
  Session 6: end-to-end test — ingest 3 real docs, run 10 test queries, evaluate quality

Phase 2 — Azure deployment
  Session 7: swap ChromaDB → pgvector
  Session 8: swap Ollama → Azure OpenAI
  Session 9: Terraform for Azure Container Apps + PostgreSQL
  Session 10: Azure AD auth on the API
  Session 11: audit logging to Azure Monitor

Phase 3 — Quality & compliance
  Session 12: reranker integration
  Session 13: confidence scoring
  Session 14: access control per document category
  Session 15: source citation formatting in responses
```

---

## File Ownership

| File | Purpose | Updated when |
|---|---|---|
| `README.md` | Hiring manager-facing overview | Phase milestones only |
| `ARCHITECTURE.md` | Technical design | When a design decision changes |
| `STATUS.md` | Living progress tracker | After every session |
| `SESSION_GUIDE.md` | This file — working protocol | Only if the protocol changes |
