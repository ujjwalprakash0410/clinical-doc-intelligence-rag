# 10. Deployment and Developer Workflow

## Local setup

Requirements are pinned in `requirements.txt`. The stack targets Python 3.11 in the Dockerfile; `pyproject.toml` also declares Ruff target version `py311`.

Typical setup:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

FastAPI docs:
`http://localhost:8000/docs`

Streamlit UI:
```bash
streamlit run streamlit_app.py
```

## LLM configuration

Default mode is `mock`, so no hosted API key is required.

Groq:
```bash
export LLM_PROVIDER=groq
export GROQ_API_KEY=...
```

Ollama:
```bash
export LLM_PROVIDER=ollama
export OLLAMA_MODEL=llama3.1:8b
```

Ollama must be running and reachable at the configured URL.

## Docker

The Dockerfile:
- Uses `python:3.11-slim`.
- Sets bytecode, stdout buffering and pip cache environment flags.
- Installs Tesseract OCR and build tools.
- Installs pinned requirements.
- Copies the project into `/app`.
- Exposes port 8000.
- Starts Uvicorn on `0.0.0.0:8000`.

Compose defines one service, `clinical-document-rag`, maps host port 8000 to container port 8000, mounts `./data` at `/app/data`, and sets `LLM_PROVIDER=mock`.

Build and run:
```bash
docker compose up --build
```

## Tests and lint

Test files:
- `tests/test_metrics.py`: hit/recall metric behavior.
- `tests/test_safety.py`: email/date redaction and diagnosis flag behavior.
- `tests/conftest.py`: adds repository root to Python import path.

Run:
```bash
pytest
ruff check .
```

Ruff config:
- line length 100,
- target Python 3.11,
- selects E, F, I, B, UP,
- ignores E501.

The current automated test suite is small and focuses on basic metric and safety helper behavior. It does not cover PDF parsing, API routes, storage integration, FAISS persistence, provider errors or end-to-end ingestion/query behavior.

## Operational caveats

- Docker Compose mounts `data/` for local persistence.
- The README quickstart uses `uvicorn main:app --reload`.
- The Docker image installs Tesseract but no additional external service.
- If using Groq, provide a valid key securely through environment configuration; do not commit secrets.
- The application currently has no authentication layer or deployment-grade secret management.
