# 12. Implementation Notes and Gaps

This page distinguishes current code behavior from intended or production-style positioning in the README.

## Confirmed implementation nuances

### 1. `use_llm` does not currently switch generation off

In `app/api/routes/query.py`, both sides of the conditional expression call:
`deps.llm_provider().generate(redacted_question, citations, safety_flags)`.

Therefore `use_llm=false` currently behaves the same as true. A future implementation should make the false branch return a deterministic/extractive response or remove the unused flag.

### 2. Safety labels do not enforce a block

PHI-like strings are redacted and query categories are added as flags. The route continues into retrieval and provider generation. There is no refusal, emergency escalation or hard gate in these helpers.

### 3. The WHO benchmark is not the runtime hybrid index

The curated WHO script implements its own token/IDF scoring over 10 hand-curated snippets. Its strong saved metrics do not measure the FAISS + Sentence Transformers + BM25 runtime class.

### 4. The generic runtime evaluation fixture is not yet populated

The current `golden_dataset.json` has one example whose gold chunk IDs are empty. Its output cannot establish meaningful retrieval quality until the gold evidence is populated.

### 5. The runtime evaluator expects a usable persisted index

`scripts/run_eval.py` calls the cached retriever and evaluates the indexed content. It does not ingest documents itself.

### 6. `rerank_k` is a configuration name, not a reranker

The Streamlit app uses this setting as the number of retrieved citations. No neural cross-encoder reranking stage exists in the repository.

### 7. Some declared settings are currently unused

`retrieval_k` is configured but not referenced in the current route/UI search calls. `artifacts_dir` is created but no current module writes evaluation artifacts there.

### 8. Metrics endpoint is simple inventory, not observability

The endpoint returns document and chunk counts, a retrieval label and a boolean index-ready value. There are no latency histograms, error rates, trace IDs or Prometheus exposition.

## Production-readiness gaps

The repository README itself calls out several limitations. The code review confirms the absence of:
- User authentication and authorization.
- Fine-grained document access controls.
- Durable audit event trail.
- Formal clinical safety review and validation.
- Comprehensive PHI detection/de-identification.
- Encryption and managed secrets.
- Robust monitoring and alerting.
- Production vector database or multi-worker-safe index lifecycle.
- File size/type validation and robust upload error handling.
- End-to-end integration tests.
- Model-output faithfulness checks or citation entailment verification.

## Suggested engineering priorities

1. Fix `use_llm` semantics and add a test.
2. Add ingestion/query end-to-end tests, including OCR fallback and index reload.
3. Validate upload type, size and parsing failures.
4. Make index rebuild atomic and safe under concurrent requests; consider incremental updates.
5. Separate safety detection from policy enforcement and define explicit behavior for urgent, diagnosis and dosing queries.
6. Expand PHI handling and avoid claiming complete de-identification.
7. Populate runtime gold cases and evaluate dense-only, BM25-only and hybrid retrieval on a realistic corpus.
8. Add source citation validation/faithfulness evaluation.
9. Add authentication, per-document authorization, audit logging and secure deployment controls before any real clinical deployment.

## What should not be inferred

- A high score on the small WHO lexical benchmark does not mean the system is clinically validated.
- The disclaimer does not make the software safe for patient care by itself.
- A citation pointing to a page does not prove the generated statement is fully supported by that page.
- The presence of a Groq/Ollama provider does not imply provider failover or retry logic; neither is implemented.
