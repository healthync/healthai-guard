# Spec: MedMesh PHI Gateway Design

## Status

Draft

## Related Requirements

This design implements:

    specs/phi-gateway/requirements.md

## Feature Name

MedMesh PHI Gateway

## Target Profile

    medmesh-external-model-strict

---

## Design Objective

Design the first MedMesh healthcare-specific runtime extension:

    a PHI Gateway that detects, masks, blocks, inspects, and audits PHI across an external-model healthcare AI workflow.

The design must extend the existing AgentGuard runtime rather than replacing it.

---

## Existing Runtime Contracts

The PHI Gateway will build on the verified existing contracts:

- AgentContext
- GuardVerdict
- GuardResult
- GuardMode
- InterceptPoint
- SecurityGuard
- SecurityPipeline
- PipelineResult

The existing runtime already supports:

- PASS
- BLOCK
- MODIFY
- intercept-point based guard execution
- fail-closed guard behavior
- guard metadata
- async guard execution

---

## Design Principle

The PHI Gateway should be implemented as MedMesh-specific extensions inside the current package structure.

Do not rename the existing internal package from agentguard to medmesh yet.

Reason:

- avoid large refactor
- preserve existing tests
- keep MVP focused
- use MedMesh as extension namespace first

---

## Proposed Module Structure

Create:

    src/agentguard/medmesh/
    ├── __init__.py
    ├── guards/
    │   ├── __init__.py
    │   ├── phi.py
    │   ├── boundary.py
    │   └── audit.py
    ├── profiles/
    │   ├── __init__.py
    │   └── external_model_strict.py
    └── metadata/
        ├── __init__.py
        └── keys.py

Create test files:

    tests/medmesh/
    ├── test_phi_guard.py
    ├── test_boundary_guards.py
    ├── test_audit_guard.py
    └── test_external_model_strict_profile.py

Create demo file:

    examples/medmesh_phi_gateway_demo.py

---

## Component Overview

The MVP has five primary components:

1. Metadata constants
2. PHIScrubberGuard
3. PHIPromptBoundaryGuard
4. PHILeakageOutputGuard
5. HealthcareAuditGuard
6. External model strict profile
7. Synthetic demo

---

# 1. Metadata Constants

## File

    src/agentguard/medmesh/metadata/keys.py

## Purpose

Centralize metadata keys so guards do not create inconsistent dictionaries.

## Design

Use simple module-level string constants.

Example:

    PHI_DETECTED = "phi_detected"
    PHI_TYPES = "phi_types"
    ENTITY_COUNT = "entity_count"

## Why Constants Instead Of Classes

The existing GuardResult.metadata accepts generic dictionaries.

For the MVP, string constants are sufficient and avoid unnecessary abstraction.

Typed metadata classes can come later if patterns stabilize.

## Required Constants

Core:

- GUARD_CATEGORY
- ENFORCEMENT_MODE
- POLICY_ID
- POLICY_VERSION
- POLICY_DECISION
- AUDIT_REQUIRED
- RISK_LEVEL
- REASON_CODE
- HUMAN_READABLE_REASON

PHI:

- PHI_DETECTED
- PHI_TYPES
- PII_TYPES
- ENTITY_COUNT
- ENTITY_SUMMARY
- CONFIDENCE_THRESHOLD
- DETECTOR_BACKEND

Transformation:

- HANDLING_MODE
- ANONYMIZATION_OPERATOR
- REVERSIBLE
- SAFE_HARBOR_APPLIED
- SAFE_HARBOR_CATEGORIES

Audit:

- AUDIT_EVENT_ID
- AUDIT_EVENT_TYPE
- SESSION_ID
- TASK_ID
- TIMESTAMP
- GUARD_NAME
- VERDICT

---

# 2. PHIScrubberGuard

## File

    src/agentguard/medmesh/guards/phi.py

## Class

    PHIScrubberGuard

## Base Class

    SecurityGuard

## Purpose

