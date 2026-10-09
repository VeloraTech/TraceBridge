# TraceBridge

**A structured evidence bridge between AgentTrace and MartinLoop.**

TraceBridge connects observations about AI coding-agent activity to the governed execution record maintained by MartinLoop. It validates, correlates, and packages evidence so that execution activity can be inspected alongside a run's objective, constraints, and verification results.

The goal is straightforward: **make execution evidence useful without overstating what it proves.**

> **Status:** Early development. The data contract and integration boundaries are being defined. Proposed capabilities are not necessarily implemented.

## Why TraceBridge?

Knowing that a coding agent completed a task is different from knowing what happened while it worked.

Execution observations may come from different sources, use different identifiers, have incomplete coverage, or disagree about which run they belong to. Passing these records directly between systems risks losing provenance or treating an inference as an established fact.

TraceBridge addresses this integration problem by providing a defined boundary between observation and governance.

It is designed to preserve:

- **Identity** — which run, repository, agent, session, and process the evidence relates to.
- **Execution history** — observed process activity, commands, filesystem operations, timestamps, and outcomes where available.
- **Provenance** — where each record originated and how it was transformed.
- **Integrity** — what checks were performed and what those checks established.
- **Coverage** — known observation gaps, event loss, unsupported capabilities, and uncertainty.
- **Artifacts** — relevant files, outputs, and hashes when available.
- **Run context** — the association between execution evidence and MartinLoop's independently maintained run contract.

## How it fits together

```text
Operating System
       |
       v
   NativeRelay
 Native observations
       |
       v
   AgentTrace
 Correlation and event history
       |
       v
   TraceBridge
 Validate · Correlate · Normalize
 Preserve provenance · Package evidence
       |
       v
   MartinLoop
 Governed execution · Verification
 Final run status and receipt
```

Each component has a distinct responsibility.

| Component | Responsibility |
|---|---|
| **NativeRelay** | Supplies native operating-system observations through its supported collectors. |
| **AgentTrace** | Records and correlates agent execution activity and provides the available event history and integrity information. |
| **TraceBridge** | Validates incoming records, associates evidence with the appropriate run, normalizes the data, and exposes provenance, integrity, and coverage limitations. |
| **MartinLoop** | Owns the governed run, including its objective, scope, budget, verification requirements, and final status. |

TraceBridge does not replace any of these components. It connects their responsibilities through a defined evidence contract.

## What TraceBridge is responsible for

### 1. Validate incoming evidence

Check the structure and required fields of incoming records, preserve original identifiers, and identify malformed or conflicting information.

### 2. Associate evidence with the correct run

Correlate AgentTrace observations with MartinLoop's run and repository context. Ambiguous or conflicting associations must be surfaced rather than silently resolved.

### 3. Normalize without losing meaning

Present records in a consistent structure while preserving their original source, event type, attribution, timestamps, and relevant metadata.

### 4. Preserve integrity and coverage information

Expose the integrity checks that actually ran, their results, and any known gaps or event loss. A successful integrity check must not be presented as proof of complete observation.

### 5. Produce a structured evidence package

Provide MartinLoop with organized execution evidence, artifact information, provenance, coverage limitations, and references to the associated run context.

### 6. Keep evidence separate from evaluation

TraceBridge supplies evidence. MartinLoop remains responsible for evaluating the governed run and recording its final status.

## Evidence is not the same as a conclusion

TraceBridge must preserve the difference between what was observed, what was inferred, and what remains unknown.

For example:

| Evidence | What it establishes |
|---|---|
| A process was observed starting. | The source recorded a process-start event. |
| A command returned exit code `0`. | The recorded process returned that exit code. |
| A file hash matches a recorded digest. | The compared bytes match under the stated hash algorithm. |
| An event chain passes integrity verification. | The checked chain passed the implemented verification procedure. |
| Some events were lost or could not be collected. | The evidence has a known limitation that must remain visible. |

None of these facts independently proves that the overall task was completed correctly.

Similarly, the absence of an event does not prove that the corresponding activity never occurred.

## Integrity and coverage

TraceBridge is designed to expose integrity and coverage as separate dimensions.

Integrity concerns whether the available records pass the checks performed on them. Coverage concerns which activity could be observed and what may be missing.

A report may therefore indicate that an event chain passed verification while filesystem coverage remains incomplete or unavailable.

TraceBridge must not report `TRACE INTACT` unless the underlying checks support that claim within a clearly defined scope. It must never turn an unavailable check, unknown loss count, or incomplete observation into a successful verification result.

## Data contract

`DATA-CONTRACT.md` is the canonical reference for the proposed exchange format.

It defines:

- Run and repository identity.
- Agent and session identity.
- Observation events and ordering.
- Process and command execution records.
- Filesystem activity and artifacts.
- Hashes, integrity checks, coverage, and event loss.
- Provenance and attribution.
- MartinLoop run context and evaluation results.
- The normalized evidence package and validation rules.

The contract also distinguishes proposed fields from implemented capabilities. Actual compatibility must be verified against the interfaces exposed by AgentTrace and MartinLoop.

## Design principles

- **Preserve provenance.** Evidence should remain traceable to its source.
- **Do not manufacture certainty.** Unknown and inferred information must remain distinguishable from direct observations.
- **Expose limitations.** Missing events and unsupported capabilities must not disappear during normalization.
- **Keep ownership clear.** TraceBridge does not take over AgentTrace's event history or MartinLoop's run decisions.
- **Prefer explicit contracts.** Data structures, status values, and compatibility expectations should be defined rather than assumed.
- **Keep the integration focused.** TraceBridge should connect existing systems instead of becoming another execution-control or monitoring platform.

## Project documentation

This repository maintains three core documents:

- **`README.md`** — project purpose, component roles, and high-level workflow.
- **`ARCHITECTURE.md`** — system boundaries, responsibilities, and data flow.
- **`DATA-CONTRACT.md`** — data structures, field definitions, validation rules, and evidence semantics.

## Current status

TraceBridge is in its contract and integration-design stage.

The immediate priorities are:

1. Finalize the data contract against the actual AgentTrace and MartinLoop interfaces.
2. Define the input validation and run-correlation behavior.
3. Implement evidence normalization while preserving provenance and uncertainty.
4. Establish integrity and coverage reporting based on real available checks.
5. Validate the resulting evidence package against representative integration scenarios.

Features described as design goals should not be assumed to exist until they are implemented and tested.

## Core principle

**TraceBridge connects the evidence to the run. It does not turn evidence into more certainty than it supports.**
