# System Design Interview: Retrieval-Augmented Generation (RAG) System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a Retrieval-Augmented Generation system that lets users ask natural language questions over a large corpus of documents and receive grounded, accurate answers. The system combines a vector search retrieval layer with an LLM generation layer, handling document ingestion, chunking, embedding, retrieval, re-ranking, and prompt construction. I will focus on the ingestion pipeline, the retrieval strategy (hybrid search with re-ranking), and the generation layer with citation tracking and hallucination mitigation."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | What is the corpus size? | 50M documents, ~500M chunks. |
| 2 | What document types (PDF, HTML, plain text)? | All three, plus structured data (tables). |
| 3 | How frequently are documents updated? | Mix: some static (archives), some updated daily. |
| 4 | Is this multi-tenant? | Yes, 1,000 tenants with isolated document collections. |
| 5 | What latency target for end-to-end query? | < 3 seconds for full answer with citations. |
| 6 | Do we need conversation history (multi-turn)? | Yes, follow-up questions should use context. |
| 7 | What LLM are we using? | 70B parameter model, hosted internally. |
| 8 | Accuracy requirements -- can we tolerate hallucination? | Minimal. Every claim must be traceable to a source document. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Corpus:**
- 50M documents, average 5 pages, 500 words/page = 125B words total.
- Chunking: 512 tokens per chunk with 128-token overlap = ~500M chunks.
- Embedding dimension: 1024 (e.g., BGE-large or E5-large).
- Vector storage: 500M x 1024 x 4 bytes (FP32) = 2 TB (or 1 TB with FP16).
- Metadata per chunk: ~200 bytes. 500M x 200 = 100 GB.

**Traffic:**
- 10K queries/minute = ~167 QPS.
- Each query: 1 embedding call + 1 vector search + 1 re-rank + 1 LLM call.
- Document ingestion: 100K new documents/day.

**Compute:**
- Embedding (query): ~5ms per query on GPU. 167 QPS = 1 GPU easily.
- Embedding (ingestion): 100K docs x 50 chunks = 5M chunks/day. At 1000 chunks/s per GPU = ~1.4 GPU-hours/day.
- Vector search: 500M vectors, top-100 ANN search ~10ms with HNSW index.
- Re-ranking: Cross-encoder on 100 candidates, ~200ms on GPU.
- LLM generation: ~1-2s for 300-token answer with 70B model.

**Storage:**
- Vectors: 1-2 TB (vector DB).
- Raw documents: 50M x 50 KB average = 2.5 TB (object storage).
- Chunk text: 500M x 300 bytes avg = 150 GB (document store).

## 3) Requirements

### Functional
- FR1: Ingest documents (PDF, HTML, text) with automatic chunking and embedding.
- FR2: Answer natural language queries with cited, grounded responses.
- FR3: Support multi-turn conversations with context.
- FR4: Multi-tenant document isolation.
- FR5: Support metadata filtering (date range, document type, tags).
- FR6: Handle incremental document updates and deletions.

### Non-Functional
- NFR1: End-to-end query latency < 3s p95.
- NFR2: Retrieval recall@20 > 90% (relevant docs in top 20 results).
- NFR3: < 5% hallucination rate (claims not supported by retrieved documents).
- NFR4: Ingestion throughput: 100K documents/day.
- NFR5: 99.9% availability.

## 4) API Design

