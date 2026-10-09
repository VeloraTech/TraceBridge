# DATA-CONTRACT.md

# TraceBridge Data Contract

**Contract version:** `0.1.0`  
**Status:** Proposed  
**Purpose:** Define the evidence exchanged between AgentTrace, TraceBridge, and MartinLoop.

## 1. Purpose

TraceBridge connects AgentTrace's execution observations to MartinLoop's governed-run context.

Its responsibility is to receive, validate, correlate, normalize, and package evidence without changing what that evidence actually establishes.

The contract defines:

- The identity of a governed run, repository, agent, session, and process.
- The observations AgentTrace can supply.
- The metadata needed to establish event ordering, provenance, coverage, and integrity.
- The files and artifacts associated with a run.
- The run context and evaluation results owned by MartinLoop.
- The evidence package TraceBridge returns.
- The rules governing missing, incomplete, unsupported, inferred, or invalid information.

This document describes the **proposed contract**. Fields and capabilities must not be treated as implemented until the relevant producer, consumer, and validation logic exist.

## 2. Core ownership rules

| Component | Owns or is responsible for |
|---|---|
| NativeRelay | Native operating-system observations made available by its collectors. |
| AgentTrace | Agent/session/process/workspace correlation, event history, observation provenance, and the integrity information it can actually verify. |
| TraceBridge | Input validation, correlation against run context, normalization, provenance preservation, coverage reporting, and evidence packaging. |
| MartinLoop | Run objective, allowed scope, budget, verification requirements, verifier outcome, stop reason, and final run status. |

TraceBridge may reference MartinLoop's run context in its evidence package, but must not silently replace or reinterpret that context.

Likewise, a MartinLoop verifier passing does not establish that every relevant system action was observed. An intact event chain does not establish that the collector captured every event.

**Core principle: Pass evidence forward without making it stronger than the observation supports.**

## 3. Contract conventions

### 3.1 Field requirements

| Convention | Meaning |
|---|---|
| Required | Must be present for the specified object to be valid. |
| Optional | May be omitted when unavailable or irrelevant. |
| Nullable | The field is expected, but its value may be `null` when unknown or unavailable. |
| Conditional | Required when the associated event or capability applies. |

An unavailable value must not be replaced with a fabricated default.

For example, an unknown exit code must remain unknown; it must not become `0`.

### 3.2 Implementation status

Every proposed field or capability should be tracked using one of these statuses in implementation documentation or code:

- `implemented` — implemented and tested in the relevant component.
- `partial` — supported only for some platforms, event types, or execution paths.
- `proposed` — defined here but not yet implemented.
- `unsupported` — explicitly not supported by the current implementation.

These are development-status labels, not evidence-verification results.

### 3.3 Serialization

The initial wire-format proposal uses JSON objects with UTF-8 encoding.

- Timestamps use RFC 3339 format with an explicit timezone, preferably UTC.
- Identifiers are opaque strings unless a field explicitly defines another format.
- Paths are represented as strings and interpreted relative to a declared repository or workspace root when applicable.
- Durations use milliseconds where a numeric duration is required.
- Byte counts use non-negative integers.
- Hashes include an explicit algorithm and digest.
- Enumerated values are case-sensitive.

A JSONL transport may be used for streams of events, but the transport format does not by itself establish event authenticity, completeness, or integrity.

The final wire format must be checked against the actual AgentTrace and MartinLoop interfaces before implementation is declared compatible.

## 4. Shared run identity

A governed run may have identifiers from multiple systems. TraceBridge must preserve those identifiers instead of assuming that one system's run ID is interchangeable with another's.

### 4.1 `RunIdentity`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `tracebridge_run_id` | string | Required | Stable identifier for the TraceBridge association, if one is created. |
| `agenttrace_run_id` | string or null | Nullable | Run identifier assigned by AgentTrace. |
| `martinloop_run_id` | string or null | Nullable | Run identifier assigned by MartinLoop. |
| `parent_run_id` | string or null | Optional | Parent or originating run when explicitly known. |
| `correlation_id` | string | Required | Identifier used to correlate the supplied evidence and context. |
| `correlation_method` | string | Required | How the association was established, such as an explicit shared ID or an explicitly configured mapping. |
| `correlation_status` | enum | Required | `matched`, `ambiguous`, `unmatched`, or `conflict`. |

