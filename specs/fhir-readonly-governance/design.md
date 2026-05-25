# Spec: MedMesh FHIR Read-Only Governance Design

## Status

Draft

## Related Requirements

This design implements:

    specs/fhir-readonly-governance/requirements.md

## Feature Name

MedMesh FHIR Read-Only Governance

## Target Capability

Governed read-only access to FHIR resources for healthcare AI agents.

---

## Design Objective

Design a MedMesh runtime governance layer that controls FHIR read access before and after healthcare AI agents call FHIR tools.

The design should prove that MedMesh can govern:

- healthcare tool authorization
- FHIR resource access
- read-only operation enforcement
- patient-scoped access
- SMART/FHIR-like scope validation
- FHIR tool output PHI inspection
- audit-safe governance metadata

This feature should build on the existing PHI Gateway MVP and should not require a full runtime rewrite.

---

## Existing Foundation

This feature builds on:

- AgentContext
- GuardResult
- GuardVerdict
- GuardMode
- InterceptPoint
- SecurityGuard
- SecurityPipeline
- MedMesh metadata keys
- PHIScrubberGuard
- HealthcareAuditGuard
- MedMesh recognizer registry
- medmesh-external-model-strict profile concepts

---

## Design Principle

FHIR governance should initially be implemented as MedMesh guards and profiles.

Do not modify the core SecurityPipeline yet.

The first FHIR governance feature should be:

- mock-based
- read-only
- synthetic-data only
- scope-inspired
- patient-context aware
- metadata-safe

---

## Proposed Module Structure

Implementation files should live in the code repo.

Expected future files:

    src/agentguard/medmesh/guards/fhir.py
    src/agentguard/medmesh/profiles/fhir_readonly.py
    examples/medmesh_fhir_readonly_demo.py
    tests/medmesh/test_fhir_guards.py
    tests/medmesh/test_fhir_readonly_profile.py

Spec files live in the docs/spec repo:

    specs/fhir-readonly-governance/requirements.md
    specs/fhir-readonly-governance/design.md
    specs/fhir-readonly-governance/tasks.md

---

## Component Overview

The MVP design includes:

1. FHIRAccessRequest model
2. FHIRReadOnlyGuard
3. FHIRResourceScopeGuard
4. PatientContextGuard
5. FHIRToolOutputPHIGuard behavior through PHIScrubberGuard
6. FHIR read-only profile
7. Mock FHIR tool
8. Synthetic FHIR demo
9. Tests

---

# 1. FHIRAccessRequest

## Purpose

Normalize FHIR tool arguments into a small internal request object that guards can reason about consistently.

## Why This Matters

FHIR tools may pass arguments in different shapes.

Example tool args may look like:

    {
      "resource_type": "Observation",
      "operation": "read",
      "patient_id": "patient-123",
      "resource_id": "obs-001"
    }

or:

    {
      "resource": "Observation",
      "action": "search",
      "params": {
        "patient": "patient-123",
        "category": "vital-signs"
      }
    }

Instead of spreading parsing logic across multiple guards, MedMesh should normalize tool arguments first.

## Proposed Fields

FHIRAccessRequest should contain:

- tool_name
- resource_type
- operation
- patient_id
- resource_id
- search_params
- raw_operation
- raw_resource_type

## Initial Implementation Approach

For MVP, this can be a simple dataclass in:

    src/agentguard/medmesh/guards/fhir.py

Potential future location:

    src/agentguard/medmesh/fhir/models.py

## Example

    FHIRAccessRequest(
        tool_name="fhir_read",
        resource_type="Observation",
        operation="read",
        patient_id="patient-123",
        resource_id="obs-001",
        search_params={}
    )

---

# 2. FHIRReadOnlyGuard

## File

    src/agentguard/medmesh/guards/fhir.py

## Class

    FHIRReadOnlyGuard

## Base Class

    SecurityGuard

## Purpose

Block non-read FHIR operations in the read-only profile.

## Intercept Points

Default intercept points:

- TOOL_AUTH
- PRE_TOOL

## Category

Recommended category:

    4

Reason:

This is an authorization and scope-style guard.

## Allowed Operations

Allowed:

- read
- search

Blocked:

- create
- update
- patch
- delete
- transaction
- batch
- bulk_export

## Check Method

