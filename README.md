# RAG-Powered Role Onboarding Tool

> A personal AI knowledge system that ingests organizational documents and serves as an intelligent onboarding and decision-support assistant — purpose-built for senior DevOps and platform engineering roles in compliance-driven environments.

---

## The Problem

When stepping into a senior DevOps or program management role at a security-focused company, you are immediately expected to make informed decisions across infrastructure, compliance posture, team topology, and tooling — often before you've had time to absorb hundreds of pages of runbooks, architecture decision records, incident postmortems, and policy documents.

Generic AI assistants hallucinate. Static wikis go stale. Human onboarding bandwidth is finite.

**This tool solves that by creating a private, auditable AI assistant grounded exclusively in your organization's actual documents.**

---

## What It Does

- Ingests internal documents (runbooks, ADRs, postmortems, compliance artifacts, architecture diagrams, Confluence/Notion exports)
- Chunks, embeds, and indexes them into a vector store
- Answers natural language questions — citing source documents
- Flags when it lacks sufficient context rather than guessing
- Runs fully within your cloud perimeter (no data leaves your environment)

**Example queries it can answer on day 30 that would otherwise take weeks to discover:**
- *"What's our current approach to secret rotation in production?"*
- *"Which services are in scope for FedRAMP and what are the boundary conditions?"*
- *"What did the March incident reveal about our alerting gaps?"*
- *"What IaC patterns does the team use for ECS task definitions?"*

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   INGESTION PIPELINE                 │
│                                                      │
│  Documents → Chunker → Embedding Model → pgvector   │
│  (PDF, MD, HTML, Confluence export, plain text)      │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                   QUERY PIPELINE                     │
│                                                      │
│  User Question → Embed → Similarity Search           │
│       → Rerank → Inject Context → LLM → Answer      │
└─────────────────────────────────────────────────────┘
```

See [ARCHITECTURE.md](./ARCHITECTURE.md) for full design detail.

---

## Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Embedding + LLM | Azure OpenAI | Native to target environment; FedRAMP eligible |
| Vector Store | pgvector (PostgreSQL) | Minimal ops overhead; reuses existing infra |
| Orchestration | LlamaIndex | Clean document-first RAG abstractions |
| API | FastAPI | Lightweight, async, easy to containerize |
| Infra | Docker Compose → AKS | Local dev parity with cloud deployment path |
| Auth | Azure AD OIDC | Consistent with enterprise zero-trust posture |

---

## Compliance & Security Posture

This tool is designed with compliance-driven environments in mind:

- **No external data egress** — all inference runs through Azure OpenAI within your tenant
- **Source attribution on every answer** — every response cites which document chunk it used
- **Access control at the document level** — ingest permissions mirror existing RBAC
- **Audit log** — all queries logged for review; important in FedRAMP/SOC 2 environments
- **Confidence thresholds** — configurable; low-confidence answers are flagged, not fabricated

---

## Project Status

| Phase | Status | Description |
|---|---|---|
| 0 — Design | ✅ Complete | Architecture decisions, stack selection |
| 1 — Local prototype | 🔄 In progress | Ollama + ChromaDB + LlamaIndex local stack |
| 2 — Core pipeline | ⬜ Planned | Ingestion, chunking, embedding, query API |
| 3 — Azure deployment | ⬜ Planned | Azure OpenAI, pgvector, AKS |
| 4 — Compliance layer | ⬜ Planned | Audit logging, access control, RBAC |
| 5 — UI | ⬜ Planned | Chat interface, source citations display |

See [STATUS.md](./STATUS.md) for current session progress.

---

## Why This Project

I built this to solve a real problem I've encountered repeatedly: senior engineers in compliance-heavy environments spend disproportionate time hunting for institutional knowledge that *exists* but isn't accessible. This tool applies RAG architecture — a proven pattern in enterprise AI — to the specific context of infrastructure and DevOps onboarding.

It also demonstrates practical competency across the full modern AI stack: document pipelines, vector search, LLM orchestration, and secure cloud deployment — applied to the kind of environment Keeper Security operates in.

---

## Repository Structure

```
.
├── README.md               ← You are here
├── ARCHITECTURE.md         ← Full technical design
├── STATUS.md               ← Living progress tracker
├── SESSION_GUIDE.md        ← How to work on this project efficiently with AI assistance
├── ingestion/              ← Document chunking and embedding pipeline
├── query/                  ← Retrieval and LLM query pipeline  
├── api/                    ← FastAPI service
├── infra/                  ← Terraform + Docker Compose
└── docs/                   ← Design notes, ADRs
```

---

*Author: Andrey Dolgov — Platform Engineering / DevOps*  
*Contact: [github.com/dolgova](https://github.com/dolgova)*
