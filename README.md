# Clinical Document Intelligence RAG

A free/local, resume-grade **clinical document intelligence** system for clinician-facing evidence review. It is intentionally designed as a production-inspired RAG project, not a diagnosis tool.

## Why this project exists

Healthcare RAG is only useful when it is auditable. This project focuses on the capabilities that make RAG helpful and safer in a high-risk clinical document setting:

- OCR and PDF parsing for scanned and born-digital clinical documents.
- Layout/table-aware evidence extraction with page provenance.
- Hybrid retrieval using dense embeddings plus BM25 lexical search.
- Citation-first answer generation with document/page references.
- PHI redaction and safety query flags.
- Evaluation scripts for retrieval quality and latency.
- Inference benchmarking for provider/model tradeoffs.

> Disclaimer: This project is for clinician decision support demonstrations only. It does not diagnose, prescribe, or replace professional medical judgment.

## Architecture

```text
PDF/Image Upload
   -> OCR/PDF/Table Parser
   -> Evidence Blocks with page provenance
   -> Provenance Chunker
   -> SQLite Metadata Store
   -> FAISS Dense Index + BM25 Keyword Index
   -> Hybrid Retrieval
   -> LLM Provider (Mock/Groq/Ollama)
   -> Citation-backed Clinical Evidence Answer
```

## Production-inspired RAG features

| Feature | Why it matters clinically |
|---|---|
| Page-level provenance | Clinicians can audit every answer back to source pages. |
| OCR fallback | Handles scanned charts, not just clean PDFs. |
| Table extraction | Captures labs, medications, vitals and structured record content. |
| Hybrid retrieval | Dense search improves semantic recall; BM25 preserves exact terms like drugs, labs, acronyms and codes. |
| Citation-first responses | Reduces unsupported claims and makes the answer reviewable. |
| PHI redaction | Prevents common identifiers from being sent to external LLMs in demo workflows. |
| Safety flags | Detects urgent, diagnosis-like, and medication dosing queries. |
| Evaluation harness | Measures whether retrieval changes improve quality. |
| Inference benchmark | Tracks latency/cost tradeoffs across mock, Groq and Ollama providers. |

## Free stack

- Backend: FastAPI, Pydantic
- UI: Streamlit
- Storage: SQLite + local filesystem
- Retrieval: FAISS + rank-bm25
- Embeddings: Sentence Transformers
- OCR/PDF: PyMuPDF, pdfplumber, pytesseract
- LLMs: Mock provider by default, Groq free tier optional, Ollama local optional
- Evaluation: custom Recall@K, Hit@K and latency metrics

## Quickstart

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

Open API docs at `http://localhost:8000/docs`.

For the demo UI:

```bash
streamlit run streamlit_app.py
```

## API examples

```bash
curl http://localhost:8000/api/v1/health
```

```bash
curl -X POST http://localhost:8000/api/v1/query \
  -H 'Content-Type: application/json' \
  -d '{"question":"What abnormal labs are present?","top_k":5}'
```

## Optional LLM providers

Default mode is `mock`, so the project runs without paid APIs.

```bash
export LLM_PROVIDER=groq
export GROQ_API_KEY=...
```

Or local Ollama:

```bash
export LLM_PROVIDER=ollama
export OLLAMA_MODEL=llama3.1:8b
```

## Evaluation

After ingesting documents and adding gold chunk IDs in `data/eval/golden_dataset.json`:

```bash
python scripts/run_eval.py
```

The evaluator reports Recall@K, Hit@K and retrieval latency.

## Inference benchmark

```bash
python scripts/benchmark_inference.py
```

Reports provider, latency, response size and estimated cost mode.

## Interview talking points

- Built a layout-aware clinical document RAG pipeline, not a generic chatbot.
- Preserved page/block provenance so answers are auditable.
- Used hybrid dense + lexical retrieval to improve clinical term matching.
- Added PHI redaction and safety flags for a regulated-domain mindset.
- Added evaluation and benchmarking so model/retriever changes are measurable.
- Kept the stack free/local while designing clear seams for production serving.

## Limitations

This is not a regulated medical device and should not be used for patient care. Production deployment would require authentication, audit logging, clinical validation, HIPAA controls, persistent vector infrastructure, access controls, monitoring and formal safety review.

## Reproducible WHO guideline evaluation

A lightweight, dependency-free retrieval evaluation is included for the bundled WHO childhood pneumonia guideline. It uses curated evidence snippets and 12 gold questions to evaluate whether the retriever can surface the correct supporting evidence.

```bash
python scripts/evaluate_who_guideline.py
```

Latest local results:

| Metric | Score |
|---|---:|
| Hit@1 | 1.0000 |
| Hit@3 | 1.0000 |
| Recall@1 | 0.8333 |
| Recall@3 | 0.8750 |
| Precision@1 | 1.0000 |
| nDCG@3 | 0.8966 |
| MRR | 1.0000 |
| Mean retrieval latency | ~0.1 ms/query |

These numbers are from a small curated benchmark, so they are best used as a resume/demo signal rather than as clinical validation.
