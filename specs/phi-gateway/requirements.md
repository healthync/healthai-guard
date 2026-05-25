# Spec: MedMesh PHI Gateway Requirements

## Status

Draft

## Feature Name

MedMesh PHI Gateway

## Feature Profile

medmesh-external-model-strict

## Objective

Build the first MedMesh healthcare-specific runtime capability:

    PHI-safe clinical note processing for external model workflows.

The PHI Gateway must detect, transform, enforce, inspect, and audit protected health information before healthcare AI workflows interact with external or untrusted model boundaries.

---

## Problem Statement

Healthcare AI developers want to use LLMs and AI agents to process clinical notes and healthcare text.

However, clinical text may contain PHI.

If raw PHI is sent to an external model or returned in an unsafe output, the system may create privacy, compliance, trust, and patient safety risks.

MedMesh needs a runtime layer that can:

- inspect healthcare text
- detect PHI or PII
- transform sensitive content
- block unsafe model boundary crossings
- inspect generated outputs
- emit audit-safe governance metadata

---

## Product Thesis

The PHI Gateway proves the first core MedMesh value proposition:

    governed healthcare AI execution at runtime.

This is not a chatbot feature.

This is infrastructure for controlling how healthcare AI agents handle sensitive data.

---

## Scope

This spec covers the first implementation of:

1. PHIScrubberGuard
2. PHIPromptBoundaryGuard
3. PHILeakageOutputGuard
4. HealthcareAuditGuard
5. medmesh-external-model-strict profile
6. synthetic clinical note demo
7. tests for the above

---

## Non-Goals

This spec does not cover:

- production HIPAA compliance certification
- full HIPAA Safe Harbor certification
- Expert Determination workflows
- FHIR integration
- SMART on FHIR integration
- clinical decision support
- diagnosis or treatment recommendation
- model fine-tuning
- real patient data
- reversible pseudonymization
- DICOM or imaging de-identification
- voice, waveform, or multimodal PHI handling
- external LLM API integration

---

## Safety Positioning

The feature must not claim:

- HIPAA compliant
- clinically validated
- certified de-identification
- production-ready for patient data
- safe for diagnosis
- safe for autonomous clinical decisions

The feature may claim:

- PHI-aware
- HIPAA-aware
- designed to support healthcare AI governance workflows
- synthetic-data demo
- runtime PHI governance prototype
- not for clinical use

---

## Users

### Primary User

Healthcare AI developer building AI workflows that may process clinical text.

### Secondary Users

- healthcare platform engineer
- AI governance engineer
- clinical informatics researcher
- healthcare security engineer
- open-source contributor

---

## User Stories

### Story 1: Mask PHI Before Model Use

As a healthcare AI developer,

I want MedMesh to detect and mask PHI in clinical notes,

so that external models do not receive raw patient identifiers.

Acceptance:

- synthetic clinical note containing PHI is modified
- PHI is replaced with placeholders
- metadata indicates PHI was detected
- metadata does not include raw PHI

---

### Story 2: Allow Clean Clinical Text

As a healthcare AI developer,

I want clean clinical text to pass without unnecessary modification,

so that safe content is not over-processed.

Acceptance:

- clean clinical text returns PASS
- metadata indicates no PHI was detected
- no content modification occurs

---

### Story 3: Block Raw PHI At External Model Boundary

As a healthcare AI developer,

I want MedMesh to block prompts containing raw PHI before external model invocation,

so that unprotected PHI does not leave the trusted boundary.

Acceptance:

- prompt containing synthetic raw PHI is blocked
- metadata includes policy decision
- metadata includes reason code
- no raw PHI is stored in metadata

---

### Story 4: Detect PHI Leakage In Model Output

As a healthcare AI developer,

I want MedMesh to inspect model output before returning it,

so that PHI leakage can be blocked or handled.

Acceptance:

- output containing synthetic PHI is blocked
- clean output passes
- metadata indicates leakage decision
- audit is required for leakage event

---

### Story 5: Emit Safe Audit Events

As a healthcare governance engineer,

I want audit events for PHI governance decisions,

so that I can trace what the runtime did without storing raw PHI.

Acceptance:

- audit event is JSON serializable
- audit event includes session_id and task_id where available
- audit event includes guard decision metadata
- audit event excludes raw PHI and raw clinical note content

---

## Functional Requirements

### FR-001: PHI Detection

The system shall detect PHI or PII-like entities in text.

Initial detection backend:

    Presidio

Initial supported entity examples:

- PERSON
- DATE
- PHONE_NUMBER
- EMAIL_ADDRESS
- LOCATION
- ID-like values when detected by backend

The system shall expose metadata indicating whether PHI was detected.

---

### FR-002: PHI Masking

The system shall support a masking mode.

Example:

    John Smith was admitted on March 3, 2024.

Expected protected form:

    [PERSON] was admitted on [DATE].

The system shall return GuardVerdict.MODIFY when masking occurs.

---

### FR-003: No Raw PHI In Metadata

The system shall not store raw PHI values in GuardResult.metadata or audit events.

Forbidden metadata examples:

- patient_name
- original_text
- raw_note
- raw_phi
- medical_record_number_value
- phone_number_value
- email_value

Allowed metadata examples:

- phi_detected
- phi_types
- entity_count
- entity_summary
- handling_mode
- policy_id
- risk_level

---

### FR-004: External Model Boundary Enforcement

The system shall inspect prompts at the PRE_LLM intercept point.

If raw PHI is detected at this point under the medmesh-external-model-strict profile, the system shall return GuardVerdict.BLOCK.