Detect PHI/PII in text and mask it before the content can reach an external model boundary.

## Intercept Points

Default intercept points:

- INPUT
- TOOL_OUTPUT

Later:

- OUTPUT if needed

## Category

Recommended category:

    5

Reason:

Existing classifier guards are category 5.

PHIScrubberGuard is a classifier/anonymization guard and should run after cheaper deterministic guards.

## Configuration

Constructor/config options:

- sensitivity_threshold
- language
- handling_mode
- policy_id

Defaults:

    sensitivity_threshold = 0.7
    language = "en"
    handling_mode = "mask"
    policy_id = "phi.mask.v1"

## Backend

Initial backend:

    Presidio

Required imports:

- presidio_analyzer.AnalyzerEngine
- presidio_anonymizer.AnonymizerEngine

## Setup Behavior

During setup:

1. Import Presidio components.
2. Initialize AnalyzerEngine.
3. Initialize AnonymizerEngine.
4. If imports fail, raise a configuration error.

Do not silently disable PHI detection.

## Check Behavior

Input:

    text: str
    context: AgentContext

Flow:

1. If text is empty or whitespace, return PASS.
2. Analyze text using Presidio.
3. Filter entities by sensitivity_threshold.
4. If no entities are found:
   - return PASS
   - include metadata showing phi_detected = false
5. If entities are found:
   - anonymize or mask text
   - return MODIFY
   - set modified_input to protected text
   - include governance metadata

## Masking Behavior

For MVP, masking should produce placeholders.

Example:

    John Smith was admitted on March 3, 2024.

Becomes:

    [PERSON] was admitted on [DATE_TIME].

Note:

Presidio entity names may use DATE_TIME instead of DATE.

Do not over-normalize entity names in the first implementation unless necessary.

## Metadata

When PHI is detected, metadata should include:

- guard_category = "phi"
- phi_detected = true
- phi_types
- pii_types
- entity_count
- entity_summary
- confidence_threshold
- detector_backend = "presidio"
- handling_mode = "mask"
- anonymization_operator = "replace"
- policy_id
- policy_decision = "modify"
- audit_required = true
- risk_level = "medium"

When no PHI is detected:

- guard_category = "phi"
- phi_detected = false
- entity_count = 0
- detector_backend = "presidio"
- policy_decision = "allow"
- audit_required = false
- risk_level = "low"

## Raw PHI Safety

The metadata must not include:

- original text
- raw entity values
- raw clinical note
- detected string values

Only entity types, counts, and safe summaries are allowed.

## Failure Behavior

SecurityGuard already fail-closes on exceptions and timeout.

PHIScrubberGuard should rely on that behavior.

Do not catch errors and return PASS.

---

# 3. PHIPromptBoundaryGuard

## File

    src/agentguard/medmesh/guards/boundary.py

## Class

    PHIPromptBoundaryGuard

## Base Class

    SecurityGuard

## Purpose

Ensure raw PHI is not present at the PRE_LLM boundary when the model destination is external or untrusted.

## Intercept Point

Default intercept point:

- PRE_LLM

## Category

Recommended category:

    5

Reason:

This is a classification and policy boundary guard.

## Configuration

Options:

- sensitivity_threshold
- language
- policy_id
- detector_backend

Defaults:

    sensitivity_threshold = 0.7
    language = "en"
    policy_id = "model.external.no_raw_phi.v1"
    detector_backend = "presidio"

## Check Behavior

Input:

    prompt: str
    context: AgentContext

Flow:

1. If prompt is empty or whitespace, return PASS.
2. Analyze prompt for PHI/PII.
3. If no entities are detected:
   - return PASS
   - metadata policy_decision = "allow"
4. If entities are detected:
   - return BLOCK
   - metadata policy_decision = "deny"
   - reason_code = "raw_phi_external_model_boundary"
   - audit_required = true
   - risk_level = "high"

## Important Distinction

PHIScrubberGuard modifies content.

PHIPromptBoundaryGuard enforces the boundary.

