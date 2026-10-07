# Architecture

TraceBridge sits between **AgentTrace** and **MartinLoop**.

Its purpose is to turn observations and execution evidence collected by AgentTrace into structured evidence that MartinLoop can consume when evaluating an agent run.

## System Overview

```text
┌────────────────────┐
│        OS          │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│    NativeRelay     │
│                    │
│ Native OS          │
│ observations       │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│     AgentTrace     │
│                    │
│ Agent/session      │
│ Process activity   │
│ Workspace activity │
│ Execution history  │
└─────────┬──────────┘
          │
          │ Evidence
          ▼
┌────────────────────┐
│    TraceBridge     │
│                    │
│ Validate           │
│ Normalize          │
│ Correlate          │
│ Package            │
└─────────┬──────────┘
          │
          │ Verification evidence
          ▼
┌────────────────────┐
│    MartinLoop      │
│                    │
│ Run control        │
│ Verification       │
│ Completion         │
│ Receipts           │
└────────────────────┘
```

## Component Responsibilities

### NativeRelay

NativeRelay is responsible for **native OS-level observation**.

It provides platform-specific process and filesystem observations through a normalized event interface.

NativeRelay does not determine:

* which AI agent caused an event
* whether a task was completed
* whether an observation proves a requirement
* whether a run should be considered successful

Those responsibilities belong to higher layers.

### AgentTrace

AgentTrace is responsible for **observing and correlating AI-agent activity**.

It consumes NativeRelay observations and combines them with agent/session information to build an execution trace.

Relevant evidence can include:

* process activity
* workspace activity
* process-attributed filesystem activity
* agent/session relationships
* execution results
* authoritative trace history
* integrity information

AgentTrace is the source of the execution evidence that TraceBridge consumes.

### TraceBridge

TraceBridge is responsible for the **evidence boundary between AgentTrace and MartinLoop**.

It should:

1. receive AgentTrace evidence
2. validate the evidence structure
3. preserve evidence provenance and uncertainty
4. associate evidence with the relevant run/repository
5. transform it into a format suitable for MartinLoop
6. expose evidence boundaries rather than inventing missing information

TraceBridge should not become another monitoring system or another run controller.

### MartinLoop

MartinLoop is responsible for **governing and verifying the run**.

It defines the task, controls execution, applies verification requirements, and determines the resulting run state.

TraceBridge provides additional evidence to this process; it does not replace MartinLoop's own verification mechanisms.

## Evidence Flow

The intended flow is:

```text
OS event
   ↓
NativeRelay
   ↓
AgentTrace observation
   ↓
Agent/session correlation
   ↓
Authoritative AgentTrace history
   ↓
TraceBridge
   ↓
Structured evidence
   ↓
MartinLoop verification
```

The important boundary is between **observation** and **interpretation**.

For example:

```text
Observed:
Process 1234 opened file A.

Inferred:
Process 1234 belongs to Agent Session X.

Not established:
Agent Session X read the contents of file A.
```

TraceBridge must preserve these distinctions.

## Attribution

AgentTrace may have different levels of confidence when connecting an observation to an agent session.

TraceBridge must not convert an inferred relationship into a definitive fact.

Possible attribution states include:

* `direct`
* `inferred`
* `unknown`

The exact schema will be defined in the evidence contract.

## Evidence Boundaries

An absence of an observation must not automatically be interpreted as proof that an action did not occur.

For example:

> File X was not observed being read.

does not necessarily mean:

> The agent did not read File X.

The first statement describes the available evidence. The second makes a stronger claim that may not be supported by the collector's coverage.

TraceBridge should preserve this distinction.

## Integrity

AgentTrace maintains the authoritative execution history.

TraceBridge consumes that evidence rather than treating exported or mutable representations as the authoritative source.

The intended integrity boundary is:

```text
NativeRelay
    ↓
AgentTrace authoritative recorder
    ↓
Integrity verification
    ↓
TraceBridge
    ↓
MartinLoop
```

TraceBridge should preserve integrity information and report evidence that is incomplete, invalid, or unable to be verified.

## Scope

TraceBridge is intentionally narrow.

It is **not**:

* an operating-system monitoring framework
* an AI coding agent
* a replacement for AgentTrace
* a replacement for NativeRelay
* a replacement for MartinLoop's verifier
* a general-purpose observability platform

Its purpose is to provide a reliable evidence boundary between AgentTrace and MartinLoop.

## Design Principle

The core principle is:

> **Pass evidence forward without making it stronger than the observation supports.**

AgentTrace observes.

TraceBridge structures and transports the evidence.

MartinLoop evaluates it within the context of a governed run.