**Rules**

1. A run association must be supported by an explicit identifier or documented correlation method.
2. Similar timestamps, repository paths, or agent names alone must not be treated as conclusive proof that two records belong to the same run.
3. If multiple candidate runs match, report `ambiguous` instead of selecting one silently.
4. Conflicting run identifiers must be preserved and surfaced for review.
5. TraceBridge must not invent a MartinLoop run ID when no valid association exists.

## 5. Repository identity

### 5.1 `RepositoryIdentity`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `repository_id` | string or null | Nullable | Stable repository identifier, if provided by an integration. |
| `repository_name` | string or null | Nullable | Human-readable repository name. |
| `repository_root` | string or null | Nullable | Absolute workspace or repository root observed or supplied. |
| `repository_url` | string or null | Optional | Repository URL, if known. |
| `vcs_type` | string or null | Optional | Version-control system, such as `git`. |
| `starting_commit` | string or null | Nullable | Starting commit associated with the run. |
| `ending_commit` | string or null | Nullable | Ending commit associated with the run. |
| `working_tree_state` | enum | Required | `clean`, `dirty`, `unknown`, or `not_applicable`. |
| `identity_source` | string | Required | Component or record that supplied the repository identity. |

**Rules**

- Starting and ending commits must come from observed or explicitly supplied version-control records.
- A missing commit must remain `null`; TraceBridge must not infer it from the latest commit on a branch.
- A repository URL is not proof that the observed workspace corresponds to that remote repository.
- A working-tree snapshot and a commit identify different things. The contract must preserve that distinction.
- Paths from different roots must not be merged without an explicit mapping.

## 6. Agent and session identity

### 6.1 `AgentSessionIdentity`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `agent_id` | string or null | Nullable | Agent identifier supplied by AgentTrace or its integration. |
| `agent_name` | string or null | Nullable | Agent name, such as Codex or Claude Code, when known. |
| `agent_version` | string or null | Optional | Agent version, if observed or supplied. |
| `session_id` | string or null | Nullable | Session identifier. |
| `session_start_time` | timestamp or null | Optional | Observed or supplied session start. |
| `session_end_time` | timestamp or null | Optional | Observed or supplied session end. |
| `identity_source` | string | Required | Source of the identity. |
| `attribution` | enum | Required | `direct`, `inferred`, or `unknown`. |

An agent name alone does not prove which agent process performed an action. If attribution depends on a correlation rule, that rule and its limitations should be included in the evidence.

## 7. Observation event contract

The event is the core unit of AgentTrace evidence consumed by TraceBridge.

### 7.1 `ObservationEvent`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `event_id` | string | Required | Unique event identifier within the originating event system. |
| `source_system` | string | Required | Component that emitted the event. |
| `source_event_id` | string | Required | Original event identifier, preserved through normalization. |
| `agenttrace_run_id` | string or null | Nullable | Associated AgentTrace run. |
| `timestamp` | timestamp or null | Nullable | Time associated with the event. |
| `timestamp_source` | string or null | Optional | Clock or component that supplied the timestamp. |
| `sequence_number` | integer or null | Nullable | Source sequence number, if available. |
| `event_type` | string | Required | Original event type. |
| `category` | enum | Required | Normalized event category. |
| `operation` | string or null | Optional | Specific operation represented by the event. |
| `observation_status` | enum | Required | Status of the observation. |
| `attribution` | enum | Required | Strength of attribution to an agent or process. |
| `process_id` | string or null | Optional | Associated process identifier. |
| `agent_id` | string or null | Optional | Associated agent identifier. |
| `session_id` | string or null | Optional | Associated session identifier. |
| `repository_id` | string or null | Optional | Associated repository identifier. |
| `resource` | object or null | Optional | File, process, command, or other affected resource. |
| `provenance` | object | Required | Information identifying the source and transformation history. |
| `metadata` | object | Optional | Additional source-specific fields that are not represented elsewhere. |

