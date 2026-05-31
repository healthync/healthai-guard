# MedMesh

## The governance mesh for healthcare AI agents.

MedMesh is a healthcare AI governance and orchestration platform focused on:

- PHI-safe AI orchestration
- Policy-aware healthcare AI agents
- Zero-trust clinical AI systems
- FHIR-native interoperability
- Auditable healthcare AI infrastructure
- Runtime governance for healthcare AI workflows

## Vision

MedMesh provides the foundational runtime infrastructure layer for safe, governable, and interoperable healthcare AI systems.

The project is evolving from a general AI guardrail runtime into a healthcare-aware governance mesh for AI agents.

---

## Current MVP: PHI Gateway

The first MedMesh MVP is the PHI Gateway.

The PHI Gateway demonstrates:

- detection of PHI/PII-like entities in synthetic clinical text
- masking before external model boundaries
- PRE_LLM boundary enforcement
- OUTPUT leakage inspection
- governance metadata
- audit-safe JSON event emission

The first implemented profile is:

    medmesh-external-model-strict

This profile is designed for workflows where raw PHI must not cross into an external or untrusted model boundary.

---

## Second Capability: FHIR Read-Only Governance

The second MedMesh capability is FHIR Read-Only Governance.

This capability demonstrates:

- read-only FHIR operation enforcement
- FHIR resource-type governance
- simplified SMART/FHIR-like scope validation
- patient-context enforcement
- synthetic FHIR tool output inspection
- PHI masking on FHIR tool outputs
- audit-safe governance metadata

The implemented profile is:

    medmesh-fhir-readonly

This profile is designed for mock or synthetic FHIR workflows where AI agents need governed read-only access to healthcare resources.

The current implementation does not connect to a real FHIR server and does not validate real OAuth/SMART tokens.

---

## Safety Notice

This project is currently a research and engineering prototype.

It is:

- PHI-aware
- HIPAA-aware
- designed to support healthcare AI governance workflows
- intended for synthetic-data demos and development

It is not:

- HIPAA compliant by default
- clinically validated
- certified for de-identification
- production-ready for real patient data
- intended for diagnosis or treatment decisions

Do not use real patient data in the current demo.

---

## Run The PHI Gateway Demo

Install the package in editable mode:

    python -m pip install -e ".[presidio]"

Run the demo:

    python examples/medmesh_phi_gateway_demo.py

The demo uses synthetic clinical text only and does not call a real external LLM.

---

## Run The FHIR Read-Only Governance Demo

Run the demo:

    PYTHONPATH=src python examples/medmesh_fhir_readonly_demo.py

The demo uses synthetic FHIR-like JSON only.

It demonstrates:

- allowed read
- blocked write
- missing scope
- patient mismatch
- tool output PHI inspection on mock FHIR output

---

## Run MedMesh Tests

    PYTHONPATH=src pytest tests/medmesh -q

Expected current result:
    all MedMesh tests pass

---

## Project Structure

    src/agentguard/medmesh/
    ├── guards/
    │   ├── audit.py
    │   ├── boundary.py
    │   ├── fhir.py
    │   └── phi.py
    ├── metadata/
    │   └── keys.py
    ├── profiles/
    │   ├── external_model_strict.py
    │   └── fhir_readonly.py
    └── recognizers/
        └── healthcare.py

---

## Current MedMesh Components

### PHIScrubberGuard

Detects PHI/PII-like entities and masks them before model use.

The current MVP includes lightweight healthcare recognizers for:

- medical record numbers
- provider title/name patterns
- healthcare facility names

Developers can also provide their own Presidio recognizers without editing core MedMesh guard code.

### PHIPromptBoundaryGuard

Blocks raw PHI before an external model boundary.

### PHILeakageOutputGuard

Blocks PHI-like leakage in final model output.

### HealthcareAuditGuard

Emits audit-safe JSON events without storing raw PHI.

### External Model Strict Profile

Builds a governed runtime pipeline for external model workflows.

### FHIRReadOnlyGuard

Blocks non-read FHIR operations in the read-only profile.

### FHIRResourceScopeGuard

Restricts FHIR access to configured resource types.

### FHIRScopeGuard

Validates simplified SMART/FHIR-like scopes such as `patient/Observation.read`.

### PatientContextGuard

Requires patient context and blocks cross-patient access.

### FHIR Read-Only Profile

Builds a governed runtime pipeline for synthetic/mock FHIR read-only workflows.

---

## Near-Term Roadmap

1. Improve healthcare-specific PHI recognition.
2. Add richer audit event aggregation.
3. Add FHIR read-only governance.
4. Add SMART on FHIR scope mapping.
5. Add policy runtime.
6. Add LangGraph/MCP adapters.
7. Add evaluation datasets and PHI benchmark cases.
8. Add optional cloud or remote recognizer integrations.

---

## Research Direction

MedMesh supports a research direction around:

    Runtime governance for healthcare AI agents.

Initial research themes include:

- runtime PHI governance
- external model boundary enforcement
- policy-aware AI agent execution
- healthcare auditability
- FHIR-aware tool authorization
- zero-trust healthcare AI infrastructure