```
# Document Ingestion
POST /v1/collections/{collection_id}/documents
Body:
{
  "documents": [
    {
      "id": "doc-123",
      "content": "...",           // or "url": "s3://..."
      "content_type": "pdf",
      "metadata": {"source": "annual_report", "year": 2025}
    }
  ]
}
Response: { "ingestion_job_id": "job-456", "status": "processing" }

# Query
POST /v1/collections/{collection_id}/query
Body:
{
  "question": "What were the Q3 revenue figures?",
  "filters": {"year": 2025},
  "top_k": 10,
  "conversation_id": "conv-789",     // optional, for multi-turn
  "include_sources": true
}
Response:
{
  "answer": "Q3 revenue was $4.2B, up 15% YoY...",
  "sources": [
    {"doc_id": "doc-123", "chunk_id": "c-45", "text": "...revenue reached $4.2B...", "score": 0.94},
    {"doc_id": "doc-456", "chunk_id": "c-12", "text": "...15% increase...", "score": 0.87}
  ],
  "conversation_id": "conv-789"
}

# Document Management
DELETE /v1/collections/{collection_id}/documents/{doc_id}
GET /v1/collections/{collection_id}/documents/{doc_id}/status
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| Collection | collection_id, tenant_id, embedding_model, chunk_config | PostgreSQL |
| Document | doc_id, collection_id, raw_uri, status, metadata, created_at | PostgreSQL |
| Chunk | chunk_id, doc_id, text, position, token_count | PostgreSQL or Elasticsearch |
| Embedding | chunk_id, vector (1024-dim), metadata | Vector DB (Milvus/Pinecone/Weaviate) |
| Conversation | conversation_id, tenant_id, messages[], created_at | Redis (TTL 24h) |
| IngestionJob | job_id, collection_id, doc_count, status, progress | PostgreSQL |

### Storage Choices
- **Raw documents**: S3/GCS.
- **Chunk text + metadata**: Elasticsearch (also powers keyword search for hybrid retrieval).
- **Vectors**: Dedicated vector DB (Milvus) with HNSW index, partitioned by collection.
- **Conversations**: Redis with TTL for session state.
- **Configuration**: PostgreSQL for collection and document metadata.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** Documents flow through a chunking and embedding pipeline into a vector database and text index; queries are embedded, searched via hybrid retrieval (vector + keyword), re-ranked by a cross-encoder, and the top results are assembled into a prompt for an LLM that generates a cited answer.

### Components

1. **Document Ingestion Pipeline** -- parse, chunk, embed, index.
2. **Embedding Service** -- encodes queries and chunks using a bi-encoder model.
3. **Vector Database** -- stores and searches embeddings (HNSW index).
4. **Text Index** -- Elasticsearch for BM25 keyword search.
5. **Hybrid Retriever** -- combines vector and keyword results.
6. **Re-Ranker** -- cross-encoder model to re-score top candidates.
7. **Prompt Builder** -- constructs LLM prompt with retrieved context and conversation history.
8. **LLM Service** -- generates the answer with inline citations.
9. **Citation Verifier** -- post-processing to verify claims against sources.
10. **Conversation Manager** -- tracks multi-turn context.

### Data Flow (Query Path)
1. User sends question.
2. Query rewriter reformulates for retrieval (e.g., expands multi-turn context).
3. Embedding Service encodes question to vector.
4. Hybrid Retriever: vector search (top 100) + BM25 search (top 100), merge with RRF.
5. Re-Ranker scores top 100 merged results, selects top 10.
6. Prompt Builder inserts top 10 chunks as context into prompt template.
7. LLM generates answer with [1], [2] citation markers.
8. Citation Verifier maps markers to source chunks, flags unsupported claims.
9. Response returned to user with answer and sources.

## 7) Deep Dive #1: Retrieval Pipeline -- Hybrid Search and Re-Ranking (8-12 min)

### Why Hybrid?
- **Vector search** (dense retrieval): Excellent for semantic similarity. Finds paraphrased content. Weak on exact keyword/entity matches.
- **Keyword search** (BM25): Excellent for exact terms, names, numbers. Weak on paraphrases.
- Combining both yields 10-20% higher recall than either alone.

### Hybrid Fusion Strategy
**Reciprocal Rank Fusion (RRF):**
```
RRF_score(doc) = sum(1 / (k + rank_in_list_i)) for each list i
```
where k = 60 (constant). Documents appearing in both lists get boosted.

Alternative: learned weighted combination (tune alpha between vector and BM25 scores on a validation set).

### Chunking Strategy
Chunking quality is the single biggest lever for RAG accuracy.

**Approach: Hierarchical chunking**
1. **Document parsing**: Extract text from PDF (layout-aware parser), HTML (structural parser), preserving headings and tables.
2. **Semantic chunking**: Split at paragraph/section boundaries, not fixed token windows.
3. **Chunk size**: Target 512 tokens (sweet spot for most embedding models). Overlap 128 tokens for context continuity.
4. **Parent-child chunks**: Store both small chunks (512 tokens, for precise retrieval) and their parent sections (2048 tokens, for LLM context). Retrieve by small chunk, return parent to LLM.
5. **Table handling**: Tables extracted as separate chunks with structured representation.

### Re-Ranking
After hybrid retrieval returns ~100 candidates, a cross-encoder re-ranks them:
- Cross-encoder (e.g., BGE-reranker, ms-marco-MiniLM): takes (query, chunk) pairs and scores relevance.
- Much more accurate than bi-encoder similarity but too slow for full corpus search.
- Latency: ~2ms per pair on GPU, 100 pairs = 200ms.
- This step typically improves precision@10 by 15-25%.

### Query Rewriting
For multi-turn conversations, the raw follow-up question may lack context:
- "What about Q4?" -- needs "Q4 revenue figures for Company X" from conversation history.
- Use a lightweight LLM call (or rule-based) to rewrite the query with context.
- Also generate query variants for retrieval: original + rewritten + hypothetical answer (HyDE).

### Metadata Filtering
- Pre-filter: Apply metadata filters (date, source, tags) before vector search. Reduces search space.
- Implemented via partitioned indexes in the vector DB (one partition per tenant or collection).
- Filter pushdown to the ANN index avoids post-filtering recall loss.

## 8) Deep Dive #2: Grounded Generation and Hallucination Mitigation (5-8 min)

### Prompt Engineering for Groundedness
```
System: You are a helpful assistant. Answer the user's question using ONLY the
provided context. Cite your sources using [1], [2], etc. If the context does not
contain enough information, say "I don't have enough information to answer this."

