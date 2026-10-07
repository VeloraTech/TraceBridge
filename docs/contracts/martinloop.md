# MartinLoop Contract

## Purpose

This document defines the contract between **TraceBridge** and **MartinLoop**.

It describes:

* what MartinLoop can provide to TraceBridge
* what TraceBridge can provide to MartinLoop
* which information is authoritative
* how evidence status and uncertainty are preserved
* what TraceBridge must not claim on MartinLoop's behalf

This is a **contract draft**. It defines the boundary between the systems and does not imply that every field or integration described here is currently implemented.

---

## 1. System Boundary

MartinLoop governs an agent run.

TraceBridge does not govern the run. It provides structured evidence about what was observed during the run.

The relationship is:

```text
Agent
  │
  ▼
MartinLoop ───────────────┐
  │                       │
  │ run identity          │
  │ commands              │
  │ verification          │
  │ changed files         │
  │ receipt               │
  ▼                       │
TraceBridge ◀─────────────┘
  │
  │ structured evidence
  ▼
MartinLoop
```

TraceBridge therefore acts as an **evidence boundary**, not as the authority that decides whether a task is complete.

---

## 2. MartinLoop → TraceBridge

MartinLoop may provide run-level context required to associate AgentTrace evidence with a specific governed run.

### Run identity

At minimum, the integration should be able to associate evidence with:

```text
run_id
repository
working_directory
session_id        optional
```

Where available, additional run metadata may include:

```text
agent
start_time
end_time
branch
commit
parent_run
```

TraceBridge must preserve the identity supplied by MartinLoop rather than generating a replacement identity for the same run.

---

## 3. Execution Context

MartinLoop may provide information about how the run was executed.

Examples include:

```text
commands
command results
exit codes
duration
stop reason
budget information
verification command
verification result
changed files
commit information
```

These records describe the **execution and governance context** of the run.

They should remain distinguishable from NativeRelay or AgentTrace observations.

For example:

```text
MartinLoop:
    verification command exited 0

AgentTrace:
    process X executed the command

NativeRelay:
    process X opened file Y
```

These are related pieces of evidence, but they are not interchangeable.

---

## 4. Verification Result

MartinLoop may provide the result of its configured verification step.

A verification record should preserve at least:

```text
verification:
    status
    command
    exit_code
    started_at
    completed_at
```

Possible status values may include:

```text
passed
failed
not_run
unknown
```

A passing verifier means that the configured verification command passed.

It does **not** automatically establish that:

* the agent performed every intended action
* every relevant file access was observed
* the implementation is correct in every respect
* no unobserved activity occurred

MartinLoop's own documentation makes this distinction: verification is evidence that the configured checks passed, not a general proof of correctness or safety.

TraceBridge must preserve that distinction.

---

## 5. Changed Files

MartinLoop may provide the files it considers changed during the run.

Example:

```text
changed_files:
    - path: src/auth.js
      status: modified

    - path: tests/auth.test.js
      status: modified
```

Changed-file information should be treated as **run-level execution evidence**.

It should not automatically be interpreted as proof that:

```text
agent X modified the file
```

unless AgentTrace has corresponding attribution evidence.

The distinction is:

```text
MartinLoop:
    file changed during the run

AgentTrace:
    process X was observed modifying the file

TraceBridge:
    preserves both facts and their provenance
```

---

## 6. Process and Workspace Evidence

AgentTrace may provide lower-level observations associated with the MartinLoop run.

Examples:

```text
process.started
process.exited

file.created
file.modified
file.deleted
file.renamed

process-attributed file activity
```

Each observation should retain its provenance and observation status.

TraceBridge must not convert:

```text
file.modified
```

into:

```text
agent modified file
```

unless the underlying evidence supports that attribution.

---

## 7. Attribution

AgentTrace may associate activity with an agent or session.

Attribution should retain its confidence state:

```text
direct
inferred
unknown
```

### Direct

The observation has a direct supported relationship to the agent/session.

### Inferred

The relationship is derived from available process, session, or execution context but is not directly observed.

### Unknown

The system cannot establish the actor with sufficient evidence.

TraceBridge must preserve this state.

It must not turn:

```text
inferred
```

into:

```text
direct
```

simply because the resulting evidence is more convenient for verification.

---

## 8. Observation Status

Evidence may also have an observation status independent of attribution.

Recommended states include:

```text
observed
unknown
unsupported
permission_denied
degraded
lost
```

For example:

```text
event:
    file.opened

attribution:
    unknown

observation:
    observed
```

means:

> The file-open event was observed, but the actor could not be established.

Whereas:

```text
event:
    file.opened

observation:
    permission_denied
```

means:

> The collector could not obtain the relevant observation because the required OS-level access was unavailable.

These states must not be collapsed.

---

## 9. Loss and Coverage

If AgentTrace reports missing observations, TraceBridge must preserve that information.

Examples:

```text
events_lost
sequence_gap
collector_failure
permission_denied
unsupported_capability
degraded_collection
```

A missing event must not be interpreted as evidence that an action did not happen.

The correct interpretation is:

```text
not observed under the available coverage
```

rather than:

```text
did not happen
```

This distinction is fundamental to the evidence model.

---

## 10. Evidence From MartinLoop