### 7.2 Normalized event categories

The proposed initial categories are:

- `process`
- `execution`
- `filesystem`
- `workspace`
- `artifact`
- `integrity`
- `coverage`
- `verification`

The original `event_type` must remain available even when TraceBridge assigns a normalized category. Unknown event types should be preserved rather than discarded simply because the current schema does not recognize them.

A category is a classification, not a claim that the event has been independently verified.

### 7.3 Observation status

Use the following proposed values:

| Value | Meaning |
|---|---|
| `observed` | The source reports that the event was observed. This does not automatically mean the event's authenticity or interpretation has been independently verified. |
| `unknown` | The available evidence cannot establish whether the event occurred. |
| `unsupported` | The relevant event or capability is not supported by the source implementation. |
| `permission_denied` | The source could not obtain the observation because access was denied. |
| `degraded` | Observation was available but subject to a known limitation. |
| `lost` | The source reports that expected event data was lost or dropped. |

These statuses must not be collapsed into a simple Boolean such as `success` or `failure`.

In particular, no event in the trace is not proof that the event never occurred.

### 7.4 Attribution

| Value | Meaning |
|---|---|
| `direct` | The source directly associates the event with the relevant process, session, or agent using the available evidence. |
| `inferred` | The association depends on a correlation rule or inference. |
| `unknown` | The responsible agent or process cannot be established. |

An inferred association must never be silently converted into direct attribution.

### 7.5 Event ordering

TraceBridge must preserve source ordering information wherever available.

- Prefer source sequence numbers for ordering within a source stream.
- Preserve timestamps independently of sequence numbers.
- Do not assume timestamps from different clocks provide a perfect total order.
- If events cannot be reliably ordered, expose the ordering limitation.
- Do not manufacture sequence numbers and present them as source-issued values.

A normalized presentation order may be added for readability, but it must remain distinguishable from original source ordering.

## 8. Process and command execution records

### 8.1 `ProcessRecord`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `process_record_id` | string | Required | Identifier for the normalized record. |
| `pid` | integer or null | Nullable | Operating-system process ID. |
| `process_start_time` | timestamp or null | Nullable | Process start time, when available. |
| `process_generation` | string or null | Optional | Additional identity information to distinguish PID reuse, when available. |
| `parent_pid` | integer or null | Optional | Parent process ID. |
| `executable` | string or null | Nullable | Executable path or name. |
| `arguments` | array of strings or null | Optional | Observed command arguments, subject to redaction and collection limits. |
| `working_directory` | string or null | Optional | Observed working directory. |
| `agent_id` | string or null | Optional | Associated agent. |
| `session_id` | string or null | Optional | Associated session. |
| `started_at` | timestamp or null | Nullable | Start time. |
| `ended_at` | timestamp or null | Nullable | End time. |
| `exit_code` | integer or null | Nullable | Actual exit code, if captured. |
| `termination_status` | enum | Required | `exited`, `signaled`, `unknown`, or `not_observed`. |
| `attribution` | enum | Required | `direct`, `inferred`, or `unknown`. |
| `source_event_ids` | array of strings | Required | Source events supporting the record. |

A PID is not a globally unique process identity. Where available, process start time or generation information should be retained to reduce ambiguity caused by PID reuse.

### 8.2 `CommandExecution`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `execution_id` | string | Required | Identifier for the execution record. |
| `process_record_id` | string or null | Nullable | Associated process record. |
| `command` | string or null | Nullable | Observed command representation. |
| `arguments` | array of strings or null | Optional | Captured arguments. |
| `working_directory` | string or null | Optional | Working directory. |
| `started_at` | timestamp or null | Nullable | Start time. |
| `ended_at` | timestamp or null | Nullable | End time. |
| `exit_code` | integer or null | Nullable | Captured exit code. |
| `stdout_capture` | enum | Required | `captured`, `partial`, `not_captured`, `redacted`, or `unknown`. |
| `stderr_capture` | enum | Required | `captured`, `partial`, `not_captured`, `redacted`, or `unknown`. |
| `failure_status` | enum | Required | `none_observed`, `failure_observed`, or `unknown`. |
| `source_event_ids` | array of strings | Required | Supporting source events. |

