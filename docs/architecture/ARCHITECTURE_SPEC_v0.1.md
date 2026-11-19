# CyberCore Conductor — Architecture Specification v0.1

**Status:** Draft for architecture review
**Date:** 2026-11-18
**Scope:** Product and system architecture before runtime implementation
**Working name:** CyberCore Conductor (provisional; naming clearance outstanding)
**Normative language:** “MUST”, “MUST NOT”, “SHOULD”, and “MAY” express requirements and recommendations within this specification.

---

## 1. Executive summary

CyberCore Conductor is a provider-independent autonomous agent runtime and orchestration platform. It executes agent work as durable, observable, policy-governed runs. It owns the control loop around model calls, planning, tool requests, approvals, budgets, memory access, execution environments, persistence, recovery, and delegation.

Conductor is not simply a command-line wrapper around an LLM. Nor is it the CyberCore Agent Evaluation Workbench. Its central responsibility is to make agent execution controllable and recoverable even when models, tools, workers, networks, or the host process fail.

The foundational authority rule is:

> **A model may propose an action; only the runtime may authorize and execute it.**

The model is an untrusted decision-support component. It cannot grant itself permissions, bypass approval, expand budgets, select arbitrary credentials, or escape the configured execution boundary.

The first implementation should be local-first and deliberately narrow: one durable agent run, one provider adapter, a typed tool interface, policy enforcement, SQLite persistence, structured events, and a CLI. Distributed operation and broad interfaces follow only after the core invariants are tested.

## 2. Goals

Conductor MUST provide architectural foundations for:

1. A provider-independent agent execution loop.
2. Explicit agent definitions, versions, and run identities.
3. Planning and iterative model/tool interaction.
4. Typed tool invocation and runtime-enforced authorization.
5. Durable run state, checkpoints, cancellation, and crash recovery.
6. Context assembly and pluggable memory.
7. Per-run budgets and usage accounting.
8. Human approval checkpoints for sensitive actions.
9. Structured events, traces, artifacts, and audit history.
10. Orchestration, scheduling, and eventual multi-agent delegation.
11. Pluggable execution environments with isolation appropriate to workload risk.
12. Independent integration with CyberCore Agent Evaluation.

## 3. Non-goals for v0.1

The architecture does not require the first release to ship:

- a hosted SaaS control plane or multi-tenant cloud service;
- a browser dashboard;
- a general-purpose visual workflow designer;
- every model provider or tool protocol;
- unrestricted autonomous shell or browser access;
- production-grade hostile-code isolation using an in-process or ordinary local process runner;
- a distributed scheduler or cluster consensus system;
- a built-in vector database;
- a claim that an agent's own final answer proves task success.

These may be added through later phases without weakening the core policy, durability, and evidence invariants.

## 4. System context and boundaries

### 4.1 Conductor owns

- agent definition registry and immutable version references;
- run creation and lifecycle;
- planning loop and model interaction orchestration;
- context construction and memory retrieval/write policy;
- tool catalog, tool dispatch, and authorization decisions;
- budgets, rate limits, and approval gates;
- durable state, checkpoints, events, and recovery;
- worker/scheduler coordination;
- delegation and child-run relationships;
- provider, tool, storage, and executor adapter contracts.

### 4.2 Conductor does not own

- independent certification of its own task success;
- the truth or trustworthiness of model output;
- unrestricted access to user credentials or host resources;
- the internal semantics of every external provider or tool;
- evaluation policy that must remain portable across agent runtimes.

### 4.3 CyberCore ecosystem integration

**CyberCore Agent Evaluation** remains a separate, runtime-independent evaluation system. Conductor emits versioned run events, trace metadata, outcome claims, and content-addressed artifact references through an adapter contract. Evaluation independently determines whether evidence supports a claim and returns structured findings. Conductor may consume those findings as feedback, but MUST NOT rewrite evaluation results or treat its own self-assessment as an external verification.

**AgentForge** remains a separate coding-agent orchestration project. If integration is warranted, it should use documented APIs/events or a dedicated adapter rather than merging AgentForge's orchestration domain into Conductor's runtime core.

## 5. Architectural principles and invariants

