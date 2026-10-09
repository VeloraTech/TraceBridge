# ARCHITECTURE.md

# TraceBridge Architecture

**Status:** Proposed architecture  
**Contract reference:** `DATA-CONTRACT.md`

## 1. Architectural objective

TraceBridge connects AgentTrace's execution evidence with MartinLoop's governed-run context.

Its purpose is to make observations usable across both systems without losing source provenance, introducing unsupported assumptions, or taking ownership of decisions that belong to another component.

TraceBridge is an integration layer, not another operating-system monitor, authoritative event recorder, or run-governance engine.

The architecture follows one principle:

**Preserve the evidence, make its limitations explicit, and leave evaluation to the system responsible for it.**

## 2. System overview

```text
┌───────────────────────────────┐
│         Operating System      │
└───────────────┬───────────────┘
                │ Native observations
                ▼
┌───────────────────────────────┐
│          NativeRelay          │
│  Platform-specific collection │
└───────────────┬───────────────┘
                │ Collected observations
                ▼
┌───────────────────────────────┐
│           AgentTrace          │
│                               │
│ Agent/session correlation     │
│ Process/workspace correlation │
│ Event history and provenance  │
│ Available integrity metadata  │
└───────────────┬───────────────┘
                │ Observations and references
                ▼
┌───────────────────────────────┐
│          TraceBridge          │
│                               │
│ Validate                      │
│ Correlate                     │
│ Normalize                     │
│ Preserve provenance           │
│ Report integrity and coverage │
│ Build evidence package        │
└───────────────┬───────────────┘
                │ Structured evidence
                ▼
┌───────────────────────────────┐
│           MartinLoop          │
│                               │
│ Run contract and constraints  │
│ Budget and stop conditions    │
│ Verification                  │
│ Final run status and receipt   │
└───────────────────────────────┘
```

This diagram represents the intended integration, not a claim that every connection is implemented.

## 3. Component responsibilities

### 3.1 NativeRelay — native observation

NativeRelay is responsible for collecting operating-system observations through the collectors and platform capabilities it supports.

Depending on the implementation, these observations may include process activity, command execution, filesystem activity, and related metadata.

Its output is constrained by operating-system permissions, collector capabilities, and the events it can actually observe.

TraceBridge must not assume that NativeRelay captures every possible system action.

### 3.2 AgentTrace — correlation and event history

AgentTrace is responsible for organizing the available observations into an execution history associated with agents, sessions, processes, and workspaces.

TraceBridge consumes the history and metadata exposed by AgentTrace. It must preserve the distinction between original records and exported representations.

An export should not automatically be treated as the authoritative history simply because it contains the same events or appears structurally valid.

The actual AgentTrace interface must determine which records, identifiers, integrity checks, and references TraceBridge can rely on.

### 3.3 TraceBridge — evidence integration

TraceBridge is responsible for:

1. Accepting AgentTrace observations and the relevant MartinLoop run context.
2. Validating input structure and required fields.
3. Correlating observations with the appropriate run and repository.
4. Preserving original identifiers, timestamps, attribution, and provenance.
5. Normalizing records into the agreed evidence contract.
6. Preserving unknown values, conflicting records, loss information, and coverage limitations.
7. Producing a structured evidence package for downstream consumption.

TraceBridge does not independently determine whether a coding task was completed correctly.

### 3.4 MartinLoop — governed execution

MartinLoop owns the governed run and its evaluation.

Its responsibilities include the run objective, allowed scope, budget, stop conditions, configured verification requirements, verifier results, and final status.

TraceBridge may attach or reference these records in the evidence package. It must not overwrite MartinLoop's authoritative values or substitute its own completion decision.

## 4. Data flow

The architecture has five logical stages.

### Stage 1: Observation ingestion

TraceBridge receives the available AgentTrace observations and associated metadata.

The input adapter must preserve original source identifiers and record types. If the source supplies sequence numbers, timestamps, collector information, or integrity references, these must remain available after ingestion.

Unknown event types should be preserved where possible rather than discarded solely because TraceBridge has not defined a normalized category for them.

### Stage 2: Validation

TraceBridge checks whether the input conforms to the supported source contract.

Validation covers:

- Required fields and data types.
- Timestamp and identifier formats.
- Duplicate or conflicting identifiers.
- Referential consistency.
- Availability of provenance metadata.
- Integrity and coverage fields.
- Source-version compatibility.