**Important distinctions**

- A command being launched does not prove it completed.
- A process ending does not prove its exit code was captured.
- An exit code of `0` establishes only the process result represented by that code; it does not prove the broader task succeeded.
- Missing stdout or stderr does not mean those streams were empty.
- Command arguments and output may contain credentials, tokens, or other sensitive data. Collection, retention, and export must follow explicit redaction rules.

## 9. Filesystem and workspace observations

### 9.1 `FilesystemEvent`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `filesystem_event_id` | string | Required | Identifier for the normalized filesystem observation. |
| `operation` | enum | Required | `create`, `open`, `read`, `write`, `modify`, `delete`, `rename`, `metadata_change`, or `unknown`. |
| `path` | string or null | Nullable | Observed path. |
| `destination_path` | string or null | Optional | Destination path for operations such as rename. |
| `repository_relative_path` | string or null | Optional | Path relative to the declared repository root. |
| `process_record_id` | string or null | Optional | Associated process. |
| `timestamp` | timestamp or null | Nullable | Event timestamp. |
| `observation_status` | enum | Required | Observation status defined in Section 7. |
| `attribution` | enum | Required | Attribution defined in Section 7. |
| `content_captured` | boolean or null | Nullable | Whether file content was actually captured. |
| `source_event_ids` | array of strings | Required | Supporting source events. |

Filesystem events must retain the distinction between the operation observed and any interpretation of its consequences.

For example:

- `open` does not prove that the file's contents were read.
- `read` does not prove that the full contents were captured.
- `write` does not necessarily establish the final file contents.
- `modify` does not by itself establish which exact bytes changed.
- A path appearing in an event does not prove the associated repository-relative path is correct unless the root mapping is established.

Platform and collector differences must be reported through capability and coverage metadata. TraceBridge must not imply that every filesystem operation can be observed on every supported platform.

## 10. Artifacts and hashes

### 10.1 `ArtifactRecord`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `artifact_id` | string | Required | Stable identifier for the artifact record. |
| `artifact_type` | string | Required | Type of artifact, such as `file`, `patch`, `log`, `test_report`, or `evidence_package`. |
| `path` | string or null | Nullable | Artifact path, if known. |
| `repository_relative_path` | string or null | Optional | Path relative to the repository root. |
| `operation` | enum | Required | `created`, `modified`, `deleted`, `captured`, or `unknown`. |
| `size_bytes` | integer or null | Nullable | Byte length of the artifact, if measured. |
| `hash` | object or null | Nullable | Hash information for the exact bytes hashed. |
| `captured_at` | timestamp or null | Optional | Time the artifact or its metadata was captured. |
| `source_event_ids` | array of strings | Required | Events supporting the artifact association. |
| `provenance` | object | Required | Source and capture details. |

### 10.2 Hash object

Proposed structure:

```json
{
  "algorithm": "sha256",
  "digest": "<64-character-hex-digest>",
  "byte_length": 12345,
  "hashed_at": "2026-09-27T12:00:00Z",
  "hash_scope": "captured_artifact_bytes"
}
```

The example is illustrative, not a real artifact hash.

Hash rules:

1. The algorithm must be named explicitly.
2. The digest must represent the bytes actually hashed.
3. The hashed byte length should be recorded.
4. If artifact bytes were unavailable, the hash must be `null`; it must not be fabricated from a path or filename.
5. A hash match establishes a match between the compared bytes under the stated algorithm. It does not by itself prove who created the file, when it was created, or whether every relevant file was captured.
6. A deleted file may have an artifact record without a recoverable content hash.
7. If a hash is computed from a patch, snapshot, or exported representation rather than the original file, the scope must state that distinction.

TraceBridge should preserve the original artifact hash and its source. If TraceBridge computes a new hash for its own package, that hash must be identified as a TraceBridge-generated package hash, not an AgentTrace observation.

## 11. Integrity verification

Integrity information must distinguish checks that were performed from checks that were unavailable or never implemented.

