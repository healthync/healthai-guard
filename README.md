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

## Run MedMesh Tests

    PYTHONPATH=src pytest tests/medmesh -q

Expected current result:

    16 passed

---

## Project Structure

    src/agentguard/medmesh/
    ├── guards/
    │   ├── audit.py
    │   ├── boundary.py
    │   └── phi.py
    ├── metadata/
    │   └── keys.py
    └── profiles/
        └── external_model_strict.py

---

## Current MedMesh Components

### PHIScrubberGuard

Detects PHI/PII-like entities and masks them before model use.

### PHIPromptBoundaryGuard

Blocks raw PHI before an external model boundary.

### PHILeakageOutputGuard

Blocks PHI-like leakage in final model output.

### HealthcareAuditGuard

Emits audit-safe JSON events without storing raw PHI.

### External Model Strict Profile

Builds a governed runtime pipeline for external model workflows.

---

## Near-Term Roadmap

1. Improve healthcare-specific PHI recognition.
2. Add custom recognizers for medical record numbers.
3. Add facility/provider pattern detection.
4. Add richer audit event aggregation.
5. Add FHIR read-only governance.
6. Add SMART on FHIR scope mapping.
7. Add policy runtime.
8. Add LangGraph/MCP adapters.

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