The metadata shall include:

- phi_detected
- policy_id
- policy_decision
- reason_code
- audit_required
- risk_level

---

### FR-005: Output Leakage Detection

The system shall inspect model output at the OUTPUT intercept point.

If PHI is detected in output, the MVP behavior shall be to block the output.

The metadata shall include:

- phi_detected
- phi_types
- entity_count
- policy_id
- policy_decision
- reason_code
- audit_required
- risk_level

---

### FR-006: Audit Event Emission

The system shall emit audit events through HealthcareAuditGuard.

For MVP, acceptable audit sinks are:

- stdout
- local JSON lines file

The audit event shall include:

- audit_event_id
- audit_event_type
- timestamp
- session_id
- task_id
- guard_name
- verdict
- policy_id
- policy_decision
- phi_detected
- phi_types
- entity_count
- handling_mode
- risk_level

The audit event shall not include raw PHI.

---

### FR-007: Profile Factory

The system shall provide a factory for the medmesh-external-model-strict profile.

The profile shall produce a SecurityPipeline configured with the required MVP guards.

---

### FR-008: Synthetic Demo

The system shall include a runnable demo using synthetic clinical text only.

The demo shall not call a real external model.

The demo shall use a mock summarizer.

The demo shall print:

- original synthetic note
- protected note
- PRE_LLM boundary result
- mock summary
- OUTPUT inspection result
- governance metadata
- audit event

---

## Non-Functional Requirements

### NFR-001: Safety

The system shall fail closed when guard execution fails.

If a PHI guard fails unexpectedly, it should not silently pass unsafe content.

---

### NFR-002: Testability

The feature shall include unit tests for:

- PHIScrubberGuard
- PHIPromptBoundaryGuard
- PHILeakageOutputGuard
- HealthcareAuditGuard
- medmesh-external-model-strict profile

---

### NFR-003: Determinism

The MVP shall avoid external LLM calls.

The demo shall use synthetic text and deterministic mock behavior.

---

### NFR-004: Minimal Runtime Refactor

The MVP shall not rewrite core AgentGuard runtime contracts.

The implementation should extend the existing architecture.

---

### NFR-005: Metadata Consistency

All new guards shall use metadata keys from:

    src/agentguard/medmesh/metadata/keys.py

The implementation shall align with:

    docs/architecture/governance-metadata-contracts.md

---

## Required New Files

Expected implementation files:

    src/agentguard/medmesh/__init__.py
    src/agentguard/medmesh/guards/__init__.py
    src/agentguard/medmesh/guards/phi.py
    src/agentguard/medmesh/guards/boundary.py
    src/agentguard/medmesh/guards/audit.py
    src/agentguard/medmesh/profiles/__init__.py
    src/agentguard/medmesh/profiles/external_model_strict.py
    src/agentguard/medmesh/metadata/__init__.py
    src/agentguard/medmesh/metadata/keys.py
    examples/medmesh_phi_gateway_demo.py
    tests/medmesh/test_phi_guard.py
    tests/medmesh/test_boundary_guards.py
    tests/medmesh/test_audit_guard.py
    tests/medmesh/test_external_model_strict_profile.py

---

## Required Existing Runtime Dependencies

The implementation shall build on existing contracts:

    AgentContext
    GuardVerdict
    GuardResult
    GuardMode
    InterceptPoint
    SecurityGuard
    SecurityPipeline
    PipelineResult

---

## MVP Acceptance Criteria

The MVP is acceptable when:

1. PHIScrubberGuard masks synthetic PHI.
2. Clean input passes.
3. PRE_LLM boundary guard blocks raw PHI.
4. OUTPUT guard blocks PHI leakage.
5. Audit guard emits safe JSON events.
6. External strict profile builds a working SecurityPipeline.
7. Demo runs locally without external API keys.
8. Tests pass.
9. No audit or metadata output contains raw PHI.
10. README or demo docs clearly state the feature is a synthetic-data prototype and not for clinical use.

---

## Open Questions

1. Should PHIScrubberGuard subclass the existing PIIScrubberGuard or wrap Presidio directly?
2. Should OUTPUT leakage initially block or modify?
3. Should audit events be emitted by ASYNC guards only, or also directly by blocking guards?
4. Should PHIPromptBoundaryGuard use the same detector backend as PHIScrubberGuard?
5. Should Presidio be required for the profile, or should a regex fallback exist?
6. Should MedMesh use [PERSON] style placeholders or [PHI:PERSON] style placeholders?
7. Should the MVP support medical record number detection through custom recognizers immediately?

---

## Spec Decision Log

### Decision 1: Use Existing Runtime

Decision:

    The MVP will extend the existing AgentGuard runtime instead of rewriting it.

Reason:

    The current guard pipeline already supports PASS, BLOCK, MODIFY, intercept points, and fail-closed behavior.

---

### Decision 2: Use Synthetic Data Only

Decision:

    The MVP demo and tests will use synthetic healthcare text only.

Reason:

    Avoid privacy risk and keep the demo suitable for open-source publication.

---

### Decision 3: Start With Masking

Decision:

    The first PHI transformation mode will be masking.

Reason:

    Masking is simpler, deterministic, auditable, and safe for the first MVP.

---

### Decision 4: Do Not Implement FHIR Yet

Decision:

    FHIR governance is deferred until after the PHI Gateway MVP.

Reason:

    The first product proof should focus on the PHI/model boundary.

---

## Next Spec Files

Create the following:

    specs/phi-gateway/design.md
    specs/phi-gateway/tasks.md