### 11.1 `IntegrityCheck`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `check_id` | string | Required | Identifier for the check. |
| `check_name` | string | Required | Name of the check. |
| `status` | enum | Required | `verified`, `failed`, `incomplete`, `unavailable`, or `not_checked`. |
| `checked_at` | timestamp or null | Nullable | Time the check was performed. |
| `checker` | string | Required | Component that performed the check. |
| `scope` | string | Required | Events, records, artifacts, or time range covered by the check. |
| `details` | string or null | Optional | Explanation of the result or limitation. |
| `evidence_refs` | array of strings | Required | References to records supporting the result. |

### 11.2 Required integrity dimensions

The proposed initial integrity summary should report these dimensions separately:

- `native_events`
- `event_chain`
- `process_records`
- `filesystem_records`
- `export_consistency`

Each dimension uses the status values in Section 11.1.

These checks must be reported only when the relevant capability exists and the check has actually run. An unsupported check is `unavailable` or `not_checked`, not `verified`.

**Interpretation limits**

- A valid event chain does not establish that every system event was captured.
- Valid process records do not prove that all processes were observed.
- Valid filesystem records do not prove complete filesystem coverage.
- Export consistency does not prove that the original source was complete or authentic.
- A hash chain alone does not establish who created the records or protect against every form of tampering.

If the underlying implementation supports stronger mechanisms such as signed checkpoints, those mechanisms must be described separately, including what they protect and what they do not protect.

### 11.3 `TraceIntegritySummary`

Proposed machine-readable structure:

```json
{
  "integrity_status": "incomplete",
  "checks": {
    "native_events": "verified",
    "event_chain": "verified",
    "process_records": "verified",
    "filesystem_records": "unavailable",
    "export_consistency": "verified"
  },
  "coverage_status": "partial",
  "loss_detected": true,
  "loss_count": null,
  "limitations": [
    "Filesystem integrity could not be checked by the available collector."
  ]
}
```

This is an illustrative payload. It does not represent a completed run.

The overall status must not be `verified` merely because most checks passed. If a required check fails, is incomplete, or is unavailable, the overall result must preserve that limitation.

The human-readable summary may use `TRACE INTACT` only when the defined integrity requirements have passed and the claim is appropriately scoped. It must not use that phrase to imply complete observation of all system activity.

## 12. Coverage, capabilities, and event loss

### 12.1 `CoverageReport`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `coverage_id` | string | Required | Identifier for the coverage report. |
| `platform` | string or null | Nullable | Operating system or platform. |
| `collector` | string or null | Nullable | Collector involved. |
| `capabilities` | array of objects | Required | Capabilities supported or attempted. |
| `coverage_status` | enum | Required | `complete_for_declared_scope`, `partial`, `unknown`, or `not_assessed`. |
| `coverage_start` | timestamp or null | Optional | Beginning of the assessed interval. |
| `coverage_end` | timestamp or null | Optional | End of the assessed interval. |
| `loss_detected` | boolean or null | Nullable | Whether loss was detected. |
| `loss_count` | integer or null | Nullable | Number of lost events, if known. |
| `loss_reason` | string or null | Optional | Reason for loss, if known. |
| `limitations` | array of strings | Required | Known gaps or constraints. |

Each capability object should identify the capability, whether it is supported, and any relevant limitation.

Coverage must be stated relative to a declared scope. `complete_for_declared_scope` means the available checks support that specific scope; it must not imply that the collector observes every possible system action.

### 12.2 Loss and gaps

When a source reports missing events, TraceBridge must preserve:

- The fact that loss was detected.
- The affected source or stream.
- The affected interval or sequence range, if known.
- The number of lost events, if known.
- The reason for loss, if known.
- Any downstream effect on attribution, ordering, or integrity.

Unknown loss counts must remain `null`.

If no loss was detected, report that no loss was detected by the available mechanism. Do not convert this into a claim that loss was impossible.

## 13. Provenance

