# AgentTrace Contract

This document defines the evidence TraceBridge expects to receive from AgentTrace.

The contract describes the information TraceBridge needs to identify an agent run, understand its execution evidence, preserve attribution and uncertainty, and associate observations with the repository and task being verified.

This is a **contract definition**, not a statement that every field is currently implemented by AgentTrace.

## 1. Run Identity

Every evidence package should identify the AgentTrace run that produced it.

```text
run_id
session_id
repository
```

### `run_id`

A unique identifier for the AgentTrace execution being reported.

### `session_id`

Identifies the agent session associated with the execution.

A run may contain one or more related sessions depending on how the agent operates.

### `repository`

Identifies the repository in which the run took place.

Where available, this should include enough information to distinguish the repository and revision involved in the run.

---

## 2. Execution Evidence

AgentTrace may provide evidence describing what occurred during the run.

Relevant evidence can include:

```text
process activity
workspace activity
process-attributed activity
agent/session relationships
execution results
```

TraceBridge should consume these as **observations**, not automatically treat every observation as proof of task completion.

---

## 3. Workspace Activity

Workspace activity describes changes observed in the repository or working directory.

Examples include:

```text
file.created
file.modified
file.deleted
file.renamed
```

A workspace event should identify the affected resource and provide the strongest available evidence about when and how the change was observed.

Workspace activity does not, by itself, establish which agent or process caused the change.

---

## 4. Process-Attributed Activity

Where the underlying monitoring system provides process attribution, AgentTrace may associate an observation with a process.

Relevant information can include:

```text
process_id
process_start_time
parent_process
process_executable
event_time
resource
operation
```

PID alone should not be treated as a permanent process identity because operating systems may reuse process IDs.

The exact native fields available will depend on the NativeRelay backend.

---

## 5. Attribution

AgentTrace should preserve how confidently an observation can be connected to an agent session.

The initial attribution model is:

```text
direct
inferred
unknown
```

### `direct`

The available evidence directly establishes the relationship.

### `inferred`

The relationship is derived from available process/session information but is not directly established.

### `unknown`

The available evidence is insufficient to establish the relationship.

TraceBridge must preserve this distinction.

It must not convert an `inferred` or `unknown` relationship into a `direct` claim.

---

## 6. Observation Status

Evidence should also communicate the state of the observation.

Possible states include:

```text
observed
unknown
unsupported
permission_denied
degraded
lost
```

These states describe the evidence available to AgentTrace.

In particular:

> **No observation does not automatically mean that the action did not occur.**

A collector that cannot observe a particular operation must not produce evidence claiming that the operation never happened.

---

## 7. Native Observation Metadata

Where evidence originated from NativeRelay, AgentTrace should preserve the relevant provenance.

This may include:

```text
platform
collector
event_type
timestamp
sequence
event_id
capabilities
loss information
```

The purpose is to allow TraceBridge and downstream systems to understand **where the evidence came from and under what observation conditions it was collected**.

---

## 8. Integrity

AgentTrace should provide information about the integrity of the evidence where available.

Relevant information may include:

```text
event identity
sequence information
loss records
integrity status
authoritative history reference
```

TraceBridge should preserve integrity information rather than attempting to reconstruct an authoritative history from an exported representation.

If the evidence is incomplete or integrity cannot be established, that condition should remain visible to MartinLoop.

---

## 9. Execution Result

AgentTrace may provide the outcome of the observed execution.

Examples include:

```text
status
exit_code
error
duration
completion_state
```

An execution result describes what happened to the run itself.

It does not independently establish that the task requirements were satisfied.

MartinLoop remains responsible for task-level verification.

---

## 10. Artifacts

AgentTrace may provide references to artifacts produced or observed during the run.

Examples include:

```text
changed files
test results
command results
generated reports
logs
trace exports
```

Artifacts should retain their relationship to the originating run and, where possible, their integrity information.

---

## 11. Evidence Provenance

Every piece of evidence should be traceable to its source.

At minimum, TraceBridge should be able to determine whether evidence originated from:

```text
NativeRelay
AgentTrace runtime
AgentTrace recorder
execution result
external artifact
```

This prevents different types of evidence from being treated as equivalent when they are not.

---

## 12. Contract Principle

The AgentTrace → TraceBridge boundary follows one rule:

> **TraceBridge may structure and transport evidence, but it must not make the evidence stronger than the observation supports.**

AgentTrace is responsible for collecting and correlating execution evidence.

TraceBridge is responsible for preserving, validating, and transforming that evidence for MartinLoop.

MartinLoop remains responsible for deciding whether the evidence satisfies the requirements of a particular task.
