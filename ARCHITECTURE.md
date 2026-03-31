# Architecture — RAG Onboarding Tool

## Core Design Decisions

### Why RAG, not fine-tuning

Fine-tuning bakes knowledge into weights — expensive, requires curated training data,
can still hallucinate, and cannot be updated without retraining. RAG retrieves exact
source chunks at query time. Answers are grounded in documents. Updates are a re-ingest.

### Why ChromaDB (Phase 1) → pgvector (Phase 2)

ChromaDB runs as a Docker container with zero config — ideal for validating the pipeline
before committing to cloud infrastructure costs. Phase 2 swaps in Cloud SQL pgvector via
an environment variable toggle. Both backends implement the same interface — no code
changes to the query pipeline on swap.

### Why LlamaIndex over LangChain

LangChain is optimized for agent workflows and tool chaining. LlamaIndex has cleaner
abstractions for document ingestion, indexing, and retrieval — which is the primary
use case here. Less boilerplate for the core RAG loop.

### Why GCP over local / Azure

Limited local compute makes running Ollama + ChromaDB + API on a laptop impractical.
GCP Compute Engine (e2-standard-2) runs the identical Docker Compose stack with
adequate resources. Phase 2 uses GCP-native services: Vertex AI or OpenAI API for
completions, Cloud SQL pgvector for vector storage.

ADR-001 filed: 2026-03-31.

---

## Ingestion Pipeline

```
Raw Documents
(flat files OR Obsidian vault)
     │
     ▼
┌─────────────────┐
│  Loader         │  Supports: .md, .txt, .pdf
│                 │  Phase 3+: Obsidian vault with frontmatter + wikilink parsing
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Preprocessor   │  Phase 1: passthrough
│  (Phase 3+)     │  Phase 3: extract frontmatter → metadata
│                 │           resolve [[wikilinks]] → related note titles injected
│                 │           into chunk context before embedding
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Chunker        │  Strategy: SentenceSplitter (LlamaIndex)
│                 │  Chunk size: 512 tokens, overlap: 64 tokens
│                 │  Metadata: source, category, chunk_index, total_chunks
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Embedder       │  Phase 1: nomic-embed-text via Ollama
│                 │  Phase 2: text-embedding-3-small via OpenAI API
│                 │  Toggle: USE_LOCAL_MODELS=true/false
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Vector Store   │  Phase 1: ChromaDB (Docker, cosine similarity, HNSW)
│                 │  Phase 2: Cloud SQL pgvector
│                 │  Metadata per chunk stored alongside vectors
└─────────────────┘
```

**Chunking rationale**: 512/64 balances retrieval precision with context coherence.
Tuned per document type in Phase 3 — runbooks may benefit from larger chunks,
policy docs from smaller. Obsidian atomic notes map naturally to this chunk size.

---

## Query Pipeline

```
User Question
     │
     ▼
┌─────────────────┐
│  Embedder       │  Same model as ingestion (consistency is required)
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Retrieval      │  Top-K cosine similarity (default K=6, env configurable)
│                 │  Optional category_filter → pre-filters before similarity search
│                 │  Phase 3+: graph walk — retrieve linked Obsidian notes
│                 │  alongside similarity hits for richer context
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Reranker       │  Phase 3: Cohere Rerank or local cross-encoder
│  (Phase 3+)     │  Improves precision on ambiguous multi-document queries
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Confidence     │  Heuristic scoring on retrieval results:
│  Scoring        │  high   → top score ≥ 0.75 AND ≥ 3 chunks
│                 │  medium → top score ≥ 0.55 OR ≥ 2 chunks
│                 │  low    → anything else (flagged in response)
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Prompt         │  System: "Answer using ONLY the provided context.
│  Assembly       │   If insufficient, say so explicitly."
│                 │  Context: retrieved chunks + source metadata
│                 │  User input treated as data — not as instructions
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  LLM            │  Phase 1: Llama 3 via Ollama (local, no API cost)
│                 │  Phase 2: GPT-4o via OpenAI API
│                 │  Toggle: USE_LOCAL_MODELS=true/false
└──────┬──────────┘
       │
       ▼
   Answer + Source Citations + Confidence Level
```

---

## API Contract (implemented in api/main.py)

```
POST /ingest
Request:  { "file_name": str, "category": str }
          category ∈ { runbook, architecture, compliance, incident, policy, general }
          file_name must exist in the docs_ingest/ volume mount
Response: { "file": str, "category": str, "chunks_stored": int }

POST /query
Request:  { "question": str, "top_k": int (1-20, default 6),
            "category_filter": str | null }
Response: { "answer": str,
            "sources": [{ "source": str, "score": float, "category": str }],
            "confidence": "high" | "medium" | "low",
            "chunks_used": int }

GET  /health   → { "status": "ok", "backend": "local"|"cloud", "collection": str }
GET  /sources  → { "sources": [{ "source": str, "category": str, "chunks": int }] }
```