### 13.1 `Provenance`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `source_system` | string | Required | Original system that supplied the information. |
| `source_record_id` | string or null | Nullable | Original record identifier. |
| `source_version` | string or null | Optional | Version of the producing component. |
| `collector` | string or null | Optional | Collector responsible for the observation. |
| `collection_method` | string or null | Optional | Method by which the information was obtained. |
| `transformation_history` | array of objects | Required | Normalization or transformation steps applied. |
| `original_reference` | string or null | Optional | Reference to the original record or authoritative source. |

Every normalized record should be traceable to its source when the source provides sufficient information.

TraceBridge must distinguish:

1. Information directly reported by AgentTrace.
2. Information derived by correlating multiple source records.
3. Information supplied by MartinLoop.
4. Metadata generated by TraceBridge.
5. Information that remains unknown.

Normalization must not erase the original event type, source identifier, attribution level, or observation limitations.

## 14. MartinLoop run context

MartinLoop owns the governed run contract and its evaluation. TraceBridge consumes this context for association and evidence packaging.

### 14.1 `MartinLoopRunContext`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `martinloop_run_id` | string | Required | MartinLoop's run identifier. |
| `objective` | string | Required | Task or objective assigned to the run. |
| `repository` | object | Required | Repository and workspace context. |
| `allowed_scope` | object | Required | Permitted paths, operations, or boundaries, when defined by the run contract. |
| `budget` | object or null | Nullable | Configured resource or token budget and related usage information. |
| `verification_requirements` | array of objects | Required | Configured verifier commands or acceptance checks. |
| `expected_outputs` | array of objects | Required | Expected deliverables, if defined. |
| `agent_context` | object or null | Optional | Agent and session identifiers supplied by MartinLoop. |
| `started_at` | timestamp or null | Nullable | Run start time. |
| `ended_at` | timestamp or null | Nullable | Run end time. |
| `context_source` | string | Required | Source of the run contract. |

An empty `allowed_scope` or `expected_outputs` array must not automatically mean that all paths or outputs are permitted. The semantics must follow MartinLoop's actual run contract.

### 14.2 Budget

The proposed budget object is:

```json
{
  "limit": 5,
  "currency": "USD",
  "unit": "cost",
  "usage": 2.4,
  "usage_provenance": "reported",
  "source": "martinloop"
}
```

This is illustrative only.

The implementation must preserve the unit, limit, usage source, and any distinction between actual, calculated, estimated, or unavailable usage. Token budgets and monetary budgets must not be treated as interchangeable.

TraceBridge may attach budget context to the evidence package, but must not recalculate or overwrite MartinLoop's authoritative accounting without an explicitly defined integration contract.

### 14.3 Verification and final status

The following information belongs to MartinLoop:

- Configured verification requirements.
- Commands run for verification.
- Verification exit codes and supporting evidence.
- Verifier outcome.
- Budget and policy outcomes.
- Stop reason.
- Final governed-run status.

TraceBridge may include references to these records and compare them with observed evidence. It must not independently declare the governed task complete merely because an observed process exited successfully or a filesystem change occurred.

## 15. TraceBridge evidence package

The evidence package is the primary output of TraceBridge.

### 15.1 `EvidencePackage`

| Field | Type | Requirement | Meaning |
|---|---|---|---|
| `schema_version` | string | Required | Version of the evidence-package schema. |
| `package_id` | string | Required | Unique identifier for the package. |
| `generated_at` | timestamp | Required | Time the package was generated. |
| `run_identity` | object | Required | Cross-system run identity and correlation result. |
| `repository` | object | Required | Repository identity and commit context. |
| `agent_sessions` | array | Required | Agent and session identities relevant to the run. |
| `events` | array | Required | Normalized observations, preserving source references. |
| `processes` | array | Required | Process records. |
| `executions` | array | Required | Command execution records. |
| `filesystem_events` | array | Required | Filesystem observations. |
| `artifacts` | array | Required | Artifact records and hashes, when available. |
| `integrity` | object | Required | Integrity checks and overall integrity status. |
| `coverage` | object | Required | Coverage, capabilities, and known gaps. |
| `martinloop_context` | object or null | Nullable | Associated MartinLoop run context. |
| `martinloop_result` | object or null | Nullable | MartinLoop's recorded verification and final result, if available. |
| `validation` | object | Required | TraceBridge's validation results. |
| `limitations` | array of strings | Required | Important gaps, ambiguity, and constraints. |

