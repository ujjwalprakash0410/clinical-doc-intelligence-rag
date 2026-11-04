# 1. System Overview

## Purpose

Clinical Document Intelligence RAG is a local/free, production-inspired retrieval-augmented generation application for clinician-facing evidence review. It is designed to make answers traceable to source documents rather than behave as an unconstrained general chatbot.

The system handles:
- Born-digital and scanned PDFs.
- Image uploads (PNG/JPG/JPEG through the Streamlit UI; API parser accepts files and routes non-PDFs through image OCR).
- Text, OCR text and extracted tables as separate evidence blocks.
- Page-level source provenance.
- Hybrid semantic and lexical retrieval.
- Citation-backed response objects.
- Basic PHI pattern redaction and query risk flags.
- Retrieval evaluation and provider inference benchmarking.

## User-facing capability

A user uploads a clinical document, asks a question about its contents, and receives:
1. A generated or extractive evidence response.
2. Retrieved source snippets.
3. Document identifiers and page references.
4. Confidence and limitation metadata.
5. Safety flags and a clinical disclaimer.

The system is document-grounded: the retrieval layer supplies the evidence snippets that the generation provider receives. The implementation does not claim to validate the medical correctness of the retrieved source itself.

## Core technologies

| Layer | Implementation |
|---|---|
| HTTP API | FastAPI + Uvicorn |
| Request/response validation | Pydantic v2 |
| Demo UI | Streamlit |
| PDF text extraction | PyMuPDF (fitz) |
| PDF table extraction | pdfplumber |
| OCR | Tesseract via pytesseract and Pillow |
| Chunk metadata | Pydantic models |
| Persistent metadata | SQLite |
| Dense retrieval | Sentence Transformers + FAISS |
| Lexical retrieval | rank-bm25 |
| LLM integrations | Mock, Groq, Ollama |
| Evaluation | Custom Python metrics/scripts |
| Packaging | Docker |

## High-level architecture

```text
                 ┌────────────────────────┐
                 │     Streamlit UI       │
                 │ Ingest / Ask / Evidence│
                 │        / Metrics       │
                 └────────────┬───────────┘
                              │ uses shared app dependencies
                              ▼
┌───────────────┐       ┌────────────────────┐
│ PDF / Image   │──────▶│ Parser + OCR +     │
│ upload        │       │ table extraction   │
└───────────────┘       └─────────┬──────────┘
                                 ▼
                         ┌─────────────────┐
                         │ EvidenceBlocks  │
                         │ page + block ID │
                         └────────┬────────┘
                                  ▼
                         ┌─────────────────┐
                         │ Provenance      │
                         │ Chunker         │
                         └────────┬────────┘
                                  ▼
                   ┌────────────────────────────┐
                   │ SQLite: documents, blocks, │
                   │ chunks                     │
                   └────────────┬───────────────┘
                                ▼
                    ┌─────────────────────────┐
                    │ HybridRetriever         │
                    │ FAISS dense + BM25      │
                    └────────────┬────────────┘
                                 │ citations
                                 ▼
                    ┌─────────────────────────┐
                    │ Mock / Groq / Ollama    │
                    │ answer generation       │
                    └────────────┬────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │ Answer + citations +    │
                    │ limitations + flags     │
                    └─────────────────────────┘
```

## Design principles visible in code

### Provenance is modeled, not added as presentation text

The pipeline carries `document_id`, `page_number`, `block_id`, `chunk_id` and source-block references through ingestion and retrieval. This makes it possible to trace retrieved evidence to a source page and extracted block.

### Retrieval is hybrid

The retriever combines semantic similarity from normalized sentence embeddings with a BM25 lexical score. This is valuable in clinical text, where exact drug names, abbreviations, lab names and codes may matter alongside semantic similarity.

### Provider selection is isolated

The `LLMProvider` interface separates answer generation from API and UI orchestration. `build_provider` chooses the configured implementation.

### Demo safety is explicit but limited

The code redacts several identifier-like patterns and attaches query-classification flags. It is a starter safety layer, not comprehensive PHI detection, triage, clinical validation or an access-control system.

## Explicit non-goals

The README states that this is not a regulated medical device. The repository does not implement diagnosis, prescription, clinician identity/authentication, authorization policies, comprehensive audit logging or formal clinical validation.
