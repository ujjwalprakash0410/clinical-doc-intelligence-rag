# Clinical Document Intelligence RAG — Wiki

This wiki documents the implementation in the repository's `main` branch. It is organized around the runtime architecture, data flow, component responsibilities, API contract, evaluation and operational behavior.

## Start here

- [System Overview](01-system-overview.md) — purpose, boundaries and architecture.
- [Architecture and Data Flow](02-architecture-and-data-flow.md) — ingestion and question-answering paths.
- [Document Processing](03-document-processing.md) — PDF/image parsing, OCR, tables and provenance chunks.
- [Retrieval and Indexing](04-retrieval-and-indexing.md) — FAISS, BM25 and score fusion.
- [LLM Providers and Answer Contract](05-llm-and-answer-generation.md) — mock, Groq and Ollama behavior.
- [Safety and Clinical Guardrails](06-safety-and-clinical-guardrails.md) — PHI redaction, query flags and limitations.
- [API and Streamlit UI](07-api-and-user-interface.md) — endpoints and four UI tabs.
- [Storage and Configuration](08-storage-and-configuration.md) — SQLite schema, local artifacts and settings.
- [Evaluation and Benchmarking](09-evaluation-and-benchmarking.md) — metrics, WHO dataset and inference benchmark.
- [Deployment and Developer Workflow](10-deployment-and-development.md) — local run, Docker and tests.
- [Code Map](11-code-map.md) — file-by-file responsibility index.
- [Implementation Notes and Gaps](12-implementation-notes.md) — observed behavior, caveats and production gaps.

## Project at a glance

The application is a clinician-facing evidence-review demo. It accepts PDF/image documents, extracts text/OCR/table evidence with page identifiers, chunks and stores evidence, indexes chunks for hybrid retrieval, and produces citation-bearing responses through a configurable LLM provider.

It is explicitly **not** a diagnosis or prescribing tool and does not replace professional clinical judgment.

## Verified source scope

The documentation is based on the recursive `main` branch repository tree and the contents of every tracked file returned by GitHub at the time this wiki was prepared. Empty package `__init__.py` files are included in the code map but have no runtime logic. The repository does not currently contain a detailed external clinical validation or authentication subsystem.