Empty arrays mean that no records are included in that section of the package. They must not be interpreted as proof that no corresponding activity occurred.

### 15.2 TraceBridge validation

The validation object should report:

- Whether the package conforms to the declared schema.
- Whether required fields are present.
- Whether referenced records can be resolved.
- Whether run and repository associations are consistent.
- Whether source identifiers or ordering information conflict.
- Whether integrity and coverage limitations are represented.
- Whether the package is suitable for downstream consumption.

Schema validity is not evidence authenticity. A structurally valid package may still contain incomplete, unavailable, or conflicting observations.

## 16. Example evidence package

The following example demonstrates the shape of the proposed contract. The IDs, values, and events are illustrative; they are not real AgentTrace output or a claim about current implementation.

```json
{
  "schema_version": "0.1.0",
  "package_id": "tb_pkg_example_001",
  "generated_at": "2026-09-27T12:30:00Z",
  "run_identity": {
    "tracebridge_run_id": "tb_run_example_001",
    "agenttrace_run_id": "at_run_example_001",
    "martinloop_run_id": "ml_run_example_001",
    "correlation_id": "corr_example_001",
    "correlation_method": "explicit_shared_identifier",
    "correlation_status": "matched"
  },
  "repository": {
    "repository_id": null,
    "repository_name": "example-project",
    "repository_root": "/workspace/example-project",
    "repository_url": null,
    "vcs_type": "git",
    "starting_commit": "abc123example",
    "ending_commit": "def456example",
    "working_tree_state": "dirty",
    "identity_source": "martinloop_run_context"
  },
  "agent_sessions": [
    {
      "agent_id": "agent_example_001",
      "agent_name": "example-agent",
      "session_id": "session_example_001",
      "identity_source": "agenttrace",
      "attribution": "direct"
    }
  ],
  "events": [
    {
      "event_id": "normalized_event_001",
      "source_system": "agenttrace",
      "source_event_id": "source_event_001",
      "agenttrace_run_id": "at_run_example_001",
      "timestamp": "2026-09-27T12:01:00Z",
      "timestamp_source": "agenttrace",
      "sequence_number": 1,
      "event_type": "process.started",
      "category": "process",
      "operation": "start",
      "observation_status": "observed",
      "attribution": "direct",
      "process_id": "process_example_001",
      "agent_id": "agent_example_001",
      "session_id": "session_example_001",
      "repository_id": null,
      "resource": null,
      "provenance": {
        "source_system": "agenttrace",
        "source_record_id": "source_event_001",
        "source_version": null,
        "collector": null,
        "collection_method": null,
        "transformation_history": [],
        "original_reference": null
      },
      "metadata": {}
    }
  ],
  "processes": [],
  "executions": [],
  "filesystem_events": [],
  "artifacts": [],
  "integrity": {
    "integrity_status": "incomplete",
    "checks": {
      "native_events": "verified",
      "event_chain": "verified",
      "process_records": "not_checked",
      "filesystem_records": "unavailable",
      "export_consistency": "verified"
    },
    "coverage_status": "partial",
    "loss_detected": null,
    "loss_count": null,
    "limitations": [
      "Filesystem coverage was not available in this example."
    ]
  },
  "coverage": {
    "coverage_id": "coverage_example_001",
    "platform": null,
    "collector": null,
    "capabilities": [],
    "coverage_status": "partial",
    "coverage_start": null,
    "coverage_end": null,
    "loss_detected": null,
    "loss_count": null,
    "loss_reason": null,
    "limitations": [
      "Example package contains only one illustrative event."
    ]
  },
  "martinloop_context": null,
  "martinloop_result": null,
  "validation": {
    "schema_valid": true,
    "references_resolved": true,
    "run_association_valid": true,
    "status": "valid"
  },
  "limitations": [
    "Illustrative data only; not an actual execution record."
  ]
}
```

## 17. Human-readable integrity report

TraceBridge should be able to produce a human-readable summary from actual check results.

