# Spec: MedMesh FHIR Read-Only Governance Tasks

## Status

Draft

## Related Specs

- specs/fhir-readonly-governance/requirements.md
- specs/fhir-readonly-governance/design.md

## Feature

MedMesh FHIR Read-Only Governance

## Objective

Break the FHIR Read-Only Governance feature into concrete engineering tasks.

This feature should be implemented after the PHI Gateway MVP is stable.

Recommended implementation branch in the code repo:

    feature/medmesh-fhir-readonly-governance

---

## Pre-Implementation Checklist

Before implementation begins in the code repo:

- PHI Gateway MVP is merged or stable
- PHI Gateway tests pass
- recognizer registry is working
- FHIR requirements spec exists
- FHIR design spec exists
- FHIR tasks spec exists
- current code branch is clean
- new feature branch is created

Recommended branch command in code repo:

    git checkout -b feature/medmesh-fhir-readonly-governance

---

# Task Group 1: Create FHIR Guard Module

## Task 1.1: Create FHIR Guard File

Create in code repo:

    src/agentguard/medmesh/guards/fhir.py

Acceptance:

- file exists
- module imports successfully
- no core runtime contracts are modified

---

## Task 1.2: Export FHIR Guards

Update in code repo:

    src/agentguard/medmesh/guards/__init__.py

Acceptance:

- FHIR guards can be imported from agentguard.medmesh.guards
- existing PHI guard imports still work

---

# Task Group 2: Add FHIRAccessRequest Model

## Task 2.1: Define FHIRAccessRequest Dataclass

Create a dataclass in:

    src/agentguard/medmesh/guards/fhir.py

Fields:

- tool_name
- resource_type
- operation
- patient_id
- resource_id
- search_params
- raw_operation
- raw_resource_type

Acceptance:

- dataclass is immutable if practical
- defaults are safe
- no raw FHIR payload is stored beyond tool args normalization

---

## Task 2.2: Implement Tool Argument Parser

Implement function:

    parse_fhir_access_request(tool_name: str, args: dict) -> FHIRAccessRequest

It should support common argument aliases:

Resource type aliases:

- resource_type
- resource
- type

Operation aliases:

- operation
- action
- method

Patient ID aliases:

- patient_id
- patient
- subject

Resource ID aliases:

- resource_id
- id

Search params aliases:

- search_params
- params
- query

Acceptance:

- valid args parse correctly
- missing resource_type is handled safely
- missing operation is handled safely
- patient_id can be extracted from search params if present

---

# Task Group 3: Implement Shared Metadata Helpers

## Task 3.1: Add FHIR Pass Metadata Helper

Implement helper for allowed FHIR decisions.

Acceptance:

- metadata uses existing keys
- includes tool_name
- includes fhir_resource_type
- includes fhir_operation
- includes policy_id
- policy_decision = allow
- audit_required = true
- no raw payloads included

---

## Task 3.2: Add FHIR Block Metadata Helper

Implement helper for blocked FHIR decisions.

Acceptance:

- metadata uses existing keys
- includes reason_code
- includes human_readable_reason
- policy_decision = deny
- audit_required = true
- risk_level = high
- no raw payloads included

---

# Task Group 4: Implement FHIRReadOnlyGuard

## Task 4.1: Define FHIRReadOnlyGuard

Class:

    FHIRReadOnlyGuard

Base:

    SecurityGuard

Intercept points:

- TOOL_AUTH
- PRE_TOOL

Category:

    4

Acceptance:

- class imports successfully
- check_tool is implemented
- check can safely pass or delegate

---

## Task 4.2: Allow Read Operations

Allowed operations:

- read
- search

Acceptance:

- read passes
- search passes
- metadata includes fhir_operation
- metadata includes policy_decision = allow

---

## Task 4.3: Block Write Operations

Blocked operations:

- create
- update
- patch
- delete
- transaction
- batch
- bulk_export

Acceptance:

- blocked operations return GuardVerdict.BLOCK
- metadata reason_code = fhir_operation_not_allowed
- audit_required = true

---

## Task 4.4: Block Unknown Operations

Unknown or missing operation should block.

Acceptance:

- missing operation blocks
- unknown operation blocks
- metadata reason_code = fhir_operation_not_allowed or fhir_request_invalid

---

# Task Group 5: Implement FHIRResourceScopeGuard

## Task 5.1: Define FHIRResourceScopeGuard

Class:

    FHIRResourceScopeGuard

Base:

    SecurityGuard

Intercept points:

- TOOL_AUTH
- PRE_TOOL

Category:

    4

Acceptance:

- class imports successfully
- allowed_resource_types config is supported

---

## Task 5.2: Allow Configured Resource Types

Default allowed resources:

- Patient
- Observation
- Condition
- MedicationRequest
- Encounter
- AllergyIntolerance
- DiagnosticReport

Acceptance:

- Observation passes
- Patient passes
- Condition passes

---

## Task 5.3: Block Unconfigured Resource Types

Example blocked resource:

- Binary
- DocumentReference
- AuditEvent
- Provenance
- Consent, unless explicitly allowed

Acceptance:

- unconfigured resource returns BLOCK
- metadata reason_code = fhir_resource_not_allowed

---

# Task Group 6: Implement PatientContextGuard

## Task 6.1: Define PatientContextGuard

Class:

    PatientContextGuard

Base:

    SecurityGuard

Intercept points:

- TOOL_AUTH
- PRE_TOOL

Category:

    4

Acceptance:

- class imports successfully
- configuration supports authorized_patient_id
- configuration supports require_patient_context
- configuration supports allow_population_search

---

## Task 6.2: Allow Matching Patient Context

Given:

    authorized_patient_id = patient-123
    args.patient_id = patient-123

Expected:

- PASS
- patient_context_present = true
- patient_scope_match = true

---

## Task 6.3: Block Missing Patient Context

Given patient context required and no patient_id present:

Expected:

- BLOCK
- reason_code = patient_context_missing

---

## Task 6.4: Block Patient Scope Mismatch

Given:

    authorized_patient_id = patient-123
    args.patient_id = patient-999

Expected:

- BLOCK
- reason_code = patient_scope_mismatch

---

## Task 6.5: Block Population Search By Default

Given search operation and no patient constraint:

Expected:

- BLOCK
- reason_code = population_search_not_allowed

Unless:

    allow_population_search = true

---

# Task Group 7: Implement FHIRScopeGuard

## Task 7.1: Define FHIRScopeGuard

Class:

    FHIRScopeGuard

Base:

    SecurityGuard

Intercept points:

- TOOL_AUTH
- PRE_TOOL

Category:

    4

Acceptance:

- class imports successfully
- granted_scopes config is supported

---

## Task 7.2: Generate Required Scope

For resource type and read/search operation, generate:

    patient/{ResourceType}.read

Examples:

- patient/Patient.read
- patient/Observation.read
- patient/Condition.read

Acceptance:

- required scope is deterministic
- metadata includes fhir_scope_required

---

## Task 7.3: Allow Present Scope

Given:

    granted_scopes = ["patient/Observation.read"]

and request:

    Observation read

Expected:

- PASS
- fhir_scope_present = true

---

## Task 7.4: Block Missing Scope

Given missing required scope:

Expected:

- BLOCK
- reason_code = fhir_scope_missing
- fhir_scope_present = false

---

# Task Group 8: Add FHIR Read-Only Profile

## Task 8.1: Create Profile File

Create in code repo:

    src/agentguard/medmesh/profiles/fhir_readonly.py

Acceptance:

- module imports successfully

---

## Task 8.2: Implement Profile Factory

Function:

    build_fhir_readonly_pipeline(config: dict | None = None) -> SecurityPipeline

Acceptance:

- returns SecurityPipeline
- includes FHIR guards
- includes PHIScrubberGuard for TOOL_OUTPUT
- includes HealthcareAuditGuard for ASYNC

---

## Task 8.3: Export Profile Factory

Update:

    src/agentguard/medmesh/profiles/__init__.py

Acceptance:

- build_fhir_readonly_pipeline is importable
- existing external strict profile export still works

---

## Task 8.4: Configure Profile Defaults

Default config:

    allowed_resource_types:
      - Patient
      - Observation
      - Condition
      - MedicationRequest
      - Encounter
      - AllergyIntolerance
      - DiagnosticReport

    granted_scopes:
      - patient/Patient.read
      - patient/Observation.read
      - patient/Condition.read
      - patient/MedicationRequest.read
      - patient/Encounter.read

    authorized_patient_id:
      patient-123

    allow_population_search:
      false

Acceptance:

- profile works with no config
- custom config can override defaults

---

# Task Group 9: Create Mock FHIR Demo

## Task 9.1: Create Demo File

Create:

    examples/medmesh_fhir_readonly_demo.py

Acceptance:

- file exists
- imports fhir readonly profile

---

## Task 9.2: Add Mock FHIR Read Function

