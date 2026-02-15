# API Contracts

## POST /api/v1/documents/ingest

Ingest a new PDF document.

**Request:**
- Method: POST
- Content-Type: multipart/form-data
- Headers: Authorization (JWT token with tenant_id)

**Body:**
```
file: <PDF file>
document_name: string (optional, defaults to filename)
```

**Response:**
```json
{
  "document_id": "uuid",
  "status": "processing",
  "chunks_created": 45,
  "estimated_completion": "2026-02-15T10:30:00Z"
}
```

**Status Codes:**
- 200: Success, processing started
- 400: Invalid file format or missing file
- 401: Unauthorized
- 403: Tenant mismatch
- 500: Server error

**Validations:**
- File must be PDF
- File size max 10MB
- tenant_id from token must match authenticated user
- document_name must be unique per tenant (if provided)

---

## POST /api/v1/query

Submit a query for AI analysis.

**Request:**
- Method: POST
- Content-Type: application/json
- Headers: Authorization (JWT token with tenant_id)

**Body:**
```json
{
  "query": "What are the safety requirements for X?",
  "document_id": "uuid (optional)",
  "max_results": 5
}
```

**Response:**
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
  "sources_used": 3,
  "model": "gpt-4o",
  "timestamp": "2026-02-15T10:25:00Z"
}
```

**Status Codes:**
- 200: Success
- 400: Invalid query (too long, empty, etc.)
- 401: Unauthorized
- 403: Tenant mismatch or document not accessible
- 404: Document not found (if document_id provided)
- 500: Server error

**Validations:**
- query max 1000 characters
- query not empty
- document_id must belong to tenant_id (if provided)
- max_results between 1 and 10

---

## GET /api/v1/responses/{response_id}

Retrieve full audit information for a response.

**Request:**
- Method: GET
- Headers: Authorization (JWT token with tenant_id)

**Response:**
```json
{
  "response_id": "uuid",
  "query": "What are the safety requirements?",
  "answer": "Based on historical documents...",
  "full_prompt": "...",
  "retrieved_context": [
    {
      "chunk_id": "uuid",
      "chunk_text": "...",
      "page": 12,
      "similarity": 0.94
    }
  ],
  "citations": [...],
  "model_metadata": {
    "model": "gpt-4o",
    "temperature": 0.7,
    "max_tokens": 1000,
    "tokens_used": 1250
  },
  "confidence": 0.88,
  "processing_time_ms": 1250,
  "created_at": "2026-02-15T10:25:00Z"
}
```

**Status Codes:**
- 200: Success
- 401: Unauthorized
- 403: Response belongs to different tenant
- 404: Response not found
- 500: Server error

**Validations:**
- response_id must belong to tenant_id from token

---

## Error Response Format

All errors return:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": {}
  }
}
```

**Common Error Codes:**
- INVALID_REQUEST: Request validation failed
- UNAUTHORIZED: Missing or invalid token
- FORBIDDEN: Tenant mismatch or insufficient permissions
- NOT_FOUND: Resource not found
- INTERNAL_ERROR: Server error