This means the normal expected flow is:

1. PHIScrubberGuard masks PHI at INPUT.
2. PHIPromptBoundaryGuard verifies that raw PHI does not remain at PRE_LLM.

If raw PHI still exists at PRE_LLM, something failed and the system should block.

## Metadata

For BLOCK:

- guard_category = "phi_boundary"
- phi_detected = true
- phi_types
- entity_count
- entity_summary
- policy_id = "model.external.no_raw_phi.v1"
- policy_decision = "deny"
- reason_code = "raw_phi_external_model_boundary"
- human_readable_reason = "Raw PHI was detected before an external model call."
- audit_required = true
- risk_level = "high"

For PASS:

- guard_category = "phi_boundary"
- phi_detected = false
- entity_count = 0
- policy_id = "model.external.no_raw_phi.v1"
- policy_decision = "allow"
- audit_required = false
- risk_level = "low"

---

# 4. PHILeakageOutputGuard

## File

    src/agentguard/medmesh/guards/boundary.py

## Class

    PHILeakageOutputGuard

## Base Class

    SecurityGuard

## Purpose

Inspect model output before it is returned to the user.

## Intercept Point

Default intercept point:

- OUTPUT

## Category

Recommended category:

    5

## Configuration

Options:

- sensitivity_threshold
- language
- policy_id
- on_leakage

Defaults:

    sensitivity_threshold = 0.7
    language = "en"
    policy_id = "output.no_phi_leakage.v1"
    on_leakage = "block"

## Check Behavior

Input:

    output: str
    context: AgentContext

Flow:

1. If output is empty or whitespace, return PASS.
2. Analyze output for PHI/PII.
3. If no entities are detected:
   - return PASS
4. If entities are detected:
   - for MVP, return BLOCK
   - include leakage metadata

## Why Block Instead Of Modify

Blocking is safer for the first MVP.

Later, the guard may support:

- redact
- mask
- allow with audit
- require review

## Metadata For BLOCK

- guard_category = "output_phi_leakage"
- phi_detected = true
- phi_types
- entity_count
- entity_summary
- policy_id = "output.no_phi_leakage.v1"
- policy_decision = "deny"
- reason_code = "phi_leakage_in_model_output"
- human_readable_reason = "PHI-like content was detected in model output."
- audit_required = true
- risk_level = "high"

## Metadata For PASS

- guard_category = "output_phi_leakage"
- phi_detected = false
- entity_count = 0
- policy_id = "output.no_phi_leakage.v1"
- policy_decision = "allow"
- audit_required = false
- risk_level = "low"

---

# 5. HealthcareAuditGuard

## File

    src/agentguard/medmesh/guards/audit.py

## Class

    HealthcareAuditGuard

## Base Class

    SecurityGuard

## Purpose

Emit audit-safe governance events.

## Intercept Point

Default intercept point:

- ASYNC

## Category

Recommended category:

    7

Reason:

Monitoring guards are category 7 in the existing architecture.

## Configuration

Options:

- sink
- file_path
- include_pass_events

Defaults:

    sink = "stdout"
    file_path = null
    include_pass_events = false

## Check Behavior

Input:

    text: str
    context: AgentContext

Flow:

1. Create audit event.
2. Include context identifiers.
3. Include safe metadata if available.
4. Emit to configured sink.
5. Return PASS.

## Challenge

The current ASYNC guard receives input and context, but not necessarily all previous PipelineResult objects.

Therefore, the first version of HealthcareAuditGuard may only emit a basic event unless the pipeline is enhanced or the demo explicitly passes audit metadata.

## Design Decision For MVP

Do not refactor SecurityPipeline yet.

For the MVP:

- blocking/modifying guards return metadata
- demo prints metadata from PipelineResult
- HealthcareAuditGuard emits a basic async execution event
- richer audit aggregation is deferred

## Audit Event Fields

- audit_event_id
- audit_event_type
- timestamp
- session_id
- task_id
- guard_name
- policy_id
- policy_decision
- phi_detected
- phi_types
- entity_count
- handling_mode
- risk_level

