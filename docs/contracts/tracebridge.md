# TraceBridge Contract

## Purpose

This document defines the contract for **TraceBridge**.

TraceBridge is the evidence boundary between AgentTrace and MartinLoop.

Its responsibility is to take execution observations and run context, validate and normalize them, preserve their provenance and uncertainty, associate them with the correct run, and expose structured evidence to MartinLoop.

TraceBridge does **not** perform native OS monitoring, determine whether an agent is correct, or independently declare a run successful.

---

## 1. System Boundary

The intended flow is:

```text
NativeRelay
    │
    ▼
AgentTrace
    │
    │ authoritative observations
    ▼
TraceBridge
    │
    │ structured evidence
    ▼
MartinLoop
```

TraceBridge sits between **observation** and **verification**.

Its job is to make evidence usable without changing what that evidence means.

---

## 2. TraceBridge Responsibilities

TraceBridge is responsible for:

* accepting AgentTrace evidence
* validating the incoming contract
* normalizing compatible evidence formats
* preserving provenance
* preserving attribution state
* preserving observation status
* preserving loss and coverage information
* associating evidence with a MartinLoop run
* producing structured evidence for MartinLoop
* exposing evidence-integrity information
* rejecting or flagging malformed or unsupported evidence

TraceBridge is not responsible for:

* collecting kernel/OS events
* monitoring filesystems directly
* identifying AI agents independently
* deciding whether an implementation is correct
* running tests on behalf of MartinLoop
* replacing MartinLoop's verifier
* converting unknown observations into negative claims

---

## 3. Input Boundary

The primary input is an AgentTrace evidence record or evidence bundle.

A bundle may contain:

```text
run identity
session identity
process observations
workspace observations
process-attributed observations
agent/session relationships
collector information
loss information
integrity information
execution results
artifact references
```

TraceBridge should accept evidence from the AgentTrace contract rather than consuming raw NativeRelay events directly.

This keeps the architecture:

```text
NativeRelay → AgentTrace → TraceBridge
```

rather than:

```text
NativeRelay → TraceBridge
```

AgentTrace remains responsible for correlation between native observations and agent/session activity.

---

## 4. Run Association

Every evidence bundle should be associated with a run.

Required identity:

```text
run_id
```

Where available:

```text
session_id
repository
working_directory
branch
commit
start_time
end_time
```

TraceBridge must not silently merge evidence from different runs.

If run identity is missing or ambiguous, the evidence should be marked accordingly rather than assigned to an arbitrary run.

---

## 5. Evidence Record

A normalized TraceBridge evidence record should contain, conceptually:

```text
evidence:
    id
    run_id
    timestamp
    source
    category
    observation
    attribution
    status
    provenance
    integrity
    metadata
```

The exact serialized schema will be defined in:

```text
docs/evidence/evidence-model.md
```

This contract defines the behavior of that schema, not every field in its final implementation.

---

## 6. Evidence Categories

TraceBridge should preserve the distinction between major evidence categories.

### Process evidence

Examples:

```text
process.started
process.exited
process.created_child
```

### Workspace evidence

Examples:

```text
file.created
file.modified
file.deleted
file.renamed
```

Workspace evidence does not inherently establish which process or agent performed the action.

### Process-attributed workspace evidence

Examples:

```text
process 1842 opened file X
process 1842 modified file Y
```

This is stronger than an un-attributed workspace change because a process relationship exists.

It still requires separate attribution to an AI agent/session.

### Execution evidence

Examples:

```text
command.started
command.completed
command.failed
```

### Verification evidence

Examples:

```text
verification.started
verification.passed
verification.failed
```

Verification evidence originates from the execution/governance layer and must remain distinguishable from low-level observations.

---

## 7. Attribution Preservation

TraceBridge must preserve attribution exactly as supplied.

Supported states:

```text
direct
inferred
unknown
```

For example:

```text
observation:
    file.modified

process:
    pid: 1842

agent:
    session_id: session-42

attribution:
    inferred
```

TraceBridge must not simplify this to:

```text
agent_modified_file: true
```

unless the source contract explicitly establishes direct attribution.

---

## 8. Observation Status

Every observation should be interpretable within its collection context.

Possible statuses include:

```text
observed
unknown
unsupported
permission_denied
degraded
lost
```

Example:

```text
file.read
status: unknown
```

means that TraceBridge does not have evidence establishing whether the read occurred.

It does not mean:

```text
file.read
status: not_occurred
```

TraceBridge must not create negative evidence from absence of observation.

---

## 9. Validation

TraceBridge validates incoming evidence before exposing it to MartinLoop.

Validation should cover:

### Identity

```text
run_id
event/evidence ID
source
timestamp
```

### Structure

Required fields must have valid types and expected structure.

### Provenance

Every observation should identify its source.

### Attribution

Attribution values must use the defined vocabulary.

### Status

Observation and integrity statuses must use supported values.

### Integrity

If sequence or chain information is supplied, TraceBridge should verify the declared relationships where possible.

### Loss

Loss records must not be silently discarded.

Invalid evidence should be rejected or explicitly marked invalid.

---

## 10. Normalization

TraceBridge may normalize equivalent representations.

For example, two sources may represent a modified file as:

```text
file.modified
```

and:

```text
workspace.modify
```

TraceBridge may normalize both into a canonical evidence type if their semantics are equivalent.

Normalization must not increase the certainty of an observation.

For example:

```text
unknown → observed
```

is not valid normalization.

Likewise:

```text
inferred → direct
```

is not valid normalization.

---

## 11. Provenance

Every evidence item must retain its origin.

Possible provenance sources:

```text
NativeRelay
AgentTrace
AgentTrace recorder
MartinLoop
execution result
external artifact
```

