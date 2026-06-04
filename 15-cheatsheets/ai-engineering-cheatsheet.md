# AI Engineering Cheatsheet

> Quick-reference checklist for model selection, vector DBs, chunking configurations, semantic caching, and LLM APIs

---

## 1. RAG Ingestion Pipeline Checklist

- **Chunking Configurations**:
  - Semantic/Text-heavy: 512 tokens with 10% overlap (51 tokens).
  - Codebases: Chunk by functions or file bounds (preserve syntactic layout).
- **Metadata Tagging**:
  - Always inject parent ID, tags, source file directory, and access permissions (ACLs) directly to the JSON metadata schema.
- **Embedding Dimensions**:
  - `text-embedding-3-small` (OpenAI): 1536 dimensions.
  - `text-embedding-3-large` (OpenAI): 3072 dimensions.

---

## 2. Vector Databases & Metadata Filters

| Vector DB | Type | Index | Strengths |
|---|---|---|---|
| **Pgvector** | SQL Extension | HNSW / IVFFlat | Relational joins, strict row-level ACL filters |
| **Pinecone** | Cloud Native | Managed HNSW | Scalable, fully-managed, sparse-dense support |
| **Milvus** | Distributed | HNSW / Quantized | High-throughput, multi-GPU indexing |

---

## 3. Prompt Injection Guardrails

- **System Prompt Integrity**:
  - Demarcate user input with delimiters: `User query: """{{user_query}}"""`.
  - System instructions must occupy the highest-priority message role.
- **Content Moderation**:
  - Inspect inputs using lightweight moderation models before sending to primary model.
  - Inspect output token streams for security violations, block if leakage signatures appear.