This guard should primarily implement:

    check_tool(tool_name, args, context)

## Behavior

If operation is read or search:

- return PASS
- metadata policy_decision = allow

If operation is write-like or unknown:

- return BLOCK
- metadata policy_decision = deny
- reason_code = fhir_operation_not_allowed

## Metadata For PASS

- guard_category = fhir
- fhir_access_requested = true
- tool_name
- fhir_resource_type
- fhir_operation
- policy_id = fhir.readonly.operation.v1
- policy_decision = allow
- audit_required = true
- risk_level = low

## Metadata For BLOCK

- guard_category = fhir
- fhir_access_requested = true
- tool_name
- fhir_resource_type
- fhir_operation
- policy_id = fhir.readonly.operation.v1
- policy_decision = deny
- reason_code = fhir_operation_not_allowed
- human_readable_reason
- audit_required = true
- risk_level = high

---

# 3. FHIRResourceScopeGuard

## File

    src/agentguard/medmesh/guards/fhir.py

## Class

    FHIRResourceScopeGuard

## Base Class

    SecurityGuard

## Purpose

Enforce allowed FHIR resource types.

## Intercept Points

Default intercept points:

- TOOL_AUTH
- PRE_TOOL

## Category

Recommended category:

    4

## Configuration

Options:

- allowed_resource_types
- policy_id

Default allowed_resource_types:

- Patient
- Observation
- Condition
- MedicationRequest
- Encounter
- AllergyIntolerance
- DiagnosticReport

## Behavior

If requested resource type is in allowed_resource_types:

- return PASS

If requested resource type is not allowed:

- return BLOCK

## Metadata For PASS

- guard_category = fhir
- fhir_access_requested = true
- fhir_resource_type
- policy_id = fhir.resource_scope.v1
- policy_decision = allow
- audit_required = true
- risk_level = low

## Metadata For BLOCK

- guard_category = fhir
- fhir_access_requested = true
- fhir_resource_type
- policy_id = fhir.resource_scope.v1
- policy_decision = deny
- reason_code = fhir_resource_not_allowed
- audit_required = true
- risk_level = high

---

# 4. PatientContextGuard

## File

    src/agentguard/medmesh/guards/fhir.py

## Class

    PatientContextGuard

## Base Class

    SecurityGuard

## Purpose

Ensure patient-scoped FHIR access has valid patient context.

## Why This Matters

Healthcare AI agents should not perform broad or cross-patient reads unless explicitly authorized.

Patient context is essential for least-privilege access.

## Intercept Points

Default intercept points:

- TOOL_AUTH
- PRE_TOOL

## Category

Recommended category:

    4

## Configuration

Options:

- require_patient_context
- authorized_patient_id
- allow_population_search
- policy_id

Defaults:

    require_patient_context = true
    authorized_patient_id = null
    allow_population_search = false
    policy_id = fhir.patient_context.v1

## Design Decision

For MVP, patient context should be provided through guard configuration or tool args.

Do not modify AgentContext yet.

Reason:

- AgentContext does not currently support healthcare fields
- changing AgentContext affects the runtime core
- MVP can prove the behavior with tool args and guard config first

## Behavior

If patient context is required and no patient_id is present:

- return BLOCK

If authorized_patient_id is configured and requested patient_id differs:

- return BLOCK

If patient_id matches:

- return PASS

If allow_population_search is false and search has no patient constraint:

- return BLOCK

## Metadata For PASS

- guard_category = fhir
- patient_context_required = true
- patient_context_present = true
- patient_scope_match = true
- policy_id = fhir.patient_context.v1
- policy_decision = allow
- audit_required = true
- risk_level = low

## Metadata For BLOCK

- guard_category = fhir
- patient_context_required
- patient_context_present
- patient_scope_match
- policy_id = fhir.patient_context.v1
- policy_decision = deny
- reason_code
- audit_required = true
- risk_level = high

Possible reason_code values:

- patient_context_missing
- patient_scope_mismatch
- population_search_not_allowed

---

# 5. FHIRScopeGuard

## File

    src/agentguard/medmesh/guards/fhir.py

## Class

    FHIRScopeGuard

## Base Class

    SecurityGuard

## Purpose

Validate simplified SMART/FHIR-like scopes for the requested resource and operation.

## Intercept Points

