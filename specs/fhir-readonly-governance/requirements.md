# Spec: MedMesh FHIR Read-Only Governance Requirements

## Status

Draft

## Feature Name

MedMesh FHIR Read-Only Governance

## Objective

Build the second MedMesh healthcare-specific runtime capability:

    governed read-only access to FHIR resources for healthcare AI agents.

This capability should ensure that AI agents can only read FHIR data when:

- the tool is allowed
- the requested FHIR operation is read-only
- the requested FHIR resource type is permitted
- patient context is present when required
- SMART/FHIR-like scopes are satisfied
- tool output can be inspected for PHI before entering model context
- audit-safe governance metadata is emitted

---

## Relationship To PHI Gateway

The PHI Gateway MVP proved:

    INPUT -> PHI masking -> PRE_LLM boundary -> OUTPUT inspection -> audit event

FHIR Read-Only Governance extends this into:

    TOOL_AUTH -> PRE_TOOL -> FHIR read -> TOOL_OUTPUT PHI inspection -> ASYNC audit

The two features should work together.

FHIR read output may contain PHI, so it should pass through MedMesh PHI inspection before model use.

---

## Problem Statement

Healthcare AI agents may need to retrieve clinical data from FHIR servers.

Examples:

- patient demographics
- observations
- medication lists
- conditions
- encounters
- allergies
- diagnostic reports

However, unrestricted FHIR access creates serious risks:

- unauthorized patient lookup
- bulk patient data extraction
- cross-patient data leakage
- resource-level overreach
- read access beyond user scope
- PHI exposure to external models
- weak auditability
- agent tool abuse

MedMesh needs a runtime governance layer that controls FHIR read access before and after tool execution.

---

## Product Thesis

The FHIR Read-Only Governance feature proves that MedMesh can govern healthcare AI tool access, not just prompts.

This expands MedMesh from PHI boundary protection into:

    policy-aware healthcare AI tool governance.

---

## Scope

This spec covers:

1. FHIR read-only tool authorization
2. FHIR operation validation
3. FHIR resource type governance
4. patient context validation
5. SMART/FHIR-like scope validation
6. FHIR tool output PHI inspection
7. audit metadata for FHIR access
8. synthetic/mock FHIR demo
9. tests

---

## Non-Goals

This spec does not cover:

- real EHR integration
- production SMART on FHIR launch
- OAuth token validation
- real HAPI FHIR server integration
- FHIR write operations
- FHIR create/update/delete/patch
- clinical decision support
- diagnosis or treatment recommendations
- full consent management
- full FHIR R6 Permission implementation
- bulk export
- real patient data

---

## Safety Positioning

The feature must not claim:

- certified FHIR compliance
- production EHR readiness
- HIPAA compliance by default
- clinical validation
- real patient data safety

The feature may claim:

- FHIR-aware
- read-only governance prototype
- SMART-scope-inspired authorization
- synthetic/mock FHIR demo
- designed to support healthcare AI governance workflows

---

## Users

### Primary User

Healthcare AI developer building an agent that needs controlled access to FHIR resources.

### Secondary Users

- clinical informatics engineer
- healthcare platform engineer
- security engineer
- AI governance engineer
- researcher
- open-source contributor

---

## User Stories

### Story 1: Authorize Read-Only FHIR Tool Access

As a healthcare AI developer,

I want MedMesh to check whether an agent is allowed to call a FHIR read tool,

so that unauthorized tools cannot access clinical data.

Acceptance:

- allowed FHIR read tool passes
- disallowed FHIR tool blocks
- metadata records tool_name and policy_decision

---

### Story 2: Block Non-Read FHIR Operations

As a healthcare governance engineer,

I want MedMesh to block write-like FHIR operations in the read-only profile,

so that agents cannot create, update, patch, or delete clinical records.

Acceptance:

- read operation passes
- search operation may pass if configured as read-only
- create/update/delete/patch operations block
- metadata includes fhir_operation and reason_code