Example format:

```text
AgentTrace Integrity Check

Run: run_82a91
Events: 18,492

Native events:       VERIFIED
Event chain:         VERIFIED
Process records:     VERIFIED
Filesystem records:  UNAVAILABLE
Export consistency:  VERIFIED

Coverage: PARTIAL
Result: INCOMPLETE
```

The event count must come from the records actually included in the declared scope. If the count is unavailable, it must be reported as unavailable.

The result must be derived from the underlying checks. It must never be hardcoded to `TRACE INTACT`.

If all required integrity checks pass but the collector's coverage is partial, the report must communicate both facts: the checked records passed their integrity checks, and the overall observation coverage remains partial.

## 18. Validation and rejection rules

TraceBridge must validate the package before passing it to MartinLoop.

At minimum, validation must check:

1. Required fields and data types.
2. Valid enum values and timestamp formats.
3. Unique identifiers within their defined scope.
4. Referential consistency between events, processes, sessions, artifacts, and runs.
5. Consistency between repository roots and repository-relative paths.
6. Preservation of source IDs, provenance, attribution, and observation status.
7. Integrity and coverage status consistency.
8. Missing, conflicting, or ambiguous run associations.
9. Artifact hash format and declared algorithm.
10. Redaction of sensitive values according to the integration's explicit rules.

Malformed data must not be silently repaired in ways that change its meaning.

Where safe normalization is possible, TraceBridge should preserve the original value and record the transformation. Where a conflict affects reliable association or interpretation, the package should be marked invalid or incomplete as appropriate.

## 19. Contract guarantees and non-guarantees

### TraceBridge guarantees, once implemented and tested

- It preserves source provenance and original event identifiers.
- It distinguishes direct observations from inferred associations.
- It exposes known gaps, loss, and unsupported capabilities.
- It associates evidence with a run only when the correlation is supported.
- It separates structural validation from integrity verification.
- It separates integrity verification from observation completeness.
- It preserves MartinLoop's ownership of verification and final run status.
- It reports unavailable information instead of inventing it.

### TraceBridge does not guarantee

- That AgentTrace observed every relevant system action.
- That every file read, write, or process invocation is visible on every platform.
- That a valid event chain proves the original source was complete.
- That an artifact hash proves the artifact's creator or origin.
- That a command's successful exit proves the task was completed correctly.
- That an inferred agent attribution is certain.
- That an evidence package being structurally valid makes its claims true.
- That MartinLoop should mark a run complete based on TraceBridge's presence alone.

## 20. Versioning and compatibility

The contract uses semantic versioning for its schema:

- **Major version:** incompatible changes to required fields, field meaning, or interpretation.
- **Minor version:** backward-compatible additions.
- **Patch version:** clarifications and corrections that do not change the contract's meaning.

Consumers must validate the schema version and handle unknown optional fields without silently dropping important provenance.

Any field that changes ownership, evidence meaning, attribution, integrity, or coverage semantics requires explicit review.

Compatibility with AgentTrace and MartinLoop must be validated against their actual versions and interfaces. This document does not assume that every proposed field is already available from either system.

## 21. Initial implementation priorities

The contract should be implemented in this order:

1. **Identity and correlation:** run IDs, repository identity, agent/session identity, and ambiguity handling.
2. **Event ingestion:** source event IDs, timestamps, ordering, observation status, attribution, and provenance.
3. **Execution and filesystem evidence:** process records, command outcomes, file operations, and artifacts.
4. **Integrity and coverage:** checks, capabilities, loss reporting, and honest overall status.
5. **MartinLoop integration:** context association, verification references, and final-result preservation.
6. **Evidence package validation:** schema validation, cross-reference checks, and representative fixtures.

For each priority, implementation tests must cover both valid data and failure cases, including missing fields, conflicting identifiers, unknown attribution, incomplete coverage, lost events, unavailable integrity checks, and malformed hashes.

**Completion criterion:** TraceBridge can produce a validated evidence package whose observations, provenance, integrity, coverage, and limitations are explicit, while leaving MartinLoop's governed-run decision under MartinLoop's control.
