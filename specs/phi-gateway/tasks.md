# Spec: MedMesh PHI Gateway Tasks

## Status

Draft

## Related Specs

- specs/phi-gateway/requirements.md
- specs/phi-gateway/design.md

## Feature

MedMesh PHI Gateway

## Target Profile

medmesh-external-model-strict

---

## Objective

Break the PHI Gateway MVP into concrete engineering tasks.

This task list should be completed on a feature branch after the spec files are committed.

Recommended branch:

    feature/medmesh-phi-gateway

---

## Pre-Implementation Checklist

Before implementation begins:

- requirements.md exists
- design.md exists
- tasks.md exists
- architecture docs are committed
- implementation plan is committed
- current branch is clean
- new feature branch is created

Check current status:

    git status

Create the feature branch only after this tasks file is committed:

    git checkout -b feature/medmesh-phi-gateway

---

# Task Group 1: Create MedMesh Extension Structure

## Task 1.1: Create MedMesh Package Directories

Create:

    src/agentguard/medmesh/
    src/agentguard/medmesh/guards/
    src/agentguard/medmesh/profiles/
    src/agentguard/medmesh/metadata/

Command:

    mkdir -p src/agentguard/medmesh/{guards,profiles,metadata}

Acceptance:

- directories exist
- no existing runtime package is renamed

---

## Task 1.2: Create Package Init Files

Create:

    src/agentguard/medmesh/__init__.py
    src/agentguard/medmesh/guards/__init__.py
    src/agentguard/medmesh/profiles/__init__.py
    src/agentguard/medmesh/metadata/__init__.py

Command:

    touch src/agentguard/medmesh/__init__.py
    touch src/agentguard/medmesh/guards/__init__.py
    touch src/agentguard/medmesh/profiles/__init__.py
    touch src/agentguard/medmesh/metadata/__init__.py

Acceptance:

- package imports are possible
- no runtime behavior changes yet

---

# Task Group 2: Add Metadata Constants

## Task 2.1: Create Metadata Keys Module

Create:

    src/agentguard/medmesh/metadata/keys.py

Acceptance:

- module exists
- constants align with governance metadata contract

---

## Task 2.2: Add Core Metadata Constants

Add constants for:

    GUARD_CATEGORY
    ENFORCEMENT_MODE
    POLICY_ID
    POLICY_VERSION
    POLICY_DECISION
    AUDIT_REQUIRED
    RISK_LEVEL
    REASON_CODE
    HUMAN_READABLE_REASON

Acceptance:

- constants are strings
- names are uppercase
- values are snake_case

---

## Task 2.3: Add PHI Metadata Constants

Add constants for:

    PHI_DETECTED
    PHI_TYPES
    PII_TYPES
    ENTITY_COUNT
    ENTITY_SUMMARY
    CONFIDENCE_THRESHOLD
    DETECTOR_BACKEND

Acceptance:

- keys match docs/architecture/governance-metadata-contracts.md

---

## Task 2.4: Add Transformation Metadata Constants

Add constants for:

    HANDLING_MODE
    ANONYMIZATION_OPERATOR
    REVERSIBLE
    SAFE_HARBOR_APPLIED
    SAFE_HARBOR_CATEGORIES

Acceptance:

- keys are reusable by PHI guards

---

## Task 2.5: Add Audit Metadata Constants

Add constants for:

    AUDIT_EVENT_ID
    AUDIT_EVENT_TYPE
    SESSION_ID
    TASK_ID
    TIMESTAMP
    GUARD_NAME
    VERDICT

Acceptance:

- keys are reusable by audit guard

---

# Task Group 3: Implement PHI Detection Utilities

## Task 3.1: Create PHI Guard Module

Create:

    src/agentguard/medmesh/guards/phi.py

Acceptance:

- module imports successfully
- no guard implemented yet

---

## Task 3.2: Add Presidio Setup Helper

Implement helper logic to initialize:

    AnalyzerEngine
    AnonymizerEngine

Acceptance:

- imports Presidio lazily
- raises clear configuration error if missing
- does not silently disable PHI protection

---

## Task 3.3: Add Entity Summary Helper

Implement helper to convert analyzer results into safe metadata:

    {
      "PERSON": 2,
      "DATE_TIME": 1
    }

Acceptance:

- no raw entity values are included
- counts are grouped by entity type

---

## Task 3.4: Add Safe Metadata Helper

Implement helper that builds metadata for:

- no PHI detected
- PHI detected and masked

Acceptance:

- metadata uses constants from keys.py
- metadata contains no raw PHI