Function:

    mock_fhir_read(resource_type, patient_id, resource_id=None)

Acceptance:

- returns synthetic FHIR-like JSON
- no real FHIR server is called
- no real patient data is used

---

## Task 9.3: Demo Scenario 1: Allowed Read

Scenario:

- resource_type = Observation
- operation = read
- patient_id = patient-123
- scope = patient/Observation.read

Expected:

- TOOL_AUTH passes
- PRE_TOOL passes
- mock tool executes
- TOOL_OUTPUT PHI inspection runs
- output is masked if PHI appears

---

## Task 9.4: Demo Scenario 2: Blocked Write

Scenario:

- resource_type = Observation
- operation = update

Expected:

- blocked before tool call
- reason_code = fhir_operation_not_allowed

---

## Task 9.5: Demo Scenario 3: Missing Scope

Scenario:

- Observation read
- missing patient/Observation.read scope

Expected:

- blocked
- reason_code = fhir_scope_missing

---

## Task 9.6: Demo Scenario 4: Patient Mismatch

Scenario:

- authorized_patient_id = patient-123
- requested patient_id = patient-999

Expected:

- blocked
- reason_code = patient_scope_mismatch

---

# Task Group 10: Add Tests

## Task 10.1: Create Test Files

Create:

    tests/medmesh/test_fhir_guards.py
    tests/medmesh/test_fhir_readonly_profile.py

Acceptance:

- files exist
- tests run under pytest

---

## Task 10.2: Test FHIRAccessRequest Parser

Test:

- standard args parse
- alias args parse
- search params parse
- missing values handled safely

---

## Task 10.3: Test FHIRReadOnlyGuard

Test:

- read passes
- search passes
- update blocks
- delete blocks
- unknown operation blocks

---

## Task 10.4: Test FHIRResourceScopeGuard

Test:

- allowed resource passes
- blocked resource blocks
- metadata contains fhir_resource_type

---

## Task 10.5: Test PatientContextGuard

Test:

- matching patient passes
- missing patient blocks
- mismatched patient blocks
- population search blocks by default
- population search passes if enabled

---

## Task 10.6: Test FHIRScopeGuard

Test:

- required scope present passes
- required scope missing blocks
- metadata includes fhir_scope_required
- metadata includes fhir_scope_present

---

## Task 10.7: Test FHIR Read-Only Profile

Test:

- profile builds SecurityPipeline
- allowed read path passes TOOL_AUTH and PRE_TOOL
- blocked write path blocks
- TOOL_OUTPUT PHI inspection modifies synthetic FHIR output

---

# Task Group 11: Documentation Updates

## Task 11.1: Update README

In code repo README, add:

- FHIR Read-Only Governance section
- safety notice
- demo command
- test command

---

## Task 11.2: Add Example Documentation

Create in docs/spec repo or docs repo:

    docs/examples/fhir-readonly-governance.md

Include:

- purpose
- safety assumptions
- demo scenarios
- expected behavior
- known limitations

---

# Task Group 12: Validation

## Task 12.1: Run FHIR Tests

Command in code repo:

    PYTHONPATH=src pytest tests/medmesh/test_fhir_guards.py tests/medmesh/test_fhir_readonly_profile.py -q

Acceptance:

- all FHIR tests pass

---

## Task 12.2: Run All MedMesh Tests

Command in code repo:

    PYTHONPATH=src pytest tests/medmesh -q

Acceptance:

- all MedMesh tests pass

---

## Task 12.3: Run FHIR Demo

Command in code repo:

    PYTHONPATH=src python examples/medmesh_fhir_readonly_demo.py

Acceptance:

- allowed read passes
- blocked write blocks
- missing scope blocks
- patient mismatch blocks
- tool output PHI inspection runs

---

# Branching Guidance

Create this branch in the code repo when ready to implement:

    git checkout -b feature/medmesh-fhir-readonly-governance

Do not implement FHIR read-only governance directly on the PHI Gateway branch.

---

# Definition Of Done

FHIR Read-Only Governance MVP is complete when:

1. FHIRAccessRequest parser exists.
2. FHIRReadOnlyGuard works.
3. FHIRResourceScopeGuard works.
4. PatientContextGuard works.
5. FHIRScopeGuard works.
6. FHIR read-only profile builds a pipeline.
7. Mock FHIR demo runs.
8. FHIR tool output passes through PHIScrubberGuard.
9. FHIR tests pass.
10. All MedMesh tests pass.
11. Docs are updated.
12. No real patient data or real FHIR server is required.

