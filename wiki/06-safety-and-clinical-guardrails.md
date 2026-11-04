# 6. Safety and Clinical Guardrails

## Safety components

- `app/safety/phi_redactor.py`: `redact_phi(text)`
- `app/safety/query_classifier.py`: `classify_query(query)`
- `app/core/constants.py`: disclaimer and insufficient-evidence constant
- `ClinicalAnswer`: exposes safety flags, limitations and disclaimer.

## PHI redaction

The redactor applies four regex patterns sequentially:
- Email-like strings → `[EMAIL]`
- Phone/ID-like digit sequences → `[PHONE_OR_ID]`
- Numeric dates → `[DATE]`
- MRN / Patient ID / ID labels followed by an identifier → `[PATIENT_ID]`

For each pattern matched, it records a flag such as `redacted_email`. Returned flags are deduplicated and sorted.

Both API query and Streamlit query paths redact the question before retrieval and provider generation.

### Scope limitation

This is pattern-based string substitution. It is not a comprehensive de-identification pipeline. It does not identify names, addresses, free-text clinical identifiers or all jurisdiction-specific protected health information. False positives and false negatives are possible.

## Query classification

The classifier lowercases the query and checks fixed term sets:
- Emergency/urgent: chest pain, stroke, suicidal, shortness of breath, severe bleeding.
- Diagnosis request: diagnose me, what disease do I have, do I have.
- Medication dosing: dose, dosage, how much should I take.

It returns labels:
- `emergency_or_urgent_symptoms`
- `diagnosis_request`
- `medication_dosing_request`

These are flags, not a semantic classifier. Matching depends on exact substring presence and may miss equivalent phrasing or match an incidental mention.

## How safety flags flow

```text
Question
  │
  ▼
redact_phi()
  ├── redacted question
  └── redaction flags
          │
          ▼
classify_query(redacted question)
          │
          ▼
merge + sort unique flags
          │
          ▼
Retriever + LLM provider
          │
          ▼
ClinicalAnswer.safety_flags + disclaimer
```

## What the code does not do

- It does not block or reject a query based on a safety flag.
- It does not trigger emergency triage or provide an emergency escalation workflow.
- It does not enforce a policy that prevents a provider from answering a dosing/diagnosis question.
- It does not validate extracted clinical content against current clinical guidelines.
- It does not implement authentication, role-based access, audit trails, encryption/key management or formal HIPAA controls.
- It does not guarantee that uploaded documents themselves are de-identified before storage.

## Clinical disclaimer

The application uses the following principle: clinician decision support only; it summarizes evidence and does not diagnose, prescribe or replace professional medical judgment. The README also states that the project is a demonstration and not for patient care.
