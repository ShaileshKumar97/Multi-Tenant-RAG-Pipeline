# Embedding Pipeline

## Process

1. Receive PDF file upload
2. Extract text from PDF
3. Chunk text into 500-1000 token segments
   - Split at paragraph boundaries
   - Maintain 50-100 token overlap between chunks
   - Preserve page numbers and section titles
4. Generate embeddings for each chunk
   - Model: text-embedding-3-small (1536 dimensions)
   - Batch processing for efficiency
5. Store in database:
   - Insert document record
   - Insert chunk records
   - Insert embedding records
   - All linked with tenant_id

## Chunking Strategy

**Size:** 500-1000 tokens per chunk
- Small enough for precise retrieval
- Large enough for context

**Overlap:** 50-100 tokens
- Ensures context continuity
- Prevents information loss at boundaries

**Boundaries:** Paragraph or section breaks
- Preserves semantic meaning
- Easier to cite specific sections

**Metadata:** Store with each chunk
- document_id
- chunk_index (order in document)
- page_number
- section_title (if available)
- token_count

## Versioning

When document is updated:

1. Mark old version inactive:
   ```sql
   UPDATE documents SET is_active = FALSE WHERE id = :doc_id;
   UPDATE embeddings SET is_active = FALSE WHERE document_id = :doc_id;
   ```

2. Create new version:
   - Increment version number
   - Process new document through pipeline
   - Store as new records (not updates)

3. Queries automatically use latest active version:
   - Filter by is_active = TRUE
   - Filter by MAX(version) per document

## Incremental Updates

Track document hash/checksum:
- Calculate hash after text extraction
- Compare with stored hash
- Only re-embed if hash changed
- Avoids unnecessary processing

## Batch Processing

For large PDFs (5MB = ~50 chunks):
- Process chunks in batches of 10
- Generate embeddings in parallel
- Insert to database in transaction
- Rollback on any failure

## Error Handling

- PDF extraction failure: Return error, don't store partial data
- Embedding generation failure: Retry with exponential backoff
- Database failure: Rollback transaction, return error
- Partial success: Not allowed, all-or-nothing
