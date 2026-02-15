# Entity Relationship Diagram

## Relationships

```
tenants (1) to (many) documents
documents (1) to (many) chunks
chunks (1) to (1) embeddings
documents (1) to (many) ai_responses
ai_responses (1) to (many) response_sources
response_sources (many) to (1) chunks
response_sources (many) to (1) documents
```

## Entity Descriptions

**tenants**
- id (PK)
- name
- created_at

**documents**
- id (PK)
- tenant_id (FK references tenants)
- document_name
- file_path
- file_size
- page_count
- version
- is_active
- created_at
- updated_at

**chunks**
- id (PK)
- tenant_id (FK references tenants)
- document_id (FK references documents)
- chunk_index
- chunk_text
- page_number
- section_title
- token_count
- version
- created_at

**embeddings**
- id (PK)
- tenant_id (FK references tenants)
- chunk_id (FK references chunks)
- document_id (FK references documents)
- embedding (vector)
- version
- is_active
- created_at

**ai_responses**
- id (PK)
- tenant_id (FK references tenants)
- user_id
- query_text
- prompt_used
- model_used
- model_parameters
- response_text
- confidence_score
- processing_time_ms
- created_at

**response_sources**
- response_id (FK references ai_responses)
- chunk_id (FK references chunks)
- document_id (FK references documents)
- similarity_score

## Traceability Path

To trace an AI response back to source documents:

1. Start with `ai_responses.id`
2. Join `response_sources` on `response_id`
3. Join `chunks` on `chunk_id` to get chunk text
4. Join `documents` on `document_id` to get source file

All relationships preserve tenant_id for isolation.