Context:
[1] {chunk_1_text} (Source: {doc_title}, Page {page_num})
[2] {chunk_2_text} (Source: {doc_title}, Page {page_num})
...
[10] {chunk_10_text}

Conversation History:
User: {previous_question}
Assistant: {previous_answer}

User: {current_question}
```

### Citation Verification
Post-generation step to validate citations:
1. Parse the answer for citation markers [1], [2], etc.
2. For each cited claim, extract the claim text and the cited chunk.
3. Use an NLI (Natural Language Inference) model to check if the chunk entails the claim.
4. If entailment score < threshold, flag as potentially unsupported.
5. Remove or rewrite unsupported claims before returning to user.

### Hallucination Prevention Strategies
1. **Constrained generation**: Instruct the model to only use provided context. Temperature = 0.1 for factual queries.
2. **Retrieval validation**: If no retrieved chunk has score > threshold (0.5), return "insufficient information" rather than guessing.
3. **Answer-context overlap**: Compute token overlap between answer and context. Low overlap = potential hallucination.
4. **Confidence scoring**: LLM self-assessment of answer confidence.
5. **Ensemble retrieval**: Run retrieval with multiple embedding models, only trust chunks found by both.

### Context Window Management
With 128K context models, we can fit many chunks. But more is not always better:
- Too many chunks dilutes relevant information ("lost in the middle" problem).
- Optimal: 5-10 highly relevant chunks, ordered by relevance score.
- For long documents: summarize less-relevant chunks, include full text of top-3.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Retrieval | Vector-only | Hybrid (vector + BM25) | Hybrid -- better recall |
| Chunking | Fixed-size (simple) | Semantic/hierarchical | Hierarchical -- better quality |
| Re-ranking | Skip (faster) | Cross-encoder (accurate) | Cross-encoder -- 200ms is acceptable |
| Embedding model | Small (384-dim, fast) | Large (1024-dim, accurate) | Large -- storage is cheap, accuracy matters |
| Generation | Stuff all chunks in prompt | Map-reduce over chunks | Stuff for < 10 chunks, map-reduce for > 10 |
| Citation | Trust LLM citations | NLI verification | NLI verification -- hallucination is costly |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (1,700 QPS, 5B chunks)
- Shard vector DB across multiple nodes (Milvus cluster).
- Cache frequent queries and their retrieved chunks (Redis, 10-min TTL).
- Batch embedding calls for ingestion throughput.
- Pre-compute embeddings for common query patterns.

### 100x (17K QPS, 50B chunks)
- Tiered retrieval: coarse ANN on compressed vectors (PQ), then fine-grained on full vectors for top candidates.
- Streaming ingestion pipeline with Kafka for real-time document updates.
- Model distillation: smaller, faster re-ranker trained on cross-encoder labels.
- Multi-region deployment with per-region vector DB replicas.
- Speculative retrieval: predict likely follow-up queries and pre-fetch results.
- LLM caching: semantic cache -- if a similar query was asked recently, reuse the answer.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Vector DB failure**: Replicated across nodes. Read from replica on primary failure. Rebuild index from stored embeddings.
- **LLM service failure**: Retry with exponential backoff. Fallback to smaller model. Return retrieved chunks without generated answer as degraded mode.
- **Embedding service failure**: Queue incoming queries. Return keyword-only search results as fallback.
- **Ingestion failure**: Kafka-backed pipeline with at-least-once delivery. Failed chunks retried. Dead letter queue for persistent failures.
- **Stale data**: Document TTL and re-indexing schedule. Version stamps on chunks to detect staleness.
- **Data isolation**: Per-tenant collection partitioning. Query-time tenant filter enforced at vector DB level.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Retrieval recall@K**: Measured on a labeled evaluation set periodically.
- **Re-ranker precision@10**: How often top-10 are truly relevant.
- **Hallucination rate**: NLI verification failure rate.
- **End-to-end latency breakdown**: embedding (5ms) + retrieval (10ms) + re-rank (200ms) + LLM (1.5s).
- **Ingestion lag**: Time from document upload to searchability.
- **Cache hit rate**: Query and chunk cache effectiveness.

### Dashboards
- Query quality: retrieval scores, citation coverage, answer length distribution.
- Performance: latency breakdown by stage, QPS by tenant.
- Ingestion: documents processed, chunks created, errors.

### Alerts
- Retrieval recall drops below 80% on evaluation set.
- Hallucination rate > 10%.
- Ingestion lag > 1 hour.
- Vector DB query latency > 50ms p99.

## 13) Security (1-3 min)

- **Document access control**: Per-collection and per-document ACLs. User's query only searches documents they have access to.
- **PII handling**: PII detection during ingestion. Option to redact or encrypt PII in chunks.
- **Prompt injection**: Input sanitization to prevent users from overriding system prompts via their queries.
- **Data residency**: Tenant-specific storage regions. Vectors and chunks co-located.
- **Audit logging**: All queries logged with tenant_id, retrieved doc_ids, and generated answer.
- **Encryption**: At rest (AES-256) and in transit (TLS 1.3).

## 14) Team and Operational Considerations (1-2 min)

- **Retrieval Team** (3-4 engineers): Embedding models, vector DB, hybrid search, re-ranking.
- **Ingestion Team** (2-3 engineers): Document parsing, chunking, pipeline orchestration.
- **Generation Team** (2-3 engineers): Prompt engineering, citation verification, LLM integration.
- **Platform Team** (2-3 engineers): API, multi-tenancy, scaling, infrastructure.
- **Evaluation Team** (1-2 engineers): Quality benchmarks, hallucination detection, A/B testing.

## 15) Common Follow-up Questions

1. **How do you evaluate RAG quality?**
   - Golden dataset of question-answer-source triples. Measure retrieval recall, answer correctness (LLM-as-judge), citation accuracy, and hallucination rate.

2. **How do you handle documents that change frequently?**
   - Incremental re-indexing: detect changed sections, re-chunk and re-embed only those. Version stamps on chunks; old versions garbage-collected.

3. **How do you handle very long documents (100+ pages)?**
   - Hierarchical chunking with section summaries. Two-stage retrieval: first find relevant sections, then search within them.

4. **How do you handle multi-modal content (images, charts)?**
   - Vision model extracts descriptions from images/charts during ingestion. These descriptions become searchable text chunks.

5. **How do you compare RAG vs. fine-tuning?**
   - RAG is better for frequently changing knowledge, auditability (citations), and multi-tenant isolation. Fine-tuning is better for style/format adaptation and when retrieval latency is unacceptable.

## 16) Closing Summary (30-60s)

"We designed a RAG system with three main innovations: a hybrid retrieval pipeline combining vector search and BM25 with reciprocal rank fusion and cross-encoder re-ranking for high recall; a hierarchical chunking strategy with parent-child relationships for precise retrieval with rich LLM context; and a grounded generation layer with NLI-based citation verification to minimize hallucinations. The system processes 50M documents across 1,000 tenants, answers queries in under 3 seconds with cited sources, and scales to billions of chunks through sharded vector indexes and tiered retrieval."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| chunk_size | 512 tokens | Larger = more context per chunk, smaller = more precise retrieval |
| chunk_overlap | 128 tokens | Prevents losing context at boundaries |
| top_k_retrieval | 100 | More candidates for re-ranker = better recall |
| top_k_rerank | 10 | Chunks passed to LLM; more = richer context but slower |
| rrf_k | 60 | RRF constant; standard value |
| vector_weight | 0.7 | Weight of vector vs. keyword in hybrid; tune on eval set |
| temperature | 0.1 | Low for factual answers |
| min_retrieval_score | 0.5 | Below this, return "insufficient information" |
| nli_threshold | 0.7 | Citation verification strictness |
| conversation_ttl | 24h | How long to keep multi-turn context |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| RAG | Retrieval-Augmented Generation -- ground LLM answers in retrieved documents |
| Bi-encoder | Encodes query and document separately; fast but less accurate |
| Cross-encoder | Encodes query-document pair together; accurate but slow |
| BM25 | Classic keyword retrieval algorithm |
| HNSW | Hierarchical Navigable Small World -- ANN index algorithm |
| RRF | Reciprocal Rank Fusion -- merging ranked lists |
| HyDE | Hypothetical Document Embedding -- generate a fake answer, embed it for retrieval |
| NLI | Natural Language Inference -- check if premise entails hypothesis |
| Chunking | Splitting documents into smaller pieces for retrieval |
| Hallucination | LLM generating claims not supported by provided context |
| Grounding | Ensuring LLM output is traceable to source documents |

## Appendix C: References

- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020)
- Gao et al., "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE, 2022)
- BGE Embedding and Re-ranker models (BAAI)
- LlamaIndex and LangChain RAG frameworks
- Milvus vector database documentation
- Microsoft "Lost in the Middle" paper (Liu et al., 2023)