Default intercept points:

- TOOL_AUTH
- PRE_TOOL

## Category

Recommended category:

    4

## Configuration

Options:

- granted_scopes
- scope_mode
- policy_id

Default:

    granted_scopes = []
    scope_mode = patient
    policy_id = fhir.scope.v1

## Required Scope Format

For MVP, generate required scopes like:

    patient/{ResourceType}.read

Examples:

- patient/Patient.read
- patient/Observation.read
- patient/Condition.read
- patient/MedicationRequest.read

## Search Operation

For search, MVP should still require:

    patient/{ResourceType}.read

Reason:

Search returns resource data and should be governed like read access.

## Behavior

If required scope exists in granted_scopes:

- return PASS

If required scope is missing:

- return BLOCK

## Metadata For PASS

- guard_category = fhir
- fhir_resource_type
- fhir_operation
- fhir_scope_required
- fhir_scope_present = true
- policy_id = fhir.scope.v1
- policy_decision = allow
- audit_required = true
- risk_level = low

## Metadata For BLOCK

- guard_category = fhir
- fhir_resource_type
- fhir_operation
- fhir_scope_required
- fhir_scope_present = false
- policy_id = fhir.scope.v1
- policy_decision = deny
- reason_code = fhir_scope_missing
- audit_required = true
- risk_level = high

---

# 6. Tool Output PHI Inspection

## Existing Guard

Use:

    PHIScrubberGuard

## Intercept Point

    TOOL_OUTPUT

## Purpose

FHIR tool outputs may contain PHI.

Before those outputs enter the LLM context, MedMesh should inspect and transform them.

## Behavior

If FHIR output contains PHI:

- return MODIFY
- masked output becomes safe tool output
- metadata records PHI types and counts

If no PHI is detected:

- return PASS

## Design Decision

Do not create a separate FHIRToolOutputPHIGuard in MVP.

Reuse PHIScrubberGuard.

Reason:

- avoids duplicate PHI logic
- validates composability of the PHI Gateway
- keeps the FHIR feature focused on authorization

---

# 7. FHIR Read-Only Profile

## File

    src/agentguard/medmesh/profiles/fhir_readonly.py

## Function

    build_fhir_readonly_pipeline(config: dict | None = None) -> SecurityPipeline

## Purpose

Create a pipeline for mock FHIR read-only governance.

## Required Guards

TOOL_AUTH:

- ToolScopeGuard if easy to compose
- FHIRReadOnlyGuard
- FHIRResourceScopeGuard
- FHIRScopeGuard
- PatientContextGuard

PRE_TOOL:

- ToolArgumentSchemaGuard if compatible
- FHIRReadOnlyGuard
- FHIRResourceScopeGuard
- FHIRScopeGuard
- PatientContextGuard

TOOL_OUTPUT:

- PHIScrubberGuard

ASYNC:

- HealthcareAuditGuard

## Configuration Shape

Example:

    {
      "fhir": {
        "allowed_resource_types": [
          "Patient",
          "Observation",
          "Condition",
          "MedicationRequest",
          "Encounter"
        ],
        "granted_scopes": [
          "patient/Patient.read",
          "patient/Observation.read"
        ],
        "authorized_patient_id": "patient-123",
        "allow_population_search": false
      },
      "phi": {
        "sensitivity_threshold": 0.7,
        "handling_mode": "mask"
      },
      "audit": {
        "sink": "stdout"
      }
    }

---

# 8. Mock FHIR Tool

## File

    examples/medmesh_fhir_readonly_demo.py

## Purpose

Simulate a FHIR read without connecting to a real server.

## Mock Function

Example:

    def mock_fhir_read(resource_type, patient_id, resource_id=None):
        return {
            "resourceType": "Observation",
            "id": "obs-001",
            "subject": {"reference": "Patient/patient-123"},
            "patientName": "John Smith",
            "effectiveDateTime": "2024-03-03",
            "code": {"text": "Blood pressure"},
            "valueString": "150/90"
        }

## Important

The mock output must be synthetic only.

---

# 9. Demo Flow

## File

    examples/medmesh_fhir_readonly_demo.py

## Flow