Structural validation and integrity verification are different operations. Valid JSON does not establish that an event is authentic or that its contents are complete.

Malformed or conflicting information must be reported rather than silently rewritten.

### Stage 3: Run and repository correlation

TraceBridge associates the observations with MartinLoop's governed run.

Correlation should use explicit shared identifiers or a documented mapping between identifiers. Repository identity, agent/session identity, and timestamps can provide supporting context, but should not independently be treated as conclusive proof of a match.

The correlation result must distinguish:

- `matched` — the available identifiers support the association.
- `ambiguous` — more than one association remains plausible.
- `unmatched` — no supported association was established.
- `conflict` — the available records contradict one another.

TraceBridge must not silently attach ambiguous evidence to a run. It should preserve the evidence and report the unresolved association.

### Stage 4: Normalization and evidence assembly

After validation and correlation, TraceBridge maps supported source records into the normalized structures defined in `DATA-CONTRACT.md`.

The package may contain:

- Run and repository identity.
- Agent and session identity.
- Ordered observations and timestamps.
- Process and command execution records.
- Filesystem observations.
- Artifact metadata and hashes.
- Integrity-check results.
- Coverage and event-loss information.
- Provenance and known limitations.
- References to MartinLoop's run context and evaluation results.

Normalization must not change the meaning of the original observation.

For example, a process exit code must not become a task-completion verdict, and an inferred agent association must not become direct attribution.

### Stage 5: Evidence delivery

TraceBridge produces a structured evidence package for MartinLoop.

The package must expose its schema version, correlation result, validation result, integrity status, coverage status, and important limitations.

MartinLoop can then use the evidence alongside its own run contract and verification records.

The package's presence alone must not be interpreted as proof that the run is complete or verified.

## 5. Evidence and trust boundaries

The architecture separates four distinct questions.

| Question | Responsible component or record |
|---|---|
| What activity was observed? | NativeRelay and AgentTrace, according to their actual capabilities. |
| Which run does the evidence belong to? | TraceBridge, using supported correlation evidence. |
| Are the available records structurally valid and do integrity checks pass? | TraceBridge validates the package; the relevant source or verifier performs the underlying integrity checks. |
| Did the governed task satisfy its requirements? | MartinLoop, using its run contract and verification process. |

These questions are related, but they are not interchangeable.

### 5.1 Observation is not completeness

A collector can report an event without proving that it captured every relevant event.

The absence of an event does not establish that the activity never occurred.

### 5.2 Integrity is not coverage

An event chain can pass verification while the collector has incomplete coverage.

A valid export can faithfully represent the records it contains while still omitting activity that was never captured.

TraceBridge must report these conditions separately.

### 5.3 Correlation is not attribution certainty

A record can be associated with a run through a documented correlation method while the responsible agent remains unknown or inferred.

Run association and actor attribution must therefore remain separate fields.

### 5.4 Verification is not observation

A successful test or verifier command establishes the result of that check within its defined scope. It does not establish that every execution action was observed or that every other requirement was satisfied.

MartinLoop retains responsibility for interpreting the verification results and determining the run's final status.

## 6. Integrity and coverage architecture

Integrity reporting should be derived from actual checks performed by the relevant implementation.

The proposed initial dimensions are:

- Native-event integrity.
- Event-chain integrity.
- Process-record integrity.
- Filesystem-record integrity.
- Export consistency.

Each check should expose its status, scope, checker, time, and supporting references where available.

Supported statuses are defined in `DATA-CONTRACT.md`, including `verified`, `failed`, `incomplete`, `unavailable`, and `not_checked`.

Coverage is reported independently, including supported capabilities, known limitations, and detected event loss.

The overall integrity result must preserve failures and unavailable checks. A missing capability must never be represented as a successful check.

A human-readable `TRACE INTACT` result is permitted only when the defined integrity requirements have actually passed within the stated scope. It must not imply complete observation of all system activity.

## 7. Failure handling

TraceBridge must preserve evidence when possible, even if some of it cannot be safely associated or validated.

