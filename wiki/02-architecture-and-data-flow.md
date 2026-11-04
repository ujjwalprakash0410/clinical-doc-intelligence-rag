# 2. Architecture and Data Flow

## Runtime entry points

There are two user entry points:

- **FastAPI**: `main.py` imports `create_app()` from `app/api/app.py`. The app registers health, documents, query and metrics routers under the configured API prefix (default `/api/v1`).
- **Streamlit**: `streamlit_app.py` imports the same dependency factories from `app.api.deps` and directly invokes the parser, chunker, store, retriever and provider.

Both paths reuse the same underlying service classes. Streamlit does not call the FastAPI endpoints; it calls the Python dependencies directly.

## Dependency construction

`app/api/deps.py` exposes cached factories using `functools.lru_cache`:
- `store()`: `SQLiteStore(settings.sqlite_path)`
- `parser()`: `ClinicalDocumentParser(settings)`
- `chunker()`: `ProvenanceChunker(chunk_size, chunk_overlap)`
- `retriever()`: `HybridRetriever(...)`, followed by `load()`
- `llm_provider()`: `build_provider(settings)`

The cache means each process generally reuses one instance for each dependency. Settings are separately cached by `get_settings()`.

## Ingestion flow

1. Receive a file.
2. Derive a safe basename from the submitted filename in the API route; Streamlit writes an uploaded file to a temporary file.
3. Parser hashes the file bytes to create a 16-character document identifier.
4. Parser extracts page text; OCR is attempted for low-text PDF pages when enabled; PDF tables are extracted with pdfplumber. Image input uses Tesseract OCR.
5. Parser returns evidence blocks with document/page/block identifiers and block type.
6. ProvenanceChunker normalizes whitespace and splits each evidence block into overlapping chunks.
7. Store document metadata, blocks and chunks in SQLite.
8. Retrieve all stored chunks and rebuild dense and BM25 indexes.
9. Report ingestion counts and index state to the caller.

The API ingestion route returns an `IngestResponse` with document ID, filename, page count, block count, chunk count, table count and `indexed=True`.

## Query flow

1. Validate `QueryRequest` (question length, top-k bounds).
2. Run PHI pattern redaction on the question.
3. Classify the redacted question for emergency/urgent symptom, diagnosis-request and medication-dosing terms.
4. Merge and deduplicate safety flags.
5. Search the hybrid retriever for the requested number of chunks.
6. Pass the redacted question, citations and flags to the configured LLM provider.
7. Measure elapsed time with `time.perf_counter()`.
8. Return `QueryResponse`: redacted question, structured answer, latency and number of retrieved chunks.

## Data flow: query

```text
Question
   │
   ▼
PHI redaction ───────▶ redacted question + PHI flags
   │
   ▼
Query classifier ────▶ urgency / diagnosis / dosing flags
   │
   ▼
HybridRetriever.search(question, top_k)
   │
   ▼
EvidenceCitation[] (chunk ID, doc ID, pages, quote, score)
   │
   ▼
Configured LLMProvider.generate(...)
   │
   ▼
ClinicalAnswer
   │
   ▼
QueryResponse + latency
```

## Index lifecycle

The retriever is loaded when its cached dependency is first constructed. On ingestion, the full set of SQLite chunks is re-indexed through `retriever.build(all_chunks)`. FAISS and the pickled chunk list are written to the configured vector directory. BM25 is constructed in memory and rebuilt from the chunk list after loading.

## Important distinction: API vs Streamlit

The API route uses `settings.rerank_k` only as a configured default? No: API `query` uses `request.top_k`. The Streamlit Ask tab uses `settings.rerank_k`. These are separate paths and their top-k defaults can therefore differ (API request schema defaults to 5; configuration `rerank_k` defaults to 5).
