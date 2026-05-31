# FHIR Read-Only Governance Demo

## Objective

Demonstrate MedMesh FHIR Read-Only Governance using synthetic FHIR-like data.

This demo shows how MedMesh can govern healthcare AI tool access before and after a mock FHIR read operation.

---

## Safety Notice

This demo uses synthetic data only.

It does not connect to a real FHIR server.

It does not validate real SMART on FHIR or OAuth tokens.

It is not for clinical use.

It does not claim:

- HIPAA compliance
- certified FHIR compliance
- production EHR readiness
- clinical validation
- patient data safety

It is a research and engineering prototype for healthcare AI governance.

---

## Profile Used

The demo uses:

    medmesh-fhir-readonly

This profile is designed for synthetic or mock FHIR workflows where AI agents need governed read-only access to healthcare resources.

---

## What The Demo Proves

The demo proves that MedMesh can enforce:

- read-only FHIR operation boundaries
- allowed FHIR resource types
- simplified SMART/FHIR-like scope requirements
- patient-context matching
- PHI inspection on tool outputs
- audit-safe metadata generation

---

## Demo Flow

The demo runs four scenarios.

---

## Scenario 1: Allowed FHIR Read

Input tool args:

    {
      "resource_type": "Observation",
      "operation": "read",
      "patient_id": "patient-123",
      "resource_id": "obs-001"
    }

Expected behavior:

- TOOL_AUTH passes
- PRE_TOOL passes
- mock FHIR read executes
- TOOL_OUTPUT inspection runs
- PHI in mock FHIR output is masked

This demonstrates the governed happy path.

---

## Scenario 2: Blocked FHIR Write

Input tool args:

    {
      "resource_type": "Observation",
      "operation": "update",
      "patient_id": "patient-123",
      "resource_id": "obs-001"
    }

Expected behavior:

- operation is blocked
- tool is not executed
- reason_code is fhir_operation_not_allowed

This demonstrates read-only enforcement.

---

## Scenario 3: Missing Scope

Example configuration:

    granted_scopes:
      - patient/Patient.read

Requested operation:

    Observation read

Expected required scope:

    patient/Observation.read

Expected behavior:

- access is blocked
- reason_code is fhir_scope_missing

This demonstrates simplified SMART/FHIR-like scope validation.

---

## Scenario 4: Patient Mismatch

Authorized patient:

    patient-123

Requested patient:

    patient-999

Expected behavior:

- access is blocked
- reason_code is patient_scope_mismatch

This demonstrates patient-context governance.

---

## Tool Output PHI Inspection

The mock FHIR read returns synthetic FHIR-like JSON containing values such as:

    John Smith
    Dr. Adams
    2024-03-03

Before this output enters model context, MedMesh runs PHI inspection at:

    TOOL_OUTPUT

Expected behavior:

- PHI-like content is detected
- output is modified
- raw synthetic identifiers are masked
- metadata records detected PHI types
- metadata does not store raw PHI

---

## Guards Used

The FHIR read-only profile uses:

- FHIRReadOnlyGuard
- FHIRResourceScopeGuard
- FHIRScopeGuard
- PatientContextGuard
- PHIScrubberGuard
- HealthcareAuditGuard

---

## How To Run

Install dependencies:

    python -m pip install -e ".[presidio]"

Run demo:

    PYTHONPATH=src python examples/medmesh_fhir_readonly_demo.py

Run tests:

    PYTHONPATH=src pytest tests/medmesh -q

---

## Current Known Limitations

The current implementation:

- does not connect to a real FHIR server
- does not validate real OAuth tokens
- does not perform real SMART App Launch
- does not support FHIR writes
- does not support consent resource evaluation
- does not support FHIR Permission evaluation
- does not emit real FHIR AuditEvent resources
- does not provide production-grade audit storage
- does not claim HIPAA compliance

---

## Next Improvements

Future FHIR governance work may include:

- real HAPI FHIR demo server integration
- SMART on FHIR token parsing
- Keycloak/OIDC integration
- Consent resource checks
- FHIR AuditEvent generation
- Provenance resource generation
- write-controlled profile
- human-in-the-loop write approval
- bulk export guardrails