---

### Story 3: Enforce Resource Type Permissions

As a healthcare platform engineer,

I want MedMesh to check whether the requested FHIR resource type is permitted,

so that agents only access approved resources.

Acceptance:

- permitted resource type passes
- unpermitted resource type blocks
- metadata includes fhir_resource_type

---

### Story 4: Enforce Patient Context

As a healthcare AI developer,

I want MedMesh to require patient context for patient-level FHIR reads,

so that agents cannot perform broad or cross-patient access.

Acceptance:

- patient-scoped read with patient context passes
- patient-scoped read without patient context blocks
- mismatched patient context blocks
- metadata includes patient_context_required, patient_context_present, and patient_scope_match

---

### Story 5: Validate SMART/FHIR-Like Scopes

As a healthcare governance engineer,

I want MedMesh to validate required scopes such as patient/Observation.read,

so that FHIR access follows least-privilege principles.

Acceptance:

- required scope present passes
- missing scope blocks
- metadata includes fhir_scope_required and fhir_scope_present

---

### Story 6: Inspect FHIR Tool Output For PHI

As a healthcare AI developer,

I want MedMesh to inspect FHIR tool output before it enters model context,

so that PHI can be masked or blocked according to the active profile.

Acceptance:

- FHIR output containing PHI is detected
- PHI metadata is attached
- output can be masked before model use
- no raw PHI appears in metadata

---

### Story 7: Emit FHIR Audit Metadata

As a healthcare governance engineer,

I want FHIR access decisions to be audit-safe,

so that I can trace which resource was accessed, by which workflow, and under what policy.

Acceptance:

- metadata includes tool_name, fhir_resource_type, fhir_operation, policy_decision
- metadata includes session_id/task_id when available
- metadata excludes raw FHIR payload and raw PHI

---

## Functional Requirements

### FR-001: FHIR Read Tool Authorization

The system shall authorize FHIR read tool calls before execution.

The guard shall evaluate:

- tool_name
- allowed_tools
- fhir_resource_type
- fhir_operation
- patient context
- required scope

---

### FR-002: Read-Only Operation Enforcement

The system shall allow only read-only operations in this profile.

Allowed operations:

- read
- search

Blocked operations:

- create
- update
- patch
- delete
- transaction
- batch write
- bulk export

---

### FR-003: Resource Type Governance

The system shall restrict FHIR resource types to configured allowed resource types.

Initial allowed resource types may include:

- Patient
- Observation
- Condition
- MedicationRequest
- Encounter
- AllergyIntolerance
- DiagnosticReport

---

### FR-004: Patient Context Requirement

The system shall require patient context for patient-scoped access.

The system shall block when:

- patient context is missing
- requested patient_id does not match authorized patient context
- a search request lacks patient constraint where required

---

### FR-005: Scope Validation

The system shall support SMART/FHIR-like scope validation.

Example required scopes:

- patient/Patient.read
- patient/Observation.read
- patient/Condition.read
- patient/MedicationRequest.read
- patient/Encounter.read

The MVP may use a simplified internal scope list rather than validating a real OAuth token.

---

### FR-006: FHIR Tool Argument Validation

The system shall validate expected FHIR tool arguments.

Expected fields may include:

- resource_type
- operation
- patient_id
- resource_id
- search_params

---

### FR-007: FHIR Output PHI Inspection

FHIR tool output shall pass through PHI inspection before model use.

The MVP may reuse PHIScrubberGuard at TOOL_OUTPUT.

---

### FR-008: FHIR Governance Metadata

The system shall attach metadata for FHIR decisions.

Required metadata keys:

- guard_category
- tool_name
- tool_allowed
- fhir_access_requested
- fhir_resource_type
- fhir_operation
- fhir_scope_required
- fhir_scope_present
- patient_context_required
- patient_context_present
- patient_scope_match
- policy_id
- policy_decision
- reason_code
- audit_required
- risk_level

