# TraceBridge

**An evidence bridge between AgentTrace and MartinLoop.**

TraceBridge connects low-level execution evidence from [AgentTrace](https://github.com/VeloraTech/AgentTrace) with [MartinLoop](https://github.com/keesan12/martin-loop), turning observed agent activity into structured evidence that can be consumed during task verification.

## Why TraceBridge?

AI coding agents can produce a result without providing enough evidence to understand what actually happened during a run.

AgentTrace observes and records the activity surrounding an agent run — including processes, workspace activity, and system-level observations.

MartinLoop governs the run and determines whether the task can be considered complete based on its verification and evidence requirements.

TraceBridge sits between them.

```text
NativeRelay
    │
    │ OS-level observations
    ▼
AgentTrace
    │
    │ Agent/session evidence
    ▼
TraceBridge
    │
    │ Structured verification evidence
    ▼
MartinLoop
```

## What TraceBridge Does

TraceBridge is responsible for:

* receiving evidence produced by AgentTrace
* validating and normalizing that evidence
* preserving the distinction between observed, inferred, and unknown information
* connecting evidence to the relevant run and repository
* producing a structured evidence package for MartinLoop

TraceBridge does **not** perform OS-level monitoring itself, and it does not replace AgentTrace or MartinLoop.

### Responsibility boundaries

| Component       | Responsibility                                                                   |
| --------------- | -------------------------------------------------------------------------------- |
| **NativeRelay** | Collects native OS-level observations                                            |
| **AgentTrace**  | Records and correlates agent, process, and workspace activity                    |
| **TraceBridge** | Transforms AgentTrace observations into verification-ready evidence              |
| **MartinLoop**  | Governs the run and evaluates whether the available evidence supports completion |

## Evidence First

TraceBridge is designed around a simple principle:

> **Evidence should describe what was actually observed, not what we assume happened.**

If an observation cannot be established with confidence, TraceBridge should preserve that uncertainty rather than turning it into a definitive claim.

This allows downstream systems to distinguish between:

* observed evidence
* inferred relationships
* unavailable evidence
* incomplete or lost observations

## Status

**Early development.**

The initial work is focused on defining the contracts between AgentTrace, TraceBridge, and MartinLoop before implementation begins.

## Roadmap

* [ ] Define AgentTrace evidence contract
* [ ] Define MartinLoop integration contract
* [ ] Define TraceBridge evidence model
* [ ] Define evidence integrity requirements
* [ ] Implement evidence validation
* [ ] Implement AgentTrace → TraceBridge pipeline
* [ ] Implement TraceBridge → MartinLoop integration
* [ ] Add integration and conformance tests

## Related Projects

* **[AgentTrace](https://github.com/VeloraTech/AgentTrace)** — observes and records AI-agent activity.
* **[NativeRelay](https://github.com/VeloraTech/NativeRelay)** — provides native OS-level observations to AgentTrace.
* **[MartinLoop](https://github.com/keesan12/martin-loop)** — governs AI coding-agent runs and verifies their outcomes.

## License

MIT