1. Create AgentContext.for_testing()
2. Build fhir-readonly pipeline
3. Define allowed FHIR read args
4. Run TOOL_AUTH
5. Run PRE_TOOL
6. Execute mock FHIR read
7. Serialize FHIR response to string
8. Run TOOL_OUTPUT PHI inspection
9. Print protected tool output
10. Run ASYNC audit tail

## Demo Scenarios

The demo should show:

### Scenario 1: Allowed FHIR Read

- Observation read
- patient_id matches
- scope exists
- operation is read
- output is inspected and masked

### Scenario 2: Blocked Write

- operation update
- should block before tool call

### Scenario 3: Missing Scope

- Observation read without patient/Observation.read
- should block

### Scenario 4: Patient Mismatch

- requested patient_id differs from authorized_patient_id
- should block

---

# 10. Testing Design

## Test Files

    tests/medmesh/test_fhir_guards.py
    tests/medmesh/test_fhir_readonly_profile.py

## Test Cases

### FHIRReadOnlyGuard

- read passes
- search passes
- update blocks
- delete blocks
- unknown operation blocks

### FHIRResourceScopeGuard

- allowed resource passes
- disallowed resource blocks

### PatientContextGuard

- matching patient passes
- missing patient blocks
- mismatched patient blocks
- population search blocks when not allowed

### FHIRScopeGuard

- required scope present passes
- missing scope blocks
- scope metadata is attached

### Profile

- builds SecurityPipeline
- allowed read path passes TOOL_AUTH and PRE_TOOL
- blocked write path blocks
- TOOL_OUTPUT PHI inspection modifies synthetic FHIR output

---

# 11. Metadata Design

FHIR guards should use existing keys from:

    src/agentguard/medmesh/metadata/keys.py

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
- policy_version
- policy_decision
- reason_code
- human_readable_reason
- audit_required
- risk_level

## Raw Payload Safety

Metadata must not include:

- raw FHIR response
- patient name
- address
- phone number
- raw clinical note
- raw observation value if sensitive

---

# 12. Error Handling

## Missing Args

If required tool args are missing:

- BLOCK
- reason_code = fhir_request_invalid

## Unsupported Operation

If operation is unsupported:

- BLOCK
- reason_code = fhir_operation_not_allowed

## Missing Scope

If required scope is missing:

- BLOCK
- reason_code = fhir_scope_missing

## Patient Context Missing

If patient context is required but missing:

- BLOCK
- reason_code = patient_context_missing

## Patient Scope Mismatch

If patient_id does not match authorized_patient_id:

- BLOCK
- reason_code = patient_scope_mismatch

---

# 13. Known MVP Limitations

The MVP will not provide:

- real SMART on FHIR launch
- real OAuth2/OIDC token validation
- real EHR integration
- real HAPI FHIR integration
- production-grade consent management
- FHIR write support
- clinical decision support
- immutable audit trail
- full FHIR Permission support
- bulk export governance

---

# 14. Future Extensions

Future FHIR governance versions may add:

- SMART on FHIR token parsing
- Keycloak/OIDC integration
- HAPI FHIR demo server
- FHIR Bundle support
- FHIR R6 Permission evaluation
- Consent resource evaluation
- Provenance resource generation
- AuditEvent resource generation
- write-controlled profile
- human-in-the-loop write approval
- bulk export guardrails
- resource sensitivity classification

---

## Design Decisions

### DD-001: Mock FHIR First

Decision:

    Use a synthetic mock FHIR tool first.

Reason:

    It proves governance semantics without EHR complexity.

---

### DD-002: Do Not Modify AgentContext Yet

Decision:

    Pass patient and scope context through guard config and tool args first.

Reason:

    Avoid core runtime changes until the need is proven.

---

### DD-003: Reuse PHIScrubberGuard For TOOL_OUTPUT

Decision:

    Use existing PHI Gateway for FHIR output inspection.

Reason:

    Validates composability and avoids duplicate PHI logic.

---

### DD-004: Read And Search Are Allowed

Decision:

    Allow read and search as read-only operations.

Reason:

    Both are retrieval operations, but search must remain patient-scoped unless population access is explicitly enabled.

---

### DD-005: Block Writes By Default

Decision:

    Block create, update, patch, delete, transaction, batch write, and bulk export.

Reason:

    Write governance requires separate safety, authorization, and human review design.

---

## Next Spec

Create:

    specs/fhir-readonly-governance/tasks.md

