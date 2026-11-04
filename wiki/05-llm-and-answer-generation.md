# 5. LLM Providers and Answer Generation

## Provider interface

`app/llm/providers.py` defines:
- `LLMProvider.generate(question, citations, safety_flags)`
- `MockProvider`
- `GroqProvider`
- `OllamaProvider`
- `build_provider(settings)`

This interface keeps provider-specific code behind one method and returns a `ClinicalAnswer` model.

## Provider selection

The `LLM_PROVIDER` setting accepts `mock`, `groq` or `ollama`.

Selection behavior:
- If provider is `groq` and a Groq API key is configured, build `GroqProvider`.
- If provider is `ollama`, build `OllamaProvider`.
- Otherwise build `MockProvider`.

Therefore, a Groq configuration without a key silently falls back to MockProvider.

## MockProvider

This is the default offline/demo provider.

If no citations are found:
- Returns a message saying there is insufficient indexed evidence.
- Uses low confidence.
- Explains that no relevant chunks were retrieved.
- Includes the clinical disclaimer and supplied safety flags.

If citations exist:
- Creates up to three key-finding strings from document ID, pages and a 120-character snippet.
- Returns an extractive evidence-review response rather than a clinical conclusion.
- Uses moderate confidence when two or more citations are present, otherwise low.
- States that demo mode uses extractive evidence synthesis and clinician judgment is required.

## GroqProvider

- Uses the Groq Python client.
- Constructs context from retrieved citations, including document IDs, page numbers and quote text.
- Prompts the model to act as a clinician-facing evidence assistant, not diagnose or prescribe, and to use only supplied context.
- Uses configured model (default `llama-3.1-8b-instant`), temperature 0.1 and max tokens 700.
- Returns the generated text, citations, confidence, limitations, safety flags and disclaimer.

## OllamaProvider

- Sends question and citation quote text to the configured Ollama generate endpoint using HTTPX.
- Uses configured model (default `llama3.1:8b`).
- Sets `stream=False` and a 60-second request timeout.
- Reads the `response` field and returns it with citations and limitations.

## ClinicalAnswer contract

The answer object contains:
- `answer`
- `key_findings`
- `citations`
- `confidence`: low, moderate or high
- `limitations`
- `safety_flags`
- `disclaimer`

The current providers emit low or moderate confidence; the schema permits high but the code does not currently assign it.

## Important observed behavior

The API query route currently uses the same provider-generation call in both branches of `request.use_llm`. Thus setting `use_llm=false` does not disable LLM provider generation in the API path.

The Streamlit Ask tab calls the configured provider directly. It does not use the `QueryRequest` schema.

## Generation boundary

The provider sees only the question, retrieved citation snippets and safety flags. It does not directly query SQLite or FAISS. Evidence selection remains the retriever's responsibility.