## Raw PHI Safety

Audit event must not include:

- raw input
- raw output
- raw prompt
- raw PHI values

## Sink Behavior

stdout:

- print JSON object to stdout

file:

- append JSON object to JSONL file

## MVP Limitation

The MVP audit system is not a full immutable audit log.

It is an audit-safe event emitter for local demonstration.

---

# 6. External Model Strict Profile

## File

    src/agentguard/medmesh/profiles/external_model_strict.py

## Function

Recommended function:

    build_external_model_strict_pipeline(config: dict | None = None) -> SecurityPipeline

## Purpose

Create a SecurityPipeline configured for strict external model PHI protection.

## Required Guards

Use existing guards where available and import paths allow.

Input guards:

- InputTypeGuard
- InputLengthGuard
- EncodingGuard
- InjectionPatternGuard
- JailbreakPhraseGuard
- PHIScrubberGuard

PRE_LLM guards:

- PHIPromptBoundaryGuard

OUTPUT guards:

- PHILeakageOutputGuard

ASYNC guards:

- HealthcareAuditGuard

## Design Constraint

Do not depend on unverified guard names.

During implementation, inspect actual class names in:

    src/agentguard/security/guards/structural.py
    src/agentguard/security/guards/pattern.py

and import only verified classes.

## Config

Profile config may include:

- phi.sensitivity_threshold
- phi.handling_mode
- audit.sink
- audit.file_path
- input.max_length
- mode

## Return

Return a configured SecurityPipeline.

---

# 7. Demo Design

## File

    examples/medmesh_phi_gateway_demo.py

## Purpose

Demonstrate the PHI Gateway MVP with synthetic data.

## Demo Must Not

- call a real external LLM
- use real patient data
- require API keys
- claim HIPAA compliance

## Demo Flow

1. Create AgentContext.
2. Build external model strict pipeline.
3. Run INPUT stage on synthetic clinical note.
4. Print original synthetic note.
5. Print protected note from PipelineResult.final_input.
6. Run PRE_LLM stage on protected note.
7. Run mock summarizer.
8. Run OUTPUT stage on mock summary.
9. Run ASYNC audit tail.
10. Print governance metadata.

## Mock Summarizer

The mock summarizer should produce a deterministic summary.

Example:

    Summary: The protected patient record describes shortness of breath, elevated blood pressure, lisinopril use, and recommended follow-up.

The mock summary should not reintroduce raw PHI.

## Synthetic Input

Use:

    John Smith is a 67-year-old male admitted to Mercy General Hospital on March 3, 2024. His MRN is 123456. He presented with shortness of breath and elevated blood pressure. He was started on lisinopril and advised to follow up with Dr. Adams.

## Expected Output

Demo should show:

- PHI was detected
- content was modified
- protected text was used for model boundary
- output passed leakage check
- metadata was generated
- audit event was emitted

---

# 8. Testing Design

## Test Directory

    tests/medmesh/

## Test Files

    test_phi_guard.py
    test_boundary_guards.py
    test_audit_guard.py
    test_external_model_strict_profile.py

## Test Categories

### PHIScrubberGuard Tests

Test clean input:

- result is PASS
- phi_detected is false
- entity_count is 0

Test PHI input:

- result is MODIFY
- modified_input is not None
- modified_input does not contain raw synthetic name
- metadata phi_detected is true
- metadata entity_count greater than 0

Test metadata safety:

- metadata does not contain raw PHI values

---

### PHIPromptBoundaryGuard Tests

Test masked prompt:

- result is PASS

Test raw PHI prompt:

- result is BLOCK
- reason_code is raw_phi_external_model_boundary
- audit_required is true

---

### PHILeakageOutputGuard Tests

Test clean output:

- result is PASS

Test PHI output:

- result is BLOCK
- reason_code is phi_leakage_in_model_output
- audit_required is true

---

### HealthcareAuditGuard Tests

