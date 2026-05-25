# PHI-Safe Clinical Summarization Demo

## Objective

Demonstrate the first MedMesh MVP:

    PHI-safe clinical note summarization with external model protection.

The demo shows how MedMesh can inspect synthetic clinical text, mask PHI/PII-like content, enforce an external model boundary, inspect output, and emit an audit-safe event.

---

## Safety Notice

This demo uses synthetic data only.

It is not for clinical use.

It does not claim:

- HIPAA compliance
- clinical validation
- certified de-identification
- production readiness
- diagnostic safety

It is a research and engineering prototype for healthcare AI governance.

---

## Profile Used

The demo uses:

    medmesh-external-model-strict

This profile is intended for workflows where raw PHI must not be sent to an external or untrusted model.

---

## Demo Flow

The demo runs the following stages:

1. INPUT stage
2. PRE_LLM boundary stage
3. mock summarizer
4. OUTPUT stage
5. ASYNC audit tail

---

## Input Stage

The INPUT stage runs structural, pattern, and PHI guards.

Expected behavior:

- validate text input
- detect PHI/PII-like entities
- mask detected entities
- attach governance metadata

Example synthetic input:

    John Smith is a 67-year-old male admitted to Mercy General Hospital on March 3, 2024. His MRN is 123456.

Example protected output:

    [PERSON] is a [DATE_TIME] male admitted to Mercy General Hospital on [DATE_TIME]. His MRN is [DATE_TIME].

Current limitation:

    MRN 123456 may be detected as DATE_TIME by the baseline Presidio model.

This confirms the need for MedMesh-specific healthcare recognizers.

---

## PRE_LLM Boundary Stage

The PRE_LLM stage checks whether raw PHI remains before external model invocation.

Expected behavior:

- pass masked input
- block raw PHI
- attach boundary policy metadata

Policy ID:

    model.external.no_raw_phi.v1

---

## Mock Summarizer

The demo uses a deterministic mock summarizer.

It does not call a real external LLM.

Reason:

- avoid accidental PHI exposure
- avoid API keys
- keep the demo deterministic
- make tests and demos reproducible

---

## OUTPUT Stage

The OUTPUT stage inspects the model output before returning it.

Expected behavior:

- pass clean output
- block PHI-like leakage
- attach output policy metadata

Policy ID:

    output.no_phi_leakage.v1

---

## ASYNC Audit Tail

The async audit tail emits a JSON audit event.

The audit event includes:

- audit_event_id
- audit_event_type
- timestamp
- session_id
- task_id
- guard_name
- verdict
- policy_id
- policy_decision
- risk_level

The audit event must not include:

- raw clinical note
- patient name
- raw MRN
- raw prompt
- raw model output

---

## How To Run

Install dependencies:

    python -m pip install -e ".[presidio]"

Run demo:

    python examples/medmesh_phi_gateway_demo.py

Run tests:

    PYTHONPATH=src pytest tests/medmesh -q

---

## Expected Result

The demo should show:

- INPUT verdict: modify
- PRE_LLM verdict: pass
- OUTPUT verdict: pass
- audit event emitted as JSON

The tests should show:

    16 passed

---

## Current Known Limitations

The current implementation uses Presidio as the baseline backend.

Known gaps:

- MRN detection is not yet healthcare-specific.
- Facility names may not always be detected.
- Provider names may be partially detected.
- Safe Harbor category mapping is not complete.
- The audit guard emits a basic event only.
- There is no FHIR integration yet.
- There is no clinical safety validation yet.

---

## Next Improvement

The next implementation step should add custom healthcare recognizers for:

- medical record numbers
- MRN-like patterns
- facility or hospital names
- provider title/name patterns

---

## Developer-Provided Recognizers

MedMesh supports developer-provided Presidio recognizers.

This allows teams to add organization-specific healthcare identifiers without editing core MedMesh guard logic.

Examples of custom recognizers:

- internal patient IDs
- hospital-specific MRN formats
- payer member IDs
- lab accession numbers
- claim IDs
- custom encounter identifiers

Example pattern:

    from presidio_analyzer import Pattern, PatternRecognizer
    from agentguard.medmesh.guards import PHIScrubberGuard

    custom_recognizer = PatternRecognizer(
        supported_entity="CUSTOM_PATIENT_CODE",
        patterns=[
            Pattern(
                name="custom_patient_code",
                regex=r"\bPATIENT-CODE-[0-9]{4}\b",
                score=0.9,
            )
        ],
        supported_language="en",
    )

    guard = PHIScrubberGuard(
        custom_recognizers=[custom_recognizer],
    )

    guard.setup()

Developers can also disable MedMesh built-in healthcare recognizers:

    guard = PHIScrubberGuard(
        enable_healthcare_recognizers=False,
        custom_recognizers=[custom_recognizer],
    )

This is useful when a healthcare organization wants complete control over recognizer behavior.

---

## Current Built-In Healthcare Recognizers

The MVP includes lightweight built-in recognizers for:

- MEDICAL_RECORD_NUMBER
- PROVIDER_NAME
- HEALTHCARE_FACILITY

These recognizers are intentionally narrow and deterministic.

They are not a replacement for a full clinical de-identification engine.