Error responses: `{ "detail": str }` with appropriate HTTP status code.
Input validation: empty questions rejected (400), top_k range enforced (400),
invalid categories rejected (400), missing files rejected (404).

---

## Infrastructure

### Phase 1 — GCP VM + Docker Compose

```
GCP Compute Engine e2-standard-2 (Ubuntu 22.04, 50GB disk)
└── Docker Compose
    ├── ollama          ollama/ollama:latest       port 11434
    ├── ollama-setup    curlimages/curl (one-shot)  pulls nomic-embed-text + llama3
    ├── chromadb        chromadb/chroma:latest     port 8000
    └── api             ./api/Dockerfile           port 8080
```

Document ingestion: drop files into `docs_ingest/` volume on the VM, POST to `/ingest`.

Provisioning:
```bash
make vm NAME=rag-dev ZONE=us-central1-a   # creates GCP VM
make ssh NAME=rag-dev                      # SSH in
bash infra/gcp/setup_vm.sh                # installs Docker, clones repo, pulls images
make up                                    # starts the stack
```

### Phase 2 — GCP-native services

| Component | Phase 1 | Phase 2 |
|---|---|---|
| Compute | GCP VM (Docker Compose) | Cloud Run or GKE |
| LLM | Llama 3 via Ollama | GPT-4o via OpenAI API |
| Embeddings | nomic-embed-text via Ollama | text-embedding-3-small via OpenAI API |
| Vector store | ChromaDB (Docker) | Cloud SQL for PostgreSQL + pgvector |
| Secrets | .env file | GCP Secret Manager |
| Auth | None | Google IAP or Cloud Endpoints JWT |
| IaC | gcloud CLI + setup_vm.sh | Terraform (google provider) |

Toggle between Phase 1 and Phase 2 backends: `USE_LOCAL_MODELS=true/false` in `.env`.
The application code does not change — only the backend configuration.

---

## Obsidian Vault Ingestion (Phase 3)

Standard RAG treats documents as flat, independent files. Obsidian vaults encode
relationships between documents via `[[wikilinks]]` and frontmatter metadata.
This improves retrieval accuracy in two ways:

**Graph-augmented retrieval**: After similarity search, walk the wikilink graph from
the top-K results to pull related notes. Retrieves contextually connected documents
that may not have scored highly on cosine similarity alone.

**Frontmatter as metadata**: Obsidian frontmatter (`category`, `owner`, `last-reviewed`)
maps directly to ChromaDB chunk metadata. No manual tagging. Staleness surfaced
automatically — answers from notes with old `last-reviewed` dates are flagged.

**Implementation**:
- Mount Obsidian vault directory in place of `docs_ingest/`
- Add `ingestion/obsidian_preprocessor.py` — parses frontmatter, resolves wikilinks,
  injects related note titles into chunk context before embedding
- Add `query/graph_retriever.py` — wraps base retriever with graph walk step
- Optionally wire Obsidian MCP server for bidirectional write-back (new notes
  created from query sessions, postmortems auto-indexed)

**Accuracy impact**: Higher retrieval relevance on ambiguous cross-document queries.
No impact on query latency — graph walk happens at ingestion time, not query time.

---

## Security Considerations

### Phase 1 (local GCP VM)

- VM access via GCP SSH (IAP tunnel) — no public SSH port
- API accessible on port 8080 — restrict via GCP firewall to known IPs only
- No credentials in code — all config via `.env`
- Audit log: GCP VM serial console + Docker logs captured by Cloud Logging

### Phase 2 additions

- API authentication via Google IAP or JWT — no anonymous access
- Document ingestion restricted to authorized service account
- All queries and retrieved chunks logged to Cloud Logging with full payload
- Cloud SQL not exposed publicly — accessed via Cloud SQL Auth Proxy only
- OpenAI API key stored in GCP Secret Manager — not in `.env`
- Path traversal protection on ingest endpoint (Phase 2 requirement — flagged by CodeGuard)
- Prompt injection mitigation — user question treated as data, not instructions

---

## Known Limitations & Planned Work

| Limitation | Phase targeted |
|---|---|
| No real-time ingestion — manual re-ingest required on document update | Phase 3: watch docs_ingest/ for changes |
| No multi-modal — diagram images in PDFs not indexed | Phase 3: vision model extraction |
| No conversation memory — each query is stateless | Phase 3: session history |
| Path traversal not explicitly validated on ingest endpoint | Phase 2 (CodeGuard flag) |
| Prompt injection mitigation is defensive-by-prompt only | Phase 2: input sanitization layer |
| Obsidian graph retrieval not yet implemented | Phase 3 |
| Chunking not tuned per document type | Phase 3: Pixel benchmark session |