1. **Runtime authority:** prompts are not security controls. Every tool call and privileged transition is checked by runtime policy.
2. **Durability:** every run has a stable ID and persisted lifecycle. State changes and externally visible side effects are recorded.
3. **Explicit state machine:** invalid transitions are rejected; terminal states are immutable except through explicit administrative/retry operations that create a new attempt or run.
4. **At-least-once recovery with idempotency:** crash recovery may repeat work around uncertain external side effects. Tool contracts therefore define idempotency behavior and reconciliation where possible. Do not promise exactly-once execution for arbitrary external systems.
5. **Least privilege:** tools receive scoped capabilities, not ambient authority. Credentials are brokered and never placed in model context unless explicitly required and safe.
6. **Untrusted context:** user content, tool output, retrieved memory, documents, and model output are data. None may override system policy or grant permissions.
7. **Provider neutrality:** provider-specific request/response types remain inside adapters. Core domain types are stable and provider-independent.
8. **Independent verification:** an agent's statement that it succeeded is a claim, not proof.
9. **Observable operation:** important decisions, policy outcomes, state transitions, tool calls, approvals, and failures are represented as structured events.
10. **Fail closed:** if authorization, budget, approval, or required policy state cannot be determined, a privileged action is not executed.

## 6. Logical architecture

The system is divided into five logical layers.

### 6.1 Interfaces

- CLI for local operation and administration.
- HTTP API for remote clients and service integration.
- Rust SDK and, later, language-neutral/other-language SDKs.
- MCP server as an interface adapter, not a privileged bypass.
- Web dashboard for run inspection and approvals.

Interfaces authenticate callers and submit commands. They do not implement the authoritative policy decision themselves.

### 6.2 Application services

- Identity and authorization service.
- Agent registry and version service.
- Run orchestration service.
- Approval service.
- Scheduler and worker coordination.
- Event/query service.
- Administrative and configuration service.

Application services coordinate use cases and transactions; they do not own provider-specific logic.

### 6.3 Runtime domain

- Agent run state machine.
- Execution loop.
- Planner/action protocol.
- Context assembly.
- Budget and usage accounting.
- Policy evaluation.
- Tool-call lifecycle.
- Checkpoint and recovery coordination.
- Delegation lifecycle.

This layer owns the invariants.

### 6.4 Ports (provider-independent contracts)

- `ModelProvider`
- `Tool`
- `ToolExecutor`
- `PolicyEngine`
- `ApprovalGateway`
- `MemoryStore`
- `RunStore`
- `EventStore`
- `ArtifactStore`
- `SecretBroker`
- `ExecutionBackend`
- `Clock` / `IdGenerator` where deterministic testing requires them.

### 6.5 Adapters

Initial and future adapters may include:

- OpenAI, Anthropic, Gemini, and local model providers;
- native Rust tools, MCP tools, and explicitly registered plugin tools;
- SQLite initially, PostgreSQL later;
- local process execution for trusted workloads, then container/VM/remote isolated execution;
- filesystem or object storage for artifacts;
- OS keyring or external secret manager integrations.

Adapters may fail, time out, rate-limit, or return malformed/untrusted data. The runtime must handle these outcomes without relaxing policy.

## 7. Runtime execution model

### 7.1 Run creation

A run is created with:

- unique `run_id`;
- immutable `agent_id` and `agent_version`;
- caller/tenant identity and authorization context;
- task input reference or bounded input payload;
- policy profile and capability grants;
- budget limits;
- provider/model routing constraints;
- creation timestamp and correlation/trace identifiers.

The run stores a reference to the exact agent definition version and relevant configuration snapshot. Updating an agent definition MUST NOT silently mutate an already-running run.

### 7.2 Execution loop

The canonical loop is:

1. Load run state, agent version, policy profile, and remaining budgets.
2. Assemble bounded context from system policy, agent instructions, task input, conversation state, and selected memory.
3. Call the selected model provider through the provider port.
4. Validate and normalize the provider response.
5. If the response is a tool/action request, validate its schema and resolve the registered tool.
6. Evaluate authorization, capability scope, risk, budgets, rate limits, and approval requirements.
7. If approval is required, persist a pending-approval state and suspend execution.
8. If approved, dispatch through the selected executor with scoped credentials/capabilities and an idempotency key where supported.
9. Persist tool result metadata, bounded output, artifact references, and structured events.
10. Update conversation/run state and memory only through configured write policy.
11. Continue the loop, request another model response, or enter finalization.
12. Persist final outcome claim and terminate the run.

The model never directly invokes a tool. It emits a structured proposal that passes through validation and policy enforcement.

### 7.3 Run lifecycle state machine

