# Multi-Tenant RAG Service for Regulated Environments

## Overview

This system provides AI-assisted document analysis for regulated domains like legal and healthcare. It enables semantic search, context-aware suggestions, and maintains strict multi-tenant isolation while ensuring full traceability and explainability.

## Assumptions

1. **Data Scale**: 1000 customers, each with ~200 PDFs of ~5MB each
   - Total data: 1000 customers * 200 PDFs * 5MB = 1TB raw data
   - After chunking: Assuming average 50 chunks per PDF, ~10M chunks total
   - Vector storage: 10M chunks * 1536 dimensions * 4 bytes = ~61GB vector data

2. **Query Load**: 1000 daily queries across all tenants
   - Average: ~1 query per customer per day
   - Peak: Assume 10x multiplier = 10k queries/day peak capacity needed

3. **PDF Processing**: All PDFs are text-extractable (not scanned images)
   - OCR not required for initial implementation
   - Text extraction libraries (PyPDF2, pdfplumber) sufficient

4. **Authentication**: Tenant ID comes from authenticated session (JWT token)
   - API gateway validates tenant_id before routing
   - No client-provided tenant_id accepted

5. **LLM APIs**: Using hosted APIs (OpenAI GPT-4o, Anthropic Claude 4.5 Sonnet)
   - No custom model training or fine-tuning
   - API rate limits and costs are manageable

6. **Versioning**: Documents updated infrequently
   - Soft delete strategy sufficient (no need for event sourcing)
   - Version numbers increment on updates

7. **Compliance**: Audit logs retained for 7 years (regulatory requirement)
   - Storage cost: ~100KB per query * 1000 queries/day * 365 days * 7 years = ~255GB

## Architecture

```
Client to API Gateway to Service Layer to [PostgreSQL + pgvector + LLM API]
```

**Components:**
- API Gateway: Tenant validation, rate limiting
- Service Layer: RAG orchestration, prompt construction, response processing
- PostgreSQL: Metadata, document storage, audit logs
- pgvector: Embedding storage and similarity search
- LLM API: Hosted model for generation

## Design Decisions

### Multi-Tenant Isolation

**Approach:** Tenant ID in all tables + strict query filtering

- Every table includes `tenant_id` column
- All queries filtered by tenant_id (defense in depth)
- Row-level security policies in PostgreSQL
- Vector search always includes tenant_id filter

**Rationale:** Simpler than separate databases per tenant, sufficient isolation for this scale.

### Vector Database: pgvector

**Approach:** PostgreSQL extension for vector storage

- Single database for metadata and vectors
- Easier to enforce tenant isolation with SQL
- Good enough performance for 1k daily queries
- Simpler operations than separate vector DB

**Rationale:** Assignment specifies PostgreSQL + Vector DB. pgvector meets both requirements in one system.

### Embedding Granularity: Chunk-Level

**Approach:** 500-1000 token chunks with 50-100 token overlap

- Chunk at paragraph/section boundaries
- Store metadata: document_id, page_number, section_title
- Average 5MB PDF = ~50 chunks

**Rationale:** Document-level too coarse for 5MB PDFs. Field-level too complex without structured schema.

### Retrieval Strategy: Semantic Search + Re-ranking

**Approach:**
- Vector similarity search (top 20)
- Re-rank to top 10 using cross-encoder
- Filter by tenant_id at every step

**Rationale:** Pure semantic search can miss exact matches. Re-ranking improves precision without significant cost increase.

### Versioning: Soft Deletes

**Approach:** `is_active` flag + version number

- On update: mark old version inactive, create new version
- Queries filter by `is_active = TRUE` and latest version
- Old embeddings remain for audit trail

**Rationale:** Simpler than event sourcing. Sufficient for infrequent updates.

## Trade-offs

### Isolation vs Performance

**Chosen:** Tenant ID filtering in shared tables
- Trade-off: Slightly slower than separate databases
- Benefit: Simpler operations, easier maintenance
- Mitigation: Proper indexing on tenant_id

### Chunk Size

**Chosen:** 500-1000 tokens
- Trade-off: More chunks to search vs less context per chunk
- Benefit: Better precision for specific sections
- Mitigation: Overlap ensures context continuity

### Retrieval Count

**Chosen:** Top 20 initial, re-rank to top 10
- Trade-off: Higher LLM token cost vs better recall
- Benefit: More context for complex queries
- Mitigation: Re-ranking reduces noise before LLM

### Model Selection

**Chosen:** GPT-4o for complex analysis, GPT-4o-mini for simple queries
- Trade-off: Cost vs quality
- Benefit: Optimize cost while maintaining quality
- Implementation: Route based on query complexity

### Sync vs Async Ingestion

**Chosen:** Async for large PDFs, sync for small updates
- Trade-off: Complexity vs user experience
- Benefit: Non-blocking for large files
- Threshold: Files > 1MB processed asynchronously

## Tenant Isolation Implementation

**Data Layer:**
- All tables have `tenant_id` column
- Foreign key constraints ensure referential integrity
- Row-level security policies (optional but recommended)

**Vector Layer:**
- All vector queries include `WHERE tenant_id = :tenant_id`
- Index on (tenant_id, embedding) for efficient filtering

**API Layer:**
- Tenant ID extracted from JWT token
- Validated against database before processing
- Never trust client-provided tenant_id

**Defense in Depth:**
- API validation (first line)
- Database row-level security (second line)
- Application-level filtering (third line)

## Traceability

Every AI response links back to source documents:

1. `ai_responses` table stores query and response
2. `response_sources` table links response to chunks
3. `embeddings` table links chunks to documents
4. `documents` table stores original file metadata

**Chain:** Response -> Sources -> Chunks -> Documents

All links preserved with timestamps for audit.

## Explainability Features

- Citations with document name, page number, section
- Confidence scores (0.0-1.0) for each suggestion
- Source excerpts showing retrieved context
- Model metadata (which model, parameters used)
- Full prompt available in audit log

Users can see exactly why the AI made each suggestion.

## Repository Structure

```cmd
│   AI_USAGE.md
│   performance.md
│   README.md
│   testing.md
│  
├───api
│       api_contracts.md
│  
├───examples
│       audit_response.json
│       prompt_template.txt
│
├───schemas
│       er_diagrams.md
│       postgres_schemas.sql
│
└───workflows
        embedding_pipeline.md
        rag_workflows.md
```
