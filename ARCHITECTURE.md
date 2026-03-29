# Architecture — RAG Onboarding Tool

## Core Design Decisions

### Why RAG, not fine-tuning

Fine-tuning a model on organizational documents is expensive, requires curated training data, produces a model that can still hallucinate, and cannot be updated without retraining. RAG retrieves exact source chunks at query time — answers are grounded in documents, not baked into weights. Updates are as simple as re-ingesting a file.

### Why pgvector over a dedicated vector DB

For a first deployment in an enterprise environment, introducing a net-new managed service (Pinecone, Weaviate) adds operational overhead and a data egress concern. pgvector extends PostgreSQL — a database most infrastructure teams already operate, monitor, and back up. It handles millions of vectors comfortably and can be replaced later if scale demands it.

### Why LlamaIndex over LangChain

LangChain is flexible but optimized for agent workflows and tool chaining. LlamaIndex has cleaner abstractions specifically for document ingestion, indexing, and retrieval — which is the primary use case here. Less boilerplate for the core RAG loop.

---

## Ingestion Pipeline

```
Raw Documents
     │
     ▼
┌─────────────┐
│  Loader     │  Supports: PDF, Markdown, HTML, plain text,
│             │  Confluence XML export, JSON
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Chunker    │  Strategy: recursive character splitter
│             │  Chunk size: 512 tokens, overlap: 64 tokens
│             │  Metadata preserved: source file, section, timestamp
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Embedder   │  Model: text-embedding-3-small (Azure OpenAI)
│             │  Local dev: nomic-embed-text via Ollama
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Vector     │  pgvector on PostgreSQL
│  Store      │  Index type: HNSW (better query perf than IVFFlat)
└─────────────┘
```

**Chunking strategy rationale**: 512-token chunks with 64-token overlap balances retrieval precision with context coherence. Smaller chunks improve retrieval accuracy; overlap prevents important context from being lost at boundaries. This may be tuned per document type (runbooks benefit from larger chunks; policy docs from smaller).

---

## Query Pipeline

```
User Question
     │
     ▼
┌─────────────┐
│  Embedder   │  Same model as ingestion
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Retrieval  │  Top-K similarity search (default K=6)
│             │  Filtered by document category if specified
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Reranker   │  Optional: Cohere Rerank or cross-encoder
│  (optional) │  Improves precision for ambiguous queries
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  Prompt     │  System: "Answer using only the provided context.
│  Assembly   │   If insufficient, say so explicitly."
│             │  Context: retrieved chunks + source metadata
└─────┬───────┘
      │
      ▼
┌─────────────┐
│  LLM        │  GPT-4o (Azure OpenAI) — production
│             │  Llama 3 via Ollama — local dev
└─────┬───────┘
      │
      ▼
   Answer + Source Citations
```

---

## API Design

```
POST /query
{
  "question": "What services are in FedRAMP scope?",
  "filters": { "category": "compliance" },  // optional
  "k": 6                                     // optional, chunks to retrieve
}

Response:
{
  "answer": "...",
  "sources": [
    { "file": "fedramp-ssp.md", "section": "System Boundary", "chunk_id": "..." }
  ],
  "confidence": "high" | "medium" | "low"
}

POST /ingest
{
  "file_path": "...",
  "category": "runbook" | "compliance" | "architecture" | "incident" | "policy"
}

GET /health
GET /sources          // list all ingested documents
DELETE /sources/{id}  // remove a document from the index
```

---

## Infrastructure

### Local Development

```yaml
# docker-compose.yml (simplified)
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: onboarding_rag
  
  ollama:
    image: ollama/ollama
    volumes:
      - ollama_data:/root/.ollama
  
  api:
    build: ./api
    depends_on: [postgres, ollama]
    environment:
      USE_LOCAL_MODELS: "true"
```

### Azure Deployment (Production)

- **Azure Container Apps** (preferred over AKS for this workload — less ops overhead)
- **Azure Database for PostgreSQL Flexible Server** with pgvector extension
- **Azure OpenAI** for embeddings and completions
- **Azure Key Vault** for secrets
- **Azure AD** for OIDC auth on the API
- All provisioned via Terraform (azurerm provider)

---

## Security Considerations

- API requires Azure AD JWT — no anonymous access
- Document ingestion restricted to service principal with defined permissions
- All queries and retrieved chunks logged to Azure Monitor
- pgvector database not exposed publicly — accessed only from Container Apps via private endpoint
- Azure OpenAI endpoint scoped to single resource group

---

## Known Limitations & Future Work

- **No real-time ingestion** — document updates require manual re-ingest trigger (v2 will watch blob storage events)
- **No multi-modal support** — architecture diagrams in PDFs are not indexed (v2 will use vision model for diagram extraction)
- **English only** — embedding model performs best on English; multilingual support requires model swap
- **No conversation memory** — each query is stateless; follow-up questions require full context re-provision (v2 will add session history)
