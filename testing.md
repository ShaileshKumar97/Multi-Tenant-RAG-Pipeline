# Testing Strategy

## Retrieval Quality Testing

**Metrics:**
- Recall: Of all relevant chunks in corpus, how many did we retrieve in top 20? (Most critical - missing relevant info leads to incomplete answers)
- Precision@20: Of top 20 retrieved chunks, how many are relevant? (Measures initial retrieval quality)
- Precision@10: Of top 10 chunks sent to LLM, how many are relevant? (Measures final context quality after re-ranking)
- MRR (Mean Reciprocal Rank): How high are relevant results ranked?

**Test Cases:**
- Known relevant documents for specific queries
- Manual labeling of expected results (ground truth)
- Compare retrieval results against ground truth labels

**Validation:**
- Run on sample of 100 queries monthly
- Track metrics over time
- Alert if Recall drops below 0.8 (should find at least 80% of relevant chunks in top 20)
- Alert if Precision@20 drops below 0.6 (12 out of 20 initial chunks should be relevant)
- Alert if Precision@10 drops below 0.7 (7 out of 10 final chunks should be relevant after re-ranking)

## Tenant Isolation Testing

**Test Cases:**
1. Query from Tenant A should never return Tenant B data
2. Document ingestion for Tenant A should not be visible to Tenant B
3. Audit logs should only show tenant's own queries
4. Vector search should filter by tenant_id

**Implementation:**
- Automated tests with multiple tenant IDs
- Verify all queries include tenant_id filter
- Check row-level security policies
- Test edge cases (tenant deletion, etc.)

## Citation Accuracy

**Test Cases:**
- Citations should link to actual source chunks
- Citation page numbers should match chunk metadata
- All claims in response should have citations
- No citations to non-existent documents

**Validation:**
- Parse citations from response text
- Verify document_id and chunk_id exist
- Check page numbers match stored metadata
- Flag responses with missing citations

## Hallucination Detection

**Test Cases:**
- Responses should only use provided context
- No information not in retrieved chunks
- Explicit "I don't know" for missing information
- No made-up document names or pages

**Validation:**
- Compare response claims against retrieved context
- Flag responses with uncited claims
- Manual review of flagged responses
- Track hallucination rate over time

## Version Handling

**Test Cases:**
- Updates should reflect in new queries
- Old queries should still reference old versions
- Version numbers increment correctly
- Soft deletes prevent old data from appearing

**Implementation:**
- Test document update flow
- Verify version isolation
- Check that old embeddings are marked inactive
- Ensure queries only use active versions

## Regression Testing

**Approach:**
- Maintain test suite of 50 representative queries
- Run after any prompt or retrieval changes
- Compare responses before/after changes
- Flag significant differences for review

**Metrics:**
- Response similarity (semantic)
- Citation overlap
- Confidence score distribution
- Processing time

## Production Monitoring

**Metrics to Track:**
- Retrieval similarity scores (cosine similarity between query and chunks)
- Citation coverage (should be >80%)
- User feedback (thumbs up/down)
- Error rates by endpoint
- Query latency percentiles

**Alerts:**
- High hallucination reports from users
- Average retrieval similarity consistently below 0.6 (low relevance)
- Missing citations in responses
- Tenant isolation violations (critical)

## Test Data

**Requirements:**
- Sample PDFs from different domains
- Known query-answer pairs
- Ground truth for retrieval relevance
- Multiple tenant scenarios

**Maintenance:**
- Update test data quarterly
- Add new test cases for edge cases
- Remove obsolete test cases