| Condition | Required behavior |
|---|---|
| Malformed input | Reject or quarantine the invalid record and report the validation failure. |
| Unknown event type | Preserve the original type and source information where possible. |
| Missing optional metadata | Keep the field unknown or absent according to the contract. |
| Missing required data | Mark the record or package invalid as appropriate. |
| Ambiguous run association | Preserve the evidence and report the ambiguity. |
| Conflicting run or repository identifiers | Surface the conflict rather than choosing silently. |
| Detected event loss | Preserve the loss indication and its known scope. |
| Unknown loss count | Report the count as unknown. |
| Integrity check unavailable | Report `unavailable` or `not_checked`, as appropriate. |
| Unsupported platform capability | Expose the limitation in coverage information. |
| MartinLoop context unavailable | Do not invent run objectives, budgets, or final statuses. |
| Downstream delivery failure | Preserve the package or failure details according to the implemented delivery strategy; do not report successful delivery unless confirmed. |

The implementation must define retry, persistence, and recovery behavior before claiming reliable delivery guarantees.

## 8. Data contract boundary

`DATA-CONTRACT.md` is the canonical definition of the proposed exchange structures.

It defines:

- Run and repository identity.
- Agent and session identity.
- Observation events and ordering.
- Process and command execution records.
- Filesystem activity and artifacts.
- Provenance and attribution.
- Integrity, coverage, and loss reporting.
- MartinLoop run context and results.
- Evidence-package validation and versioning.

This architecture document explains where those structures are produced, consumed, and interpreted. It should not introduce a competing schema.

Any field or capability not supported by the actual source interfaces must be treated as a proposed adapter requirement until implemented.

## 9. Security and data handling

Execution evidence can contain sensitive information, including command arguments, paths, process metadata, file contents, and tool output.

The implementation must define:

- Which data is collected and retained.
- Which fields may contain secrets or personal information.
- What redaction rules apply before evidence is exported.
- How artifact contents and hashes are handled.
- Which components may access evidence packages.
- How temporary files and failed-delivery packages are managed.
- Which integrity guarantees apply to stored and transmitted evidence.

Redaction must be visible where it affects interpretation. A redacted value must not be presented as though the original value was observed and preserved in full.

The architecture must not claim authentication, encryption, tamper resistance, or secure deletion until the corresponding mechanisms are implemented and tested.

## 10. Proposed implementation sequence

### Phase 1 — Contract and adapters

- Validate the proposed schema against the actual AgentTrace interface.
- Identify the actual MartinLoop integration boundary.
- Define source-to-contract field mappings.
- Establish schema validation and representative fixtures.

### Phase 2 — Correlation and normalization

- Implement run and repository association.
- Handle ambiguous and conflicting identities.
- Preserve provenance and source ordering.
- Normalize supported event, process, execution, and artifact records.

### Phase 3 — Integrity and coverage

- Integrate the integrity checks actually available from AgentTrace.
- Report unsupported checks and known gaps.
- Preserve loss information and uncertainty.
- Test that overall integrity status cannot conceal a failed or unavailable required check.

### Phase 4 — Evidence delivery

- Generate the structured evidence package.
- Validate it against the contract.
- Deliver it through the confirmed MartinLoop integration.
- Preserve delivery errors and distinguish attempted delivery from confirmed delivery.

### Phase 5 — Integration testing

Test successful and unsuccessful cases, including:

- Valid and malformed input.
- Correct, ambiguous, and conflicting run identities.
- Missing process or filesystem observations.
- Unknown attribution.
- Incomplete coverage and event loss.
- Failed or unavailable integrity checks.
- Missing MartinLoop context.
- Package validation failures and downstream delivery errors.

## 11. Architecture invariants

The following invariants should hold throughout implementation:

1. Original source identifiers and provenance are preserved.
2. Observed, inferred, and unknown information remain distinguishable.
3. Missing observations are never treated as proof that an action did not occur.
4. Integrity and coverage are reported separately.
5. Unsupported or unperformed checks are never reported as verified.
6. Run and repository associations are not invented.
7. MartinLoop remains the authority for governed-run evaluation and final status.
8. TraceBridge does not claim integration capabilities that have not been implemented and tested.

## 12. Definition of done

The initial integration is ready for release when:

- The source-to-contract mappings are documented and tested.
- Run and repository correlation handles ambiguity and conflicts.
- Evidence normalization preserves source meaning and provenance.
- Integrity and coverage limitations are represented accurately.
- Evidence packages pass schema and referential validation.
- MartinLoop's run context and final status remain under MartinLoop's control.
- Integration tests cover the principal failure paths.
- Documentation accurately distinguishes implemented behavior from proposed capabilities.

The goal is not to claim perfect visibility. It is to make the available evidence structured, traceable, and honest about its limitations.
