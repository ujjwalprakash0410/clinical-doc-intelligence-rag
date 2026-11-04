# 4. Retrieval and Indexing

## Implementation

The hybrid search implementation is `app/retrieval/hybrid.py`, class `HybridRetriever`.

It combines:
- **Dense semantic retrieval**: Sentence Transformers embeddings stored in a FAISS inner-product index.
- **Lexical retrieval**: BM25 scores from `rank-bm25`.

Default configuration:
- Embedding model: `sentence-transformers/all-MiniLM-L6-v2`
- Dense weight: 0.55
- BM25 weight: 0.45
- API top-k default: 5 (request schema)
- Retrieval candidate pool: up to 4 × requested k from each retrieval signal.

## Index build

For every chunk:
1. Encode chunk text with Sentence Transformers.
2. Normalize embeddings.
3. Cast to float32.
4. Create `faiss.IndexFlatIP` with the embedding dimension.
5. Add all vectors.
6. Tokenize chunk text and construct `BM25Okapi`.
7. Persist FAISS index to `dense.faiss`.
8. Pickle the chunk list to `chunks.pkl`.

Because vectors are normalized, inner product acts as cosine similarity for the dense score.

## Tokenization

The runtime retriever's `_tokenize`:
- Replaces slash and hyphen with spaces.
- Splits on whitespace.
- Lowercases tokens.
- Drops empty tokens.

It is intentionally simple and does not apply a clinical tokenizer, stemming, stopword removal, synonym expansion or abbreviation normalization.

## Search and score fusion

Given a query:
1. Ensure the index/chunks are loaded.
2. Embed and normalize the query.
3. FAISS retrieves up to `min(max(k * 4, k), number_of_chunks)` dense candidates.
4. BM25 scores the query against all chunks.
5. Choose the union of dense candidates and the top `k * 4` lexical candidates.
6. Normalize BM25 scores by dividing each score by `max(max_bm25_score, 1.0)`.
7. Compute weighted score:

`combined = dense_weight * dense_score + bm25_weight * normalized_bm25_score`

8. Sort candidates descending and return the first k citations.

Each `EvidenceCitation` includes chunk ID, document ID, page list, first 350 characters of chunk text and rounded combined score.

## Persistence behavior

The FAISS index and chunk list are persisted under `vector_dir` (default `data/vector_index`). The BM25 index is not persisted independently; it is reconstructed from the chunk list after load.

The retriever's `load()` returns false when either persisted file is absent. The dependency factory calls `load()` once on construction; search also attempts load if in-memory index/chunks are missing.

## Why hybrid retrieval?

Dense search helps retrieve evidence phrased differently from the question. BM25 helps retain exact terminology, abbreviations, values and clinical keywords. Weighted fusion gives both signals a contribution to ranking.

## Current limitations and engineering considerations

- The FAISS index is an exact `IndexFlatIP`, not an approximate-nearest-neighbor index.
- BM25 normalization is a simple max-based scaling rather than a calibrated score normalization.
- The candidate union uses a set, so tie ordering before final score sort is not guaranteed.
- No cross-encoder reranker is implemented despite the `rerank_k` setting name.
- Rebuilding the index after each upload processes all chunks again.
- The retriever pickle is trusted local application state; untrusted pickle files must not be loaded.
- If there are zero chunks, `build([])` returns without clearing a previously built in-memory/persisted index. Treat empty-ingestion behavior carefully.