Test stdout sink:

- emits valid JSON
- includes session_id and task_id
- excludes raw input

Test file sink:

- appends valid JSON line
- JSON can be parsed

---

### Profile Tests

Test profile creation:

- returns SecurityPipeline

Test stage execution:

- INPUT stage runs
- PRE_LLM stage runs
- OUTPUT stage runs
- ASYNC tail can run

---

# 9. Error Handling Design

## Presidio Missing

If Presidio is not installed:

- setup should fail clearly
- error message should explain required extra

Example:

    Install required dependencies with: pip install -e ".[presidio]"

## Guard Runtime Failure

Existing SecurityGuard fail-closed behavior should return BLOCK.

## Audit Sink Failure

Audit failure should not expose PHI.

For MVP, if audit sink fails:

- return BLOCK or PASS?

Decision:

    return BLOCK in ENFORCE mode

Reason:

In strict external model mode, failed audit for a sensitive event should be treated conservatively.

This may be revisited later.

---

# 10. Configuration Design

## Minimal Config Shape

Initial config may be plain dictionary.

Example:

    {
      "phi": {
        "sensitivity_threshold": 0.7,
        "handling_mode": "mask"
      },
      "audit": {
        "sink": "stdout",
        "file_path": null
      }
    }

## Future Config Shape

Later, this can become:

    medmesh:
      profile: medmesh-external-model-strict
      phi:
        handling_mode: mask
        sensitivity_threshold: 0.7
      audit:
        enabled: true
        sink: otel
      model:
        destination: external
        allow_raw_phi: false

Do not implement full YAML config in MVP unless already easy.

---

# 11. Security Considerations

## Raw PHI

Raw PHI must not be stored in:

- metadata
- audit events
- test snapshots
- logs
- demo output except original synthetic input clearly labeled synthetic

## Synthetic Data

All demos and tests must use synthetic data only.

## Boundary Enforcement

The PRE_LLM boundary check is required even if PHIScrubberGuard ran earlier.

Reason:

Defense in depth.

## Output Inspection

Output inspection is required because models may reintroduce memorized or inferred identifiers.

---

# 12. Known MVP Limitations

The MVP will not provide:

- full HIPAA compliance
- full Safe Harbor certification
- Expert Determination workflow
- complete medical record number detection
- custom clinical NER
- FHIR integration
- immutable audit log
- production SIEM integration
- real external model integration
- semantic clinical safety validation

These are future phases.

---

# 13. Future Extensions

Future versions may add:

- custom Presidio recognizers for medical record numbers
- Safe Harbor category mapping
- date generalization
- age over 89 handling
- reversible pseudonymization
- FHIRScopeGuard
- PatientContextGuard
- ConsentContextGuard
- OpenTelemetry audit sink
- OPA policy integration
- MCP adapter
- LangGraph adapter
- clinical output safety classifier

---

## Design Decisions

### DD-001: Extend Existing Runtime

Decision:

    Use existing SecurityGuard and SecurityPipeline.

Reason:

    Avoid unnecessary runtime rewrite.

---

### DD-002: Use Presidio First

Decision:

    Presidio is the first PHI/PII detection backend.

Reason:

    Existing project already has Presidio optional dependency and PIIScrubberGuard precedent.

---

### DD-003: Mask First

Decision:

    Masking is the first transformation mode.

Reason:

    It is simple, auditable, and enough for MVP.

---

### DD-004: Block Output Leakage

Decision:

    PHILeakageOutputGuard blocks instead of modifies in MVP.

Reason:

    Safer first behavior for external model strict mode.

---

### DD-005: Use Metadata Constants

Decision:

    Use metadata constants instead of ad hoc strings.

Reason:

    Improves audit consistency.

---

### DD-006: Mock External Model

Decision:

    Demo uses mock summarizer, not real LLM.

Reason:

    Prevents accidental PHI exposure and avoids API complexity.

---

## Next Spec

Create:

    specs/phi-gateway/tasks.md

