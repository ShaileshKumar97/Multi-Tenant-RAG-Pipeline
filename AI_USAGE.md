# AI Tool Usage

## Tools Used (ChatGPT)
- For architecture validation, basic checks and for API contract review.
- For files structure validation from raw text

## Representative Prompts

1. "Review this multi-tenant RAG schema design. Any issues with tenant isolation?"

2. "Validate these REST API contracts for document ingestion and query endpoints."

3. "Check if this chunking strategy (500-1000 tokens) makes sense for 5MB PDFs."

## Changes After AI Review

**Schema Validation:**
- AI suggested issue with tenant isolation. Verified tenant_id is in all queries and added row-level security policies.

**API Contracts:**
- AI suggested adding rate limiting details. Added to error handling section.

**Chunking Strategy:**
- AI confirmed 500-1000 token chunks are reasonable for this use case. No changes needed but could be adjusted based on the testing.

## Rejected Output

**Separate Vector DB per Tenant:**
AI suggested using separate vector database instances per tenant. Rejected because:
- Too much operational overhead for 1000 tenants
- Current approach with tenant_id filtering is sufficient

**Complex Re-ranking Pipeline:**
AI suggested multi-stage re-ranking with multiple models. Rejected because:
- Not required for current scale
- Simple cross-encoder re-ranking is sufficient
