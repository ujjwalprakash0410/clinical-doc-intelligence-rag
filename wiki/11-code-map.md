# 11. Code Map

This page inventories all tracked files returned by the recursive repository tree.

```text
clinical-doc-intelligence-rag/
├── .gitignore
├── Dockerfile
├── README.md
├── main.py
├── pyproject.toml
├── requirements.txt
├── docker-compose.yml
├── streamlit_app.py
├── app/
│   ├── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── app.py
│   │   ├── deps.py
│   │   └── routes/
│   │       ├── __init__.py
│   │       ├── documents.py
│   │       ├── health.py
│   │       ├── metrics.py
│   │       └── query.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py
│   │   └── constants.py
│   ├── document_ai/
│   │   ├── __init__.py
│   │   ├── chunker.py
│   │   └── parser.py
│   ├── evaluation/
│   │   ├── __init__.py
│   │   └── metrics.py
│   ├── llm/
│   │   ├── __init__.py
│   │   └── providers.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── schemas.py
│   ├── retrieval/
│   │   ├── __init__.py
│   │   └── hybrid.py
│   ├── safety/
│   │   ├── __init__.py
│   │   ├── phi_redactor.py
│   │   └── query_classifier.py
│   └── storage/
│       ├── __init__.py
│       └── sqlite_store.py
├── data/
│   └── eval/
│       ├── golden_dataset.json
│       ├── who_pneumonia_eval.json
│       └── who_pneumonia_eval_results.json
├── scripts/
│   ├── benchmark_inference.py
│   ├── evaluate_who_guideline.py
│   └── run_eval.py
└── tests/
    ├── conftest.py
    ├── test_metrics.py
    ├── test_safety.py
```

## File responsibilities

| File | Responsibility |
|---|---|
| `main.py` | ASGI application entry point |
| `app/api/app.py` | FastAPI factory, middleware and router registration |
| `app/api/deps.py` | Cached construction of service dependencies |
| `app/api/routes/documents.py` | Document upload, ingestion and listing |
| `app/api/routes/health.py` | Health endpoint |
| `app/api/routes/query.py` | Redaction, safety flags, retrieval, generation and latency |
| `app/api/routes/metrics.py` | Document/chunk counts and index status |
| `app/core/config.py` | Environment-backed Settings and data-directory creation |
| `app/core/constants.py` | Clinical disclaimer and insufficient-evidence text |
| `app/document_ai/parser.py` | PDF text, OCR and table extraction |
| `app/document_ai/chunker.py` | Provenance-preserving overlapping chunks |
| `app/models/schemas.py` | Pydantic data contracts |
| `app/retrieval/hybrid.py` | Dense FAISS + BM25 search and citation creation |
| `app/llm/providers.py` | Mock, Groq and Ollama provider implementations |
| `app/safety/phi_redactor.py` | Regex-based PHI-like string replacement |
| `app/safety/query_classifier.py` | Fixed-keyword safety flags |
| `app/storage/sqlite_store.py` | SQLite persistence for documents, blocks and chunks |
| `app/evaluation/metrics.py` | Hit@K, Recall@K and mean metric helpers |
| `streamlit_app.py` | Ingest, Ask, Evidence and Metrics tabs |
| `scripts/run_eval.py` | Runtime hybrid retrieval evaluation |
| `scripts/evaluate_who_guideline.py` | Curated WHO lexical retrieval benchmark |
| `scripts/benchmark_inference.py` | One-call provider latency/cost-mode benchmark |
| `tests/*` | Basic safety and metric tests |
| `Dockerfile`, `docker-compose.yml` | Container build and local service definition |
| `requirements.txt`, `pyproject.toml` | Dependency pins and Ruff configuration |
| `.gitignore` | Git ignore rules |

All package `__init__.py` files in this repository are empty.