---

# Task Group 4: Implement PHIScrubberGuard

## Task 4.1: Define PHIScrubberGuard Class

Class:

    PHIScrubberGuard

Base:

    SecurityGuard

File:

    src/agentguard/medmesh/guards/phi.py

Acceptance:

- class extends SecurityGuard
- category is 5
- default intercept points are INPUT and TOOL_OUTPUT
- default timeout is reasonable

---

## Task 4.2: Add Constructor Configuration

Constructor should accept:

- sensitivity_threshold
- language
- handling_mode
- policy_id
- mode
- timeout_ms

Defaults:

    sensitivity_threshold = 0.7
    language = "en"
    handling_mode = "mask"
    policy_id = "phi.mask.v1"

Acceptance:

- defaults work
- custom config works

---

## Task 4.3: Implement Setup

Setup should initialize Presidio engines.

Acceptance:

- setup succeeds when Presidio is installed
- setup fails clearly when Presidio is missing

---

## Task 4.4: Implement Empty Input Behavior

If input is empty or whitespace:

- return PASS

Acceptance:

- no exception
- no modification

---

## Task 4.5: Implement No PHI Behavior

If no entities are detected:

- return PASS
- metadata includes phi_detected = false
- entity_count = 0
- policy_decision = allow

Acceptance:

- clean clinical text passes
- metadata is safe

---

## Task 4.6: Implement PHI Masking Behavior

If entities are detected:

- anonymize or mask content
- return MODIFY
- modified_input contains protected text
- metadata includes phi_detected = true

Acceptance:

- synthetic name is removed or replaced
- synthetic date is removed or replaced
- metadata contains entity types and counts
- metadata contains no raw PHI

---

## Task 4.7: Validate Raw PHI Is Not In Metadata

Add explicit test helper or assertion to ensure metadata does not contain known synthetic PHI values.

Acceptance:

- metadata does not include John Smith
- metadata does not include MRN 123456
- metadata does not include raw note text

---

# Task Group 5: Implement Boundary Guards

## Task 5.1: Create Boundary Guard Module

Create:

    src/agentguard/medmesh/guards/boundary.py

Acceptance:

- module imports successfully

---

## Task 5.2: Implement PHIPromptBoundaryGuard

Class:

    PHIPromptBoundaryGuard

Base:

    SecurityGuard

Intercept point:

    PRE_LLM

Behavior:

- PASS if no PHI detected
- BLOCK if PHI detected

Acceptance:

- raw synthetic PHI blocks
- masked prompt passes
- metadata includes policy decision
- metadata includes reason_code on block
- metadata excludes raw PHI

---

## Task 5.3: Implement PHILeakageOutputGuard

Class:

    PHILeakageOutputGuard

Base:

    SecurityGuard

Intercept point:

    OUTPUT

Behavior:

- PASS if no PHI detected
- BLOCK if PHI detected

Acceptance:

- clean output passes
- output with synthetic name blocks
- metadata includes leakage reason
- metadata excludes raw PHI

---

## Task 5.4: Share Detection Logic Safely

Boundary guards may reuse helper logic from phi.py.

Acceptance:

- no duplicated unsafe metadata logic
- no raw PHI stored

---

# Task Group 6: Implement HealthcareAuditGuard

## Task 6.1: Create Audit Guard Module

Create:

    src/agentguard/medmesh/guards/audit.py

Acceptance:

- module imports successfully

---

## Task 6.2: Implement HealthcareAuditGuard

Class:

    HealthcareAuditGuard

Base:

    SecurityGuard

Intercept point:

    ASYNC

Category:

    7

Acceptance:

- guard returns PASS after emitting event
- guard does not include raw input in audit event

---

## Task 6.3: Implement Audit Event Builder

Audit event should include:

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

Acceptance:

- JSON serializable
- no raw PHI fields

---

## Task 6.4: Implement stdout Sink

Behavior:

- print audit event as JSON

Acceptance:

- output can be parsed as JSON
- event excludes raw PHI

---

## Task 6.5: Implement Optional File Sink

Behavior:

- append audit event to JSONL file

Acceptance:

- file is created when configured
- one JSON object per line
- event excludes raw PHI

---

# Task Group 7: Implement External Model Strict Profile

## Task 7.1: Create Profile Module

Create:

    src/agentguard/medmesh/profiles/external_model_strict.py

Acceptance:

- module imports successfully

---

## Task 7.2: Verify Existing Guard Class Names