The initial domain should support these states:

- `created`
- `queued`
- `running`
- `waiting_for_approval`
- `waiting_for_input`
- `paused`
- `completed`
- `failed`
- `cancelled`
- `timed_out`

Expected transitions include:

- `created → queued`
- `queued → running`
- `running → waiting_for_approval`
- `waiting_for_approval → running` after approval
- `waiting_for_approval → cancelled` or `failed` after denial/expiry according to policy
- `running → waiting_for_input → running`
- `running → paused → running`
- `running → completed | failed | cancelled | timed_out`

Terminal states are `completed`, `failed`, `cancelled`, and `timed_out`. A retry SHOULD create a distinct attempt identity and preserve the history of the prior attempt. Whether a retry remains under the same logical run or creates a new run is a domain/API decision to finalize before implementation; it must never erase prior attempt history.

Every transition MUST include actor, timestamp, reason, correlation ID, and expected prior version/sequence to prevent stale concurrent updates.

### 7.4 Tool execution lifecycle

A tool call has its own stable `tool_call_id` and lifecycle, at minimum:

- proposed;
- validated;
- denied / approval_pending / authorized;
- dispatched;
- succeeded / failed / timed_out / indeterminate;
- reconciled (where an external side effect's outcome was initially uncertain).

The runtime records the exact tool name/version, normalized argument digest, policy decision, approval reference, executor identity, idempotency key, timing, result digest, and artifact references. Sensitive argument/result content is stored separately with access control and retention policy.

For a timeout after dispatch, the runtime MUST NOT assume the side effect did not occur. It should reconcile using the tool's idempotency or status-query contract where available; otherwise mark the result indeterminate and require policy-defined handling.

## 8. Policy, permissions, and approvals

### 8.1 Policy decision inputs

A policy decision may consider:

- caller identity and tenant;
- agent identity/version;
- tool identity/version and declared risk;
- requested capability and resource scope;
- argument classification and validated schema;
- execution environment;
- network/filesystem/process permissions;
- budget remaining;
- rate limits;
- approval rules;
- current run state and policy version.

### 8.2 Decision outcomes

The policy engine returns a typed decision such as:

- `allow`;
- `deny` with machine-readable reason;
- `require_approval` with required approver scope and expiry;
- `allow_with_constraints` with narrowed capabilities/limits.

No adapter may convert a deny or unresolved policy result into allow.

### 8.3 Capability model

Capabilities are explicit, scoped, and time-bounded where practical. Examples include read access to a specific workspace path, access to a named API, or permission to invoke a specific tool with constrained parameters. Avoid broad ambient host access.

Secrets are resolved by a secret broker at dispatch time. Secret values MUST NOT be written to model context, ordinary logs, event payloads, or unredacted artifacts.

### 8.4 Human approval

Approval requests are durable objects bound to the exact proposed action, normalized argument digest, run, tool version, policy version, and expiry. Any material change to the action invalidates the approval and requires a new decision. Approval is not a general grant of future authority.

## 9. Budgets and usage accounting

Budgets are enforced at the boundary where usage occurs, not inferred from model claims. Initial dimensions:

- provider input/output tokens (where provider reports them);
- model requests;
- tool calls;
- wall-clock duration;
- executor CPU, memory, process count, and output bytes where backend supports enforcement;
- estimated monetary cost;
- delegated child-run limits.

Accounting records source, measurement type, unit, amount, timestamp, and confidence/measurement status. Unknown usage is recorded as unknown, never silently as zero. Reservations may prevent concurrent overspend; settlement reconciles reservations against actual usage. Hard limits must be checked before dispatch when possible and after execution where only measured afterward.

## 10. Memory and context

Memory is a pluggable data service, not an authority service.

Separate at least:

- run-local conversation/state;
- agent-scoped durable memory;
- user/tenant-scoped memory;
- retrieved external knowledge/documents.

Each memory item should carry provenance, scope, creation/update time, content type, trust classification, and retention metadata. Retrieval is bounded and relevance-ranked, but relevance does not imply trust. Memory writes require explicit policy and should support provenance and deletion/retention requirements.

Context assembly applies deterministic precedence: runtime/system policy cannot be overridden by agent prompts, user input, retrieved memory, or tool output. The exact prompt serialization is provider-adapter territory; policy precedence is a runtime invariant.

## 11. Persistence, events, checkpoints, and recovery

### 11.1 Initial storage

Use SQLite for the first local-first implementation, behind repository/store ports. Keep schema migrations explicit and versioned. Avoid leaking SQLite-specific types into the runtime domain.

PostgreSQL is a future adapter, not a reason to prematurely design every query as a distributed transaction.

### 11.2 Durable records

At minimum persist:

- agent definitions and immutable versions;
- runs and attempts;
- state transitions;
- model request/response metadata and bounded/redacted content references;
- tool calls and policy decisions;
- approvals;
- budget reservations and usage;
- checkpoints;
- event sequence;
- artifacts and content digests;
- scheduler/lease ownership if workers are introduced.

### 11.3 Event model

Events are append-only, versioned, and ordered per run. A common envelope includes:

- `event_id`, `event_type`, `schema_version`;
- `run_id`, optional `attempt_id`, optional `tool_call_id`;
- sequence number;
- timestamp;
- actor/component;
- correlation/causation IDs;
- payload or protected payload reference;
- classification/redaction metadata.

Do not place secrets or unrestricted raw payloads in the event stream. Store sensitive content in separately protected records or artifact storage and reference it. Events should be sufficient to understand decisions and reconstruct operational history, while avoiding an unbounded event store of raw prompts and outputs.

### 11.4 Checkpoint and recovery

A checkpoint captures the minimum durable state required to resume safely: run state/version, conversation/context references, pending action state, budget reservations, and relevant adapter reconciliation tokens.

On restart, the runtime loads nonterminal runs and reconciles in-flight operations before resuming. It must distinguish:
- definitely not dispatched;
- dispatched with known result;
- dispatched with unknown result;
- awaiting approval;
- safe to retry under idempotency contract.

Never blindly replay a possibly completed external side effect.

### 11.5 Concurrency and fencing

Every state update uses optimistic versioning or an equivalent compare-and-swap invariant. Distributed workers, when introduced, require leases with monotonically increasing fencing tokens. Lease expiry alone is not proof that an old worker stopped; stale workers must be prevented from committing state or performing unguarded side effects. Long-running workers require heartbeat/renewal and explicit ownership checks.

## 12. Execution environments and isolation

Execution backends are pluggable and selected by workload trust/risk.

- **In-process/native tool:** only for trusted, reviewed code with narrow capabilities.
- **Local subprocess:** convenience for trusted local workflows; not a hostile-code security boundary by itself.
- **Container:** stronger isolation when correctly configured, but shares a kernel and requires hardened configuration.
- **Disposable VM or dedicated remote executor:** preferred direction for hostile or untrusted code.

An execution backend contract must declare and enforce supported controls: filesystem mounts, network policy, credentials, CPU/memory/PID limits, wall-clock timeout, output byte caps, and cleanup guarantees. Unsupported controls must be explicit. Do not claim a security property the selected backend cannot enforce.

## 13. Provider abstraction and routing

The `ModelProvider` port should normalize:

- model invocation request;
- messages/content parts;
- tool schemas and tool-call proposals;
- streaming events (optional in first release);
- token/usage metadata;
- provider errors and retryability;
- finish reason and response metadata.

Provider-specific extensions may be preserved as opaque metadata but must not become required by core runtime logic. Provider routing/fallback is a later capability. Fallback must preserve policy, context constraints, budget accounting, and trace continuity; it must not silently change model trust assumptions.

## 14. Agent definitions and versioning

An agent definition is a declarative, validated configuration containing:

- stable `agent_id`;
- immutable version;
- instructions/system behavior;
- allowed provider/model constraints;
- tool allowlist and capability requests;
- memory policy;
- budget defaults/ceilings;
- approval policy;
- execution profile;
- input/output schemas;
- metadata and provenance.

Definitions are versioned and immutable once used by a run. Updating an agent creates a new version. A run pins its version and policy snapshot for reproducibility. Loading untrusted agent bundles/plugins requires signature/trust policy and must not grant capabilities by declaration alone.

## 15. Multi-agent orchestration

Multi-agent delegation is a later phase, but the domain should reserve a first-class parent/child relationship.

A parent run delegates a bounded task to a child run with:
- explicit purpose and input;
- child agent/version;
- restricted capability subset;
- independent budget ceiling;
- deadline and cancellation propagation policy;
- bounded result contract;
- parent/child trace linkage.

A child cannot inherit more authority than its parent possesses. Child output is untrusted input to the parent. Parent completion policy must account for child failures, timeouts, and partial results. Avoid shared mutable memory between agents by default.

## 16. Interfaces and protocol boundaries

### 16.1 CLI

The CLI is an operator/client interface, not the runtime itself. Initial commands should focus on configuration validation, agent inspection, run creation/inspection/cancellation, and local service operation.

### 16.2 HTTP API

The API will expose authenticated application commands and queries. Authorization must be enforced in application services for every request. API gateway rules or dashboard visibility are not substitutes.

### 16.3 MCP

MCP is an adapter for exposing approved Conductor capabilities to external clients. It must use the same identity, authorization, policy, budget, and audit services as other interfaces. Tool descriptions are not security controls.

### 16.4 SDKs

SDKs wrap stable public contracts and should not duplicate policy logic. Versioning and compatibility rules must be explicit before public release.

## 17. Security and threat model

Assume:
- model output may be malicious, malformed, or manipulated by prompt injection;
- tool output and retrieved documents may contain adversarial instructions;
- plugins/tools may be buggy or compromised;
- external providers may fail or return malformed content;
- credentials may be exposed through logs, arguments, crash reports, or artifacts;
- workers may crash, stall, become partitioned, or continue after lease expiry;
- users may attempt cross-run or cross-tenant access;
- untrusted code may attempt filesystem, network, process, or resource abuse.

Required controls:
- authenticated identity and service-to-service authorization;
- tenant/run/resource scoping at data-access boundaries;
- least-privilege capabilities and secret brokering;
- strict input/output schemas and payload-size limits;
- log/event redaction and secret scanning;
- network egress and filesystem controls in execution backends;
- CPU, memory, PID, time, and output quotas where supported;
- dependency and plugin provenance controls;
- durable audit events for security-relevant decisions;
- safe cancellation, timeout, and cleanup semantics;
- no trust in model-provided claims about permissions, tool results, or success.

Security review and adversarial testing are release gates for any hosted or untrusted-code execution mode.

## 18. Observability and evidence

Every run should be inspectable through structured events and correlated traces. Core telemetry includes state transitions, model calls, tool dispatches, policy decisions, approvals, budget changes, retries, timeouts, and recovery decisions.

Artifacts are stored by content digest and referenced from events. The event stream should preserve provenance and integrity metadata. Redaction and access controls apply to both event payloads and artifacts.

Operational observability is not the same as independent verification. Conductor records what happened and what the agent claims; CyberCore Agent Evaluation determines whether evidence supports the requested outcome.

## 19. Proposed Rust workspace boundaries

The following is the **target logical decomposition**, not a requirement to create every crate in the first commit. Start with fewer cohesive crates and split when dependencies and ownership are clear.

| Crate | Responsibility | Must not own |
|---|---|---|
| `conductor-core` | IDs, domain types, errors, state machine, core contracts | Provider SDKs, database drivers |
| `conductor-runtime` | Execution loop, context assembly, action lifecycle, checkpoints | HTTP handlers, vendor-specific APIs |
| `conductor-policy` | Capability model, policy evaluation, budget/approval gates | UI-only authorization |
| `conductor-providers` | Model provider traits and adapters | Run state ownership |
| `conductor-tools` | Tool registry, schemas, dispatch contracts | Unchecked ambient host access |
| `conductor-memory` | Memory interfaces, provenance, retrieval/write policy | Security authority |
| `conductor-store` | Persistence adapters, migrations, event/artifact repositories | Domain policy decisions |
| `conductor-executor` | Execution backend interface and implementations | Model planning |
| `conductor-orchestrator` | Scheduling, workers, parent/child coordination | Provider-specific request formats |
| `conductor-cli` | Operator CLI | Runtime invariants |

A first implementation may combine `core`, `runtime`, `policy`, `store`, and `cli`, with provider/tool/executor adapters in separate modules. Crate boundaries should follow stable dependency direction, not a desire to maximize crate count.

Dependency direction: interfaces and adapters depend inward on domain contracts; the domain must not depend on concrete adapters. Avoid cyclic crate dependencies.

## 20. API and contract design requirements

Before implementation, define versioned schemas for:
- agent definition and version;
- run creation and run snapshot;
- state transition and event envelope;
- model request/response normalization;
- tool definition, proposal, result, and error;
- policy decision and approval request/result;
- budget reservation/usage record;
- checkpoint and recovery record;
- artifact reference;
- parent/child delegation.

Contracts must specify required fields, validation, compatibility behavior, idempotency, error taxonomy, and sensitivity classification. Use stable machine-readable identifiers; human-readable messages are supplemental.

## 21. Testing strategy and quality gates

Testing must be designed around invariants, not just happy-path model responses.

- Unit tests for state transitions, policy decisions, budget arithmetic, and schema validation.
- Property/state-machine tests for invalid transitions, replay, and concurrency races.
- Adapter contract tests for providers, tools, stores, and executors.
- Fault-injection tests for process crash, provider timeout, tool timeout, duplicate delivery, and partial persistence.
- Recovery tests proving in-flight side effects are reconciled rather than blindly repeated.
- Security tests for prompt injection, tool-output injection, authorization bypass, secret leakage, path traversal, and resource exhaustion.
- Integration tests for SQLite transactions, event ordering, checkpoint restore, and cancellation.
- Evaluation integration tests where Agent Evaluation independently checks outcomes and evidence.

No test should require a live paid model provider by default. Use deterministic fake providers and tools for core correctness. Live-provider tests are opt-in integration tests.

## 22. Phased delivery

### Phase 0 — Architecture and repository foundation
- confirm repository metadata and naming/license decisions;
- accept this specification or record amendments;
- define contract schemas and initial ADRs;
- establish Rust toolchain, workspace, CI, formatting, linting, and test policy.

### Phase 1 — Single-agent runtime foundation
- minimal Rust domain and state machine;
- one provider adapter plus deterministic fake provider;
- typed native tool interface;
- policy gate and bounded budgets;
- SQLite persistence and append-only event history;
- one-agent execution loop;
- CLI to create, inspect, and cancel runs.

### Phase 2 — Durable autonomy
- checkpoint and crash recovery;
- idempotency/reconciliation for tool side effects;
- approval checkpoints;
- memory interface and provenance;
- scheduler, cancellation, and worker ownership;
- hardened execution backend appropriate to supported workloads;
- complete usage accounting.

### Phase 3 — Extensible platform
- MCP adapter;
- plugin/skill registration and trust model;
- provider routing/fallback;
- agent definition version lifecycle;
- bounded multi-agent delegation;
- HTTP API and SDK contracts.

### Phase 4 — Distributed platform
- distributed workers and lease/fencing model;
- PostgreSQL and remote artifact storage;
- VM/remote execution;
- dashboard and operational administration;
- deployment/upgrade/backup/restore story;
- deeper CyberCore Agent Evaluation integration.

## 23. Open decisions

These decisions must be resolved before the affected implementation begins:

1. **Product name:** "CyberCore Conductor" remains provisional; perform legal, package, domain, and ecosystem clearance.
2. **License and contribution model:** select before public source release.
3. **Minimum supported Rust version and edition:** choose with the initial toolchain bootstrap.
4. **Run vs attempt semantics:** finalize whether retry creates a new logical run or a new attempt under the same run; preserve immutable history either way.
5. **Event payload strategy:** choose inline bounded payloads versus protected payload references and retention defaults.
6. **Initial provider:** select one provider adapter based on availability and testability; keep fake provider first-class.
7. **Initial executor trust boundary:** explicitly limit local executor to trusted workloads until stronger isolation is implemented and reviewed.
8. **Identity model:** define local single-user identity first, while keeping service authorization boundaries suitable for later multi-user operation.
9. **Memory scope and deletion semantics:** define user/tenant ownership, retention, and deletion before persistent user memory ships.
10. **Public API stability policy:** decide schema/version compatibility before SDK publication.

## 24. Acceptance criteria for v0.1

This specification is ready to guide implementation when reviewers agree that:

- product scope and non-goals are clear;
- Conductor and Agent Evaluation have independent responsibilities;
- model proposals cannot bypass runtime policy;
- run and tool-call lifecycle semantics are explicit;
- durability, recovery, idempotency, and fencing concerns are acknowledged;
- memory is treated as untrusted data;
- budgets and approvals are enforced at runtime boundaries;
- execution isolation claims match backend capabilities;
- crate boundaries and dependency direction are understandable;
- phased delivery is realistic and does not force premature distributed architecture;
- open decisions are visible rather than silently assumed.

**v0.1 does not authorize implementation of every planned capability.** It establishes the invariants and direction. Implementation work should begin with Phase 0 and the smallest vertical slice in Phase 1.
