# 7. API and User Interface

## FastAPI application

`main.py` creates the app with `create_app()`. The factory:
- Loads settings.
- Creates FastAPI with configured app name and version `0.1.0`.
- Adds CORS middleware using configured allowed origins.
- Includes health, documents, query and metrics routers under `api_prefix` (default `/api/v1`).

## Endpoints

| Method | Path | Responsibility |
|---|---|---|
| GET | `/api/v1/health` | Basic health response |
| POST | `/api/v1/documents/ingest` | Upload, parse, chunk, persist and index a document |
| GET | `/api/v1/documents` | List stored document metadata |
| POST | `/api/v1/query` | Redact, classify, retrieve and generate answer |
| GET | `/api/v1/metrics` | Return document/chunk counts and index metadata |

### Health

Returns:
`{"status":"ok","service":"clinical-document-intelligence-rag"}`

It is a process-level endpoint; it does not check LLM connectivity, SQLite integrity or vector-index health.

### Ingest

Accepts a multipart upload named `file`. It:
- Sanitizes the submitted name to its basename.
- Writes file bytes to configured upload directory.
- Parses the file.
- Chunks evidence.
- Stores document, blocks and chunks in SQLite.
- Rebuilds the retriever from all stored chunks.

The response model is `IngestResponse`.

### Query

Request model:
- `question`: 3–2000 characters.
- `top_k`: default 5, range 1–20.
- `use_llm`: default true.

Response model:
- Redacted question.
- Structured `ClinicalAnswer`.
- `latency_ms`.
- `retrieved_chunks`.

The route currently calls `generate()` regardless of the value of `use_llm`; see [LLM Providers](05-llm-and-answer-generation.md).

### Metrics

Returns:
- document count,
- chunk count,
- retrieval label `hybrid_dense_bm25`,
- `index_ready` based on whether stored chunks exist.

This is a lightweight application metric endpoint, not a Prometheus exporter.

## Streamlit UI

The Streamlit app has four tabs:

### Ingest

Allows PDF, PNG, JPG and JPEG upload. On button click, writes the upload to a temporary file, parses and chunks it, stores data and builds the retriever. Displays counts for chunks, pages, blocks and tables.

### Ask

Provides a clinical question text area. On submit:
- redacts PHI patterns,
- adds classifier flags,
- searches using configured `rerank_k`,
- calls the configured provider,
- displays answer, disclaimer, flags and source citations.

### Evidence

Displays document metadata and total chunk count. Shows a dataframe containing up to the first 50 chunks.

### Metrics

Displays indexed chunk and document counts and directs the user to run `scripts/run_eval.py` after adding gold chunk IDs.

## API documentation

When running locally, FastAPI's generated OpenAPI docs are available at `http://localhost:8000/docs` (also described in README).
