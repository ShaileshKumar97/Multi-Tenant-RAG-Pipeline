# RAG Workflow

## Flow

1. User submits query with document context (optional)
2. Extract tenant_id from authenticated session
3. Generate embedding for user query
4. Vector search in embeddings table (filtered by tenant_id)
   - Retrieve top 20 similar chunks
   - Filter by is_active = TRUE and latest version
5. Re-rank top 20 to top 10 using cross-encoder
6. Construct prompt with retrieved context
7. Call LLM API with prompt
8. Post-process response:
   - Extract citations from response text
   - Calculate confidence scores
   - Link citations to source chunks
9. Store in audit tables (ai_responses, response_sources)
10. Return response with citations

## Retrieval Query

```sql
SELECT
    e.chunk_id,
    e.document_id,
    c.chunk_text,
    c.page_number,
    c.section_title,
    1 - (e.embedding <=> :query_embedding) as similarity
FROM embeddings e
JOIN chunks c ON e.chunk_id = c.id
WHERE e.tenant_id = :tenant_id
  AND e.is_active = TRUE
  AND e.version = (
      SELECT MAX(version)
      FROM embeddings e2
      WHERE e2.document_id = e.document_id
  )
ORDER BY e.embedding <=> :query_embedding
LIMIT 20;
```

## Retrieval Strategy

**Initial Retrieval:** Top 20 chunks by cosine similarity
- Fast approximate search using IVFFlat index
- Filters ensure tenant isolation and active versions only

**Re-ranking:** Cross-encoder model ranks top 20 to top 10
- More accurate than cosine similarity alone
- Considers query-chunk interaction
- Reduces noise before LLM processing

**Final Context:** Top 10 chunks sent to LLM
- Balance between context size and token cost
- Enough context for complex queries
- Not too large to cause confusion

## Prompt Construction

See examples/prompt_template.txt for full template.

Key elements:
- System message defining role and constraints
- Retrieved context with citations
- Current document being analyzed (if provided)
- User query
- Instructions for citations and confidence

## Hallucination Prevention

1. Explicit instruction: "Only use information from provided context"
2. Citation requirement: Every claim must cite source
3. Confidence scoring: Low confidence triggers disclaimers
4. Source verification: Citations validated against retrieved chunks

## Response Format

```json
{
  "response_id": "uuid",
  "answer": "Based on historical documents...",
  "citations": [
    {
      "document_id": "uuid",
      "document_name": "Doc-Q1",
      "page": 12,
      "section": "Safety Protocols",
      "excerpt": "The requirement states...",
      "similarity": 0.94
    }
  ],
  "confidence": 0.88,
  "sources_used": 3
}
```