---

### FR-009: Synthetic Mock FHIR Demo

The system shall include a mock FHIR read-only demo.

The demo shall not connect to a real FHIR server.

The demo shall use synthetic FHIR-like JSON.

The demo shall show:

- allowed read
- blocked write
- blocked missing scope
- PHI inspection on tool output
- audit-safe metadata

---

## Non-Functional Requirements

### NFR-001: No Real Patient Data

The feature shall use synthetic data only.

---

### NFR-002: No Real EHR Connection

The MVP shall use a mock FHIR tool.

---

### NFR-003: Fail Closed

If a FHIR authorization guard fails unexpectedly, it shall block in ENFORCE mode.

---

### NFR-004: Metadata Safety

No raw FHIR payload or raw PHI shall be stored in metadata.

---

### NFR-005: Minimal Runtime Refactor

Do not rewrite the core SecurityPipeline.

Implement FHIR governance as MedMesh guards and profile configuration.

---

## Proposed New Files

Implementation files expected later:

    src/agentguard/medmesh/guards/fhir.py
    src/agentguard/medmesh/profiles/fhir_readonly.py
    examples/medmesh_fhir_readonly_demo.py
    tests/medmesh/test_fhir_guards.py
    tests/medmesh/test_fhir_readonly_profile.py

Spec files:

    specs/fhir-readonly-governance/requirements.md
    specs/fhir-readonly-governance/design.md
    specs/fhir-readonly-governance/tasks.md

---

## Required Existing Foundation

This feature should build on:

- AgentContext
- GuardResult
- GuardVerdict
- InterceptPoint
- SecurityGuard
- SecurityPipeline
- ToolScopeGuard behavior
- PHIScrubberGuard
- metadata keys
- recognizer registry

---

## MVP Acceptance Criteria

The MVP is acceptable when:

1. FHIR read operation is allowed when scope and patient context match.
2. FHIR write operation is blocked.
3. Missing required scope blocks access.
4. Missing patient context blocks patient-scoped access.
5. Mismatched patient context blocks access.
6. Unapproved resource type blocks access.
7. Tool output PHI inspection runs.
8. Metadata is audit-safe.
9. Mock FHIR demo runs locally.
10. Tests pass.

---

## Open Questions

1. Should patient context be added to AgentContext or passed through tool args first?
2. Should FHIR scopes live in AgentContext, guard config, or metadata?
3. Should the first implementation use a MedMeshHealthcareContext wrapper?
4. Should search be allowed by default or treated as higher risk than read?
5. Should Patient resource access be treated differently from other resources?
6. Should DiagnosticReport and Observation be assigned higher sensitivity?
7. Should FHIR Bundle outputs be summarized before PHI inspection?
8. Should the FHIR profile compose the PHI Gateway profile or duplicate parts of it?

---

## Spec Decision Log

### Decision 1: Mock FHIR First

Decision:

    Use a mock FHIR tool and synthetic FHIR-like JSON first.

Reason:

    Avoid EHR complexity and privacy risk while proving governance semantics.

---

### Decision 2: Read-Only First

Decision:

    Only read and search operations are allowed in the first FHIR governance feature.

Reason:

    Write operations require stronger review, authorization, audit, and clinical safety controls.

---

### Decision 3: Scope-Inspired, Not Full OAuth Yet

Decision:

    MVP uses simplified SMART/FHIR-like scopes rather than full OAuth token validation.

Reason:

    Token validation should come later with identity architecture.

---

### Decision 4: Reuse PHI Gateway

Decision:

    FHIR tool output inspection should reuse PHIScrubberGuard.

Reason:

    FHIR outputs may contain PHI and must be governed before model use.

---

## Next Spec Files

Create:

    specs/fhir-readonly-governance/design.md
    specs/fhir-readonly-governance/tasks.md

