# 9. Evaluation and Benchmarking

The repository contains two distinct evaluation paths. They should not be conflated.

## A. Runtime retriever evaluation

Files:
- `scripts/run_eval.py`
- `app/evaluation/metrics.py`
- `data/eval/golden_dataset.json`

The runner loads `EvalCase` objects from JSON, queries the configured runtime `HybridRetriever`, and compares retrieved chunk IDs to the gold chunk IDs.

Metrics:
- Hit@K: 1 when at least one retrieved ID is relevant; otherwise 0.
- Recall@K: relevant retrieved IDs divided by number of gold IDs.
- Mean metric: arithmetic mean across cases, rounded to four decimals.
- Mean latency: measured per search and averaged.

The bundled `golden_dataset.json` currently contains one demo question with empty `gold_chunk_ids` and `gold_document_ids`. Since the gold ID list is empty, recall is computed as 0 by `recall_at_k`; this fixture is a template, not a meaningful retrieval benchmark until populated.

Run:
`python scripts/run_eval.py`

## B. Curated WHO guideline lexical evaluation

Files:
- `scripts/evaluate_who_guideline.py`
- `data/eval/who_pneumonia_eval.json`
- `data/eval/who_pneumonia_eval_results.json`

This is a separate, dependency-light retrieval experiment. It does not call the production `HybridRetriever`, FAISS, Sentence Transformers, API, or LLM. It:
1. Loads curated evidence snippets and questions.
2. Tokenizes terms using a regular expression.
3. Calculates an IDF map over the small chunk corpus.
4. Scores each chunk using query term IDF × term frequency divided by square root of document length.
5. Ranks the snippets.
6. Computes Hit@K, Recall@K, Precision@K, nDCG@K, MRR and retrieval latency.

The dataset represents one WHO childhood pneumonia guideline with 10 curated evidence chunks and 12 questions. Gold chunk IDs link questions to expected evidence snippets.

### Saved reported result snapshot

The committed results JSON records:
- Hit@1: 1.0000
- Hit@3: 1.0000
- Hit@5: 1.0000
- Recall@1: 0.8333
- Recall@3: 0.8750
- Recall@5: 0.8750
- Precision@1: 1.0000
- nDCG@3: 0.8966
- MRR: 1.0000
- Mean latency: 0.0856 ms/query
- Questions: 12
- Chunks: 10

These are small-corpus, curated, lexical retrieval results. They are not evidence of clinical validation, real-world patient safety, or performance of the runtime hybrid FAISS/BM25 pipeline. The README presents rounded latency around 0.1 ms/query.

Run:
`python scripts/evaluate_who_guideline.py`

The script overwrites `data/eval/who_pneumonia_eval_results.json`.

## C. Provider inference benchmark

File: `scripts/benchmark_inference.py`.

This script creates the configured provider, supplies one hard-coded demo citation and a fixed question, and measures one generation call. It prints:
- configured provider,
- latency in milliseconds,
- answer character count,
- estimated-cost mode.

The cost output is a mode marker: 0 for mock/Ollama and the string `provider_free_tier` for Groq. It is not measured token-level provider billing or a robust multi-run benchmark.

## Metric implementation

`hit_at_k`, `recall_at_k` and `mean_metric` live in `app/evaluation/metrics.py`. The WHO script implements its own precision/nDCG/MRR calculations.

## How to make evaluation stronger

- Add a representative corpus with source-grounded gold chunk IDs.
- Evaluate the actual runtime hybrid retriever on the same cases.
- Split data by document and clinical topic to avoid narrow benchmark overfitting.
- Include retrieval ablations (dense only, BM25 only, hybrid).
- Record multiple runs and percentile latency.
- Review citation correctness and answer faithfulness separately from retrieval relevance.
- Obtain appropriate clinical expert review; retrieval metrics alone do not establish clinical validity.