Before importing existing guards, inspect actual class names in:

    src/agentguard/security/guards/structural.py
    src/agentguard/security/guards/pattern.py
    src/agentguard/security/guards/budget.py
    src/agentguard/security/guards/scope.py

Acceptance:

- only verified class names are imported
- no guessed imports

---

## Task 7.3: Implement Profile Factory

Function:

    build_external_model_strict_pipeline(config: dict | None = None) -> SecurityPipeline

Acceptance:

- function returns SecurityPipeline
- includes MedMesh PHI guards
- includes existing structural and pattern guards where importable

---

## Task 7.4: Configure Intercept Points

Ensure guards are attached to correct intercept points:

INPUT:

- structural guards
- pattern guards
- PHIScrubberGuard

PRE_LLM:

- PHIPromptBoundaryGuard

OUTPUT:

- PHILeakageOutputGuard

ASYNC:

- HealthcareAuditGuard

Acceptance:

- INPUT pipeline modifies synthetic PHI
- PRE_LLM pipeline blocks raw PHI
- OUTPUT pipeline blocks PHI leakage
- ASYNC tail can run

---

# Task Group 8: Create Demo

## Task 8.1: Create Demo File

Create:

    examples/medmesh_phi_gateway_demo.py

Acceptance:

- file exists
- imports profile factory

---

## Task 8.2: Add Synthetic Clinical Note

Use synthetic text only.

Acceptance:

- demo text clearly marked synthetic
- no real patient data

---

## Task 8.3: Add Mock Summarizer

Implement deterministic mock summarizer.

Acceptance:

- no external API call
- no API keys
- no network dependency

---

## Task 8.4: Run Full Pipeline Demo

Demo should run:

1. INPUT
2. PRE_LLM
3. mock summarizer
4. OUTPUT
5. ASYNC audit

Acceptance:

- demo prints protected note
- demo prints metadata
- demo prints final safe summary
- demo emits audit event

---

# Task Group 9: Add Tests

## Task 9.1: Create Test Directory

Create:

    tests/medmesh/

Command:

    mkdir -p tests/medmesh

Acceptance:

- directory exists

---

## Task 9.2: Test PHIScrubberGuard

Create:

    tests/medmesh/test_phi_guard.py

Test:

- clean input passes
- synthetic PHI modifies
- metadata includes phi_detected
- metadata excludes raw PHI
- empty input passes

---

## Task 9.3: Test Boundary Guards

Create:

    tests/medmesh/test_boundary_guards.py

Test:

- PHIPromptBoundaryGuard blocks raw PHI
- PHIPromptBoundaryGuard passes masked text
- PHILeakageOutputGuard blocks leaked PHI
- PHILeakageOutputGuard passes clean output

---

## Task 9.4: Test Audit Guard

Create:

    tests/medmesh/test_audit_guard.py

Test:

- stdout event is JSON
- file event is JSONL
- event excludes raw PHI
- event includes session_id and task_id

---

## Task 9.5: Test Profile Factory

Create:

    tests/medmesh/test_external_model_strict_profile.py

Test:

- profile builds SecurityPipeline
- INPUT stage runs
- PRE_LLM stage runs
- OUTPUT stage runs
- ASYNC tail runs

---

# Task Group 10: Documentation Updates

## Task 10.1: Update README

Add MedMesh MVP section.

Include:

- what MedMesh is
- what PHI Gateway does
- safety disclaimer
- demo command

---

## Task 10.2: Create Demo Documentation

Create:

    docs/examples/phi-safe-clinical-summarization.md

Include:

- demo purpose
- how to run
- expected output
- safety assumptions
- synthetic data notice

---

# Task Group 11: Validation

## Task 11.1: Run Tests

Command:

    pytest tests/medmesh

Acceptance:

- all MedMesh tests pass

---

## Task 11.2: Run Demo

Command:

    python examples/medmesh_phi_gateway_demo.py

Acceptance:

- demo runs
- no external API call
- PHI is masked
- metadata is printed
- audit event is emitted

---

## Task 11.3: Check Git Diff

Command:

    git diff

Acceptance:

- no unrelated changes
- no raw PHI in committed files except clearly synthetic examples

---

# Branching Point

After this tasks spec is committed, create the implementation branch:

    git checkout -b feature/medmesh-phi-gateway

Do not start implementation until on this branch.

---

# Definition Of Done

The implementation is complete when:

- all required files exist
- PHIScrubberGuard works
- boundary guards work
- audit guard emits safe events
- profile factory works
- demo runs
- tests pass
- docs are updated
- no raw PHI is stored in metadata or audit outputs
- README clearly states that the MVP is not for clinical use

