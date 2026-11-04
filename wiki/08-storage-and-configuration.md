# 8. Storage and Configuration

## SQLite storage

Implementation: `app/storage/sqlite_store.py`.

The store creates three tables if they do not exist:

### documents

| Column | Purpose |
|---|---|
| document_id | Primary key; content-derived identifier |
| filename | Uploaded filename |
| pages | Parsed page count |
| created_at | SQLite timestamp, defaults to current timestamp |

### blocks

| Column | Purpose |
|---|---|
| block_id | Primary key |
| document_id | Parent document identifier |
| page_number | One-based page |
| block_type | PDF text, OCR text or table |
| text | Extracted block content |
| bbox | JSON text for optional bounding box |
| confidence | Optional extraction confidence |

### chunks

| Column | Purpose |
|---|---|
| chunk_id | Primary key |
| document_id | Parent document |
| page_numbers | JSON-encoded page list |
| text | Chunk text |
| source_blocks | JSON-encoded source block IDs |
| source_type | Block type from which chunk originated |

Writes use SQLite parameterized statements and `INSERT OR REPLACE`. Reads reconstruct Pydantic `Chunk` models and return document rows ordered by creation time descending.

## Local artifacts

Default paths from `Settings`:
- `data/uploads`: API-uploaded files.
- `data/artifacts`: reserved/configured artifact directory; no current code writes generated outputs here.
- `data/clinical_rag.sqlite3`: SQLite database.
- `data/vector_index/dense.faiss`: FAISS index.
- `data/vector_index/chunks.pkl`: serialized chunk list.
- `data/eval/`: bundled evaluation inputs and results.

`get_settings()` creates data, upload, artifact and vector directories at first load.

## Configuration reference

| Setting | Default | Purpose |
|---|---|---|
| `APP_NAME` | Clinical Document Intelligence RAG | FastAPI title |
| `API_PREFIX` | `/api/v1` | Router prefix |
| `APP_ENV` | `demo` | Environment label |
| `DATA_DIR` | `data` | Data root |
| `UPLOAD_DIR` | `data/uploads` | API upload location |
| `ARTIFACTS_DIR` | `data/artifacts` | Artifact directory |
| `SQLITE_PATH` | `data/clinical_rag.sqlite3` | SQLite file |
| `VECTOR_DIR` | `data/vector_index` | Vector index location |
| `EMBEDDING_MODEL` | `sentence-transformers/all-MiniLM-L6-v2` | Sentence Transformer |
| `CHUNK_SIZE` | 900 | Chunk character length |
| `CHUNK_OVERLAP` | 150 | Chunk overlap |
| `RETRIEVAL_K` | 8 | Configured retrieval count; not used by current API route |
| `RERANK_K` | 5 | Streamlit Ask result count |
| `DENSE_WEIGHT` | 0.55 | Dense score weight |
| `BM25_WEIGHT` | 0.45 | Lexical score weight |
| `LLM_PROVIDER` | `mock` | Provider selector |
| `GROQ_API_KEY` | empty | Groq credential |
| `GROQ_MODEL` | `llama-3.1-8b-instant` | Groq model |
| `OLLAMA_URL` | `http://localhost:11434/api/generate` | Ollama endpoint |
| `OLLAMA_MODEL` | `llama3.1:8b` | Ollama model |
| `ENABLE_OCR` | true | OCR switch |
| `MIN_PDF_TEXT_CHARS` | 80 | OCR fallback threshold |
| `ALLOWED_ORIGINS` | localhost:8501 and localhost:8000 | CORS origins |

Settings are loaded with pydantic-settings from environment and `.env`; unknown fields are ignored. The configured Python literals map to environment variable names through Pydantic Settings behavior.

## Concurrency and durability notes

Each SQLite operation opens a new connection. No explicit connection pool or async DB layer is present. FAISS/chunk artifacts are local filesystem files. The SQLite database and index files are mounted to the host through the Compose data volume.