MartinLoop's run receipt may contain information such as:

```text
commands run
exit codes
changed files
commit hashes
budget usage
verifier result
stop reason
evidence artifact reference
```

MartinLoop describes these records as part of its inspectable run history.

TraceBridge may consume this information as **execution-context evidence**.

It must preserve the original provenance.

---

## 11. Evidence Returned to MartinLoop

TraceBridge should provide structured evidence that MartinLoop can consume during or after verification.

The evidence may include:

```text
run identity
observed process activity
workspace activity
process-attributed activity
agent/session attribution
collector capabilities
observation status
loss information
integrity status
artifact references
execution relationships
```

A simplified representation:

```text
evidence:
    run_id
    observations[]
    attribution[]
    coverage
    integrity
    artifacts[]
    execution_context
```

The exact schema will be defined separately in the TraceBridge evidence-model contract.

---

## 12. Evidence Status

TraceBridge should expose an overall evidence state without pretending that one status represents every individual observation.

For example:

```text
complete
partial
degraded
incomplete
lost
unknown
```

This status describes the **quality/completeness of the evidence available to TraceBridge**.

It must not replace individual observation states.

For example:

```text
overall:
    partial

observations:
    file.modified → observed
    file.opened   → observed
    file.read     → unknown
```

The individual records remain authoritative for what was and was not observed.

---

## 13. Integrity

Where AgentTrace provides authoritative-history or integrity information, TraceBridge should preserve a reference to it.

Possible fields include:

```text
integrity:
    status
    sequence_valid
    loss_detected
    history_reference
```

TraceBridge should not independently claim that an event history is trustworthy merely because it received valid JSON.

Integrity is a property that must come from the underlying evidence and its verification process.

---

## 14. Verification vs Observation

The systems answer different questions.

### MartinLoop

```text
Did the configured verification command pass?
```

### AgentTrace

```text
What agent/session/process activity was observed?
```

### NativeRelay

```text
What native OS activity was observed?
```

### TraceBridge

```text
How can those observations be represented as structured evidence
without overstating what they prove?
```

These questions may reinforce one another, but they must remain separate.

---

## 15. Completion

TraceBridge must not independently declare a MartinLoop run complete.

A possible evidence record can say:

```text
verification:
    passed

agent_activity:
    observed

workspace_activity:
    observed

evidence_integrity:
    valid
```

That is evidence available to MartinLoop.

MartinLoop remains responsible for applying its own completion and verification rules.

---

## 16. Evidence Artifact

MartinLoop may reference an evidence artifact associated with a run.

TraceBridge may produce or reference an artifact containing its structured evidence.

Example:

```text
.martinloop/
    evidence/
        <run-id>.json
```

The exact storage location and ownership should be defined by the integration rather than assumed by TraceBridge.

The important requirement is that the artifact retains:

```text
run identity
evidence provenance
observation status
attribution state
integrity information
loss information
```

---

## 17. Failure Conditions

TraceBridge should explicitly represent conditions that weaken the evidence.

Examples:

```text
collector unavailable
permission denied
unsupported platform capability
event loss
sequence gap
recorder integrity failure
AgentTrace terminated unexpectedly
MartinLoop run identity unavailable
evidence artifact unavailable
```

TraceBridge should not silently convert these conditions into an apparently complete evidence record.

---

## 18. Contract Principle

The central rule of the MartinLoop integration is:

> **TraceBridge may organize evidence, but it must not make the evidence stronger than the observation supports.**

MartinLoop decides what constitutes a verified or completed run.

AgentTrace supplies observations and correlations.

NativeRelay supplies native OS observations.

TraceBridge preserves the relationship between those facts.

---

## 19. Example

Consider an agent working on:

```text
src/auth.js
```

MartinLoop reports:

```text
verification:
    command: npm test -- auth
    exit_code: 0
    status: passed
```

AgentTrace reports:

```text
process:
    agent_session: session-42
    process: 1842

workspace:
    src/auth.js → modified
```

NativeRelay reports:

```text
process 1842
    opened src/auth.js
    modified src/auth.js
```

TraceBridge can represent:

```text
run:
    run_id: run-123

verification:
    passed

observations:
    process 1842 → opened src/auth.js
    process 1842 → modified src/auth.js

attribution:
    process 1842 → session-42
    confidence: direct

integrity:
    valid
```

But if the process relationship was only inferred, the evidence must instead preserve:

```text
attribution:
    process 1842 → session-42
    confidence: inferred
```

And if the filesystem collector could not observe reads:

```text
read_activity:
    status: unknown
```

TraceBridge must not turn that into:

```text
read_activity:
    none
```

---

## 20. Responsibility Summary

| Responsibility                    | System      |
| --------------------------------- | ----------- |
| Native OS observation             | NativeRelay |
| Agent/process/session correlation | AgentTrace  |
| Authoritative event history       | AgentTrace  |
| Evidence normalization            | TraceBridge |
| Evidence provenance               | TraceBridge |
| Evidence transport                | TraceBridge |
| Run governance                    | MartinLoop  |
| Verification command              | MartinLoop  |
| Completion decision               | MartinLoop  |
| Final task interpretation         | MartinLoop  |

The boundary exists so that each system can remain narrow and independently verifiable.