Example:

```text
provenance:
    source: NativeRelay
    collector: linux.fanotify
```

or:

```text
provenance:
    source: AgentTrace
    component: session-correlator
```

TraceBridge may add provenance about its own transformation:

```text
transformed_by:
    TraceBridge
```

but must not replace the original source.

---

## 12. Integrity Boundary

TraceBridge consumes evidence from the AgentTrace authoritative history.

The intended chain is:

```text
Native observation
      │
      ▼
AgentTrace recorder
      │
      ▼
Integrity verification
      │
      ▼
TraceBridge
      │
      ▼
MartinLoop
```

TraceBridge should retain enough information for MartinLoop to determine whether the evidence has:

```text
valid integrity
invalid integrity
unknown integrity
loss detected
```

A valid JSON document is not itself proof of an untampered history.

---

## 13. Loss Handling

If AgentTrace reports:

```text
sequence_gap
event_loss
collector_loss
recorder_failure
```

TraceBridge must preserve that information.

Example:

```text
coverage:
    status: incomplete

loss:
    detected: true
    source: NativeRelay
```

TraceBridge must not silently discard the affected events and return an apparently complete evidence bundle.

---

## 14. Coverage

Evidence should describe the collection coverage under which it was produced.

Example:

```text
coverage:
    platform: linux
    collectors:
        process: available
        filesystem: available

    limitations:
        - mmap_access_not_observed
```

Coverage information is important because:

```text
no event observed
```

and:

```text
event impossible under this collector
```

are different situations.

---

## 15. Transformation Rules

TraceBridge may:

```text
validate
normalize
group
index
associate
serialize
transport
```

TraceBridge must not:

```text
invent observations
upgrade attribution
upgrade confidence
hide loss
convert unknown into negative evidence
claim unsupported coverage
declare task completion
```

This is the core behavioral boundary of the project.

---

## 16. Evidence Grouping

TraceBridge may group related observations.

Example:

```text
process.started
      │
      ├── file.opened
      ├── file.modified
      └── process.exited
```

This can make evidence easier for MartinLoop to consume.

However, grouping must not imply causal relationships that were not established.

A visual or structural relationship such as:

```text
process → file.modified
```

must not automatically become:

```text
process caused file.modified
```

unless the underlying observation supports that conclusion.

---

## 17. Artifact References

TraceBridge may reference artifacts rather than embedding them.

Examples:

```text
AgentTrace history
JSONL export
verification receipt
test output
diff
report
integrity manifest
```

An artifact reference should identify:

```text
artifact_id
type
location/reference
source
integrity information
```

Sensitive content should not be copied into evidence unnecessarily.

---

## 18. Privacy Boundary

Evidence may contain sensitive information.

Examples include:

```text
file paths
command arguments
repository information
process names
environment-specific metadata
```

TraceBridge should therefore distinguish:

```text
evidence identity
evidence metadata
evidence payload
```

The bridge should not automatically expose full sensitive payloads simply because they were observable.

Redaction or minimization should be explicit and must not falsely imply that the underlying observation did not contain the removed information.

---

## 19. Output to MartinLoop

TraceBridge produces structured evidence that MartinLoop can consume.

Conceptually:

```text
TraceBridge output:

run:
    run_id

evidence:
    observations[]
    attribution[]
    coverage
    integrity
    losses[]
    artifacts[]

verification_context:
    execution_results[]
```

MartinLoop can then combine this with its own run and verification state.

TraceBridge does not decide how MartinLoop weighs the evidence.

---

## 20. Failure Behavior

TraceBridge should fail explicitly when it cannot safely preserve the evidence contract.

Examples:

```text
invalid evidence schema
missing run identity
unsupported evidence version
corrupt integrity information
ambiguous provenance
unrecoverable serialization failure
```

Where evidence can still be safely represented, TraceBridge should prefer:

```text
status: degraded
```

or:

```text
status: incomplete
```

over silently dropping information.

---

## 21. Versioning

The evidence contract must be versioned.

TraceBridge should identify:

```text
schema_version
contract_version
producer
```

An incompatible contract change should not be silently interpreted as an older format.

Compatibility behavior should be explicit:

```text
supported
deprecated
unsupported
```

This allows AgentTrace and MartinLoop to evolve independently.

---

## 22. Example End-to-End Record

Input from AgentTrace:

```text
run_id: run-123

observation:
    type: file.modified
    path: src/auth.js

process:
    pid: 1842

attribution:
    state: direct
    session_id: session-42

status:
    observed

provenance:
    source: NativeRelay
    collector: linux.fanotify

integrity:
    status: valid
```

TraceBridge validates and normalizes the record.

Its output may become:

```text
run:
    run_id: run-123

evidence:
    id: event-981
    category: process_attributed_workspace
    operation: modified
    resource:
        type: file
        path: src/auth.js

    attribution:
        state: direct
        session_id: session-42
        process_id: 1842

    status:
        observed

    provenance:
        source: NativeRelay
        collector: linux.fanotify

    integrity:
        status: valid
```

The important fact is that TraceBridge has **structured the evidence**, not strengthened it.

---

## 23. Contract Principle

TraceBridge follows one governing rule:

> **Transform evidence for interoperability, never for certainty.**

AgentTrace observes and correlates.

TraceBridge structures and transports.

MartinLoop verifies and governs.

No layer should silently assume the responsibility of another.

---

## 24. Current Status

This contract defines the intended TraceBridge boundary.

It does not yet define:

* the complete evidence schema
* storage format
* transport protocol
* authentication
* cryptographic signing format
* API endpoints
* CLI commands
* MartinLoop implementation details

Those belong to subsequent design documents.
