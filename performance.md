# Performance & Scalability

## Current Scale

- 1000 customers
- ~200 PDFs per customer = 200k total PDFs
- ~1TB raw data
- 1000 daily queries (1 per customer average)
- Peak: 10k queries/day capacity needed

## Caching Strategy

**Query Result Cache:**
- Cache frequent queries for 1 hour
- Key: hash(tenant_id + query + document_id)
- Invalidate on document updates
- Expected hit rate: 20-30% for common queries

**Embedding Cache:**
- Cache document embeddings (long TTL, 7 days)
- Only invalidate on document updates
- Reduces embedding generation cost

**Metadata Cache:**
- Cache document metadata in Redis
- TTL: 24 hours
- Reduces database load

## Async Processing

**Document Ingestion:**
- Large files (>1MB) processed asynchronously
- Job queue (Celery/RQ) for background processing
- User receives immediate acknowledgment
- Status polling endpoint for completion

**Embedding Generation:**
- Batch processing for multiple chunks
- Parallel API calls to embedding service
- Rate limiting to avoid API throttling

**Audit Logging:**
- Fire-and-forget async writes
- No blocking on user request
- Eventual consistency acceptable

## Cost Control

**LLM Usage:**
- Route simple queries to GPT-4o-mini (cheaper)
- Use GPT-4o only for complex analysis
- Limit retrieved chunks to top 10 (reduces context tokens)
- Cache common queries

**Embedding Costs:**
- Cache embeddings (avoid re-generation)
- Batch API calls
- Only re-embed on document updates

**Storage Costs:**
- Vector data: ~61GB (one-time)
- Audit logs: ~255GB over 7 years
- Total: ~316GB storage needed

## Scaling Considerations

**Read Replicas:**
- PostgreSQL read replicas for query load
- pgvector queries can use replicas
- Write to primary, read from replicas

**Horizontal Scaling:**
- API servers: Stateless, easy to scale
- Load balancer distributes requests
- No shared state between servers

**Database Partitioning:**
- Partition embeddings table by tenant_id (if needed)
- Only if single database becomes bottleneck
- Current scale doesn't require partitioning

**Vector Search Optimization:**
- IVFFlat index with correct lists parameter
- Tune based on query patterns
- Use HNSW index if frequent data changes expected

## Performance Targets

- Query latency: <3 seconds (p95) - includes vector search and LLM call
- Document ingestion: <5 minutes for 5MB PDF
- Vector search: <500ms for top 20 results
- LLM response: <2 seconds (depends on API)

## Monitoring

- Track query latency (p50, p95, p99)
- Monitor LLM token usage per tenant
- Alert on high error rates
- Track cache hit rates
- Monitor database connection pool
