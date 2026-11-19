# CyberCore Conductor — Architecture Specification v0.2

**Status:** Consolidated architecture baseline — approved direction; implementation not yet started
**Date:** 2026-11-18
**Supersedes:** Architecture Specification v0.1 as the current architecture baseline
**Working name:** CyberCore Conductor (provisional; naming clearance outstanding)
**Repository:** `cybercoretech/cybercore-conductor`

Normative terms **MUST**, **MUST NOT**, **SHOULD**, and **MAY** express requirements within this specification.

---

## 1. Executive Summary

CyberCore Conductor is a provider-independent autonomous agent runtime and orchestration platform. It executes agent work as durable, observable, policy-governed runs.

Conductor owns the control loop around model calls, planning, tool proposals, authorization, approvals, budgets, memory access, execution environments, persistence, recovery, and eventual delegation.

Conductor is not simply a command-line wrapper around an LLM. It is also not the CyberCore Agent Evaluation Workbench.

Its foundational authority rule is:

> **A model may propose an action; only the runtime may authorize and execute it.**

Models, retrieved context, tools, and external providers are untrusted decision-support or data sources. None may grant permissions, expand budgets, bypass approvals, select arbitrary credentials, or change runtime policy.

The initial implementation is local-first and deliberately narrow. It establishes a reliable single-agent execution path, explicit authorization, durable state, recoverable tool operations, bounded resource use, and auditable evidence before introducing distributed workers or multi-agent orchestration.

## 2. Goals

Conductor MUST provide architectural foundations for:

1. Provider-independent agent execution.
2. Explicit agent definitions, immutable versions, and resolved configuration manifests.
3. Planning and iterative model/tool interaction.
4. Typed tool invocation and runtime-enforced authorization.
5. Durable runs, attempts, checkpoints, cancellation, and crash recovery.
6. Context assembly and pluggable memory.
7. Atomic budget reservations and usage accounting.
8. Human approval checkpoints for sensitive actions.
9. Structured events, traces, artifacts, and audit history.
10. Pluggable execution environments with isolation appropriate to workload risk.
11. Bounded orchestration and eventual multi-agent delegation.
12. Independent integration with CyberCore Agent Evaluation.

## 3. Non-goals for the Initial Release

The first release does not require:

- Hosted SaaS or a multi-tenant cloud control plane.
- A browser dashboard or visual workflow designer.
- Every model provider or tool protocol.
- Unrestricted autonomous shell or browser access.
- Hostile-code isolation through ordinary in-process or local-process execution.
- A distributed scheduler or cluster consensus system.
- A built-in vector database.
- Unrestricted plugin execution.
- Automatic cross-provider fallback.
- Multi-agent delegation.
- A claim that an agent's own final answer proves task success.

These capabilities may be introduced in later phases without weakening the core authority, durability, and evidence invariants.

## 4. System Boundaries

### 4.1 Conductor owns

- Agent definition registration and immutable version references.
- Run and attempt creation and lifecycle.
- Planning loop and model interaction orchestration.
- Context construction and memory retrieval/write policy.
- Tool catalog, validation, authorization, and dispatch.
- Budgets, usage, rate limits, and approval gates.
- Durable state, checkpoints, events, and recovery.
- Worker and scheduler coordination when introduced.
- Delegation and parent/child relationships when introduced.
- Provider, tool, storage, and executor adapter contracts.
- Evidence export for independent evaluation.

### 4.2 Conductor does not own

- Independent certification of its own task success.
- The truth or trustworthiness of model output.
- Unrestricted access to user credentials or host resources.
- The internal semantics of every external provider or tool.
- Evaluation policy that must remain portable across runtimes.

### 4.3 CyberCore ecosystem boundaries

**CyberCore Agent Evaluation** remains a separate, runtime-independent evaluation system. Conductor exports versioned evidence through an explicit contract. Evaluation independently determines whether the evidence supports a claim and returns separately attributable findings.

Conductor MAY store and display evaluation results. It MUST NOT modify, impersonate, or self-certify those results.

**AgentForge** remains a separate coding-agent orchestration project. Integration, if warranted, uses documented APIs, events, or dedicated adapters. AgentForge's orchestration domain MUST NOT be merged into Conductor's runtime core.

## 5. Architectural Principles and Invariants

1. **Runtime authority:** Prompts and model instructions are not security controls. Every privileged action is authorized by runtime policy.
2. **Durability:** Runs and attempts have stable identities and persisted lifecycle state.
3. **Explicit state machines:** Invalid transitions are rejected. Prior attempts and their evidence are preserved.
4. **At-least-once recovery:** Recovery may encounter uncertain external side effects. Exactly-once execution is not promised for arbitrary external systems.
5. **Least privilege:** Tools receive runtime-issued scoped capabilities, not ambient authority.
6. **Untrusted context:** User content, model output, retrieved memory, documents, and tool output are data, not policy.
7. **Provider neutrality:** Vendor-specific request and response types remain within adapters.
8. **Independent verification:** An agent's statement of success is a claim, not proof.
9. **Observable operation:** Important state changes, decisions, dispatches, approvals, and failures produce structured events.
10. **Fail closed:** If required authorization, budget, approval, or policy state cannot be established, the privileged action is not executed.
11. **Bounded execution:** Every attempt has mandatory resource and execution limits.
12. **Reproducibility:** Runs pin immutable agent configuration and dependency versions.
13. **Conservative uncertainty:** Unknown usage and uncertain external outcomes are represented explicitly, never silently treated as zero or success.

## 6. Logical Architecture

The system has five logical layers.

### 6.1 Interfaces

- CLI.
- HTTP API.
- Rust SDK and future language-neutral SDKs.
- MCP interface adapter.
- Eventual web dashboard.

Interfaces authenticate callers and submit commands. They do not independently implement authoritative policy.

### 6.2 Application services

- Identity and authorization.
- Agent registry and version service.
- Run orchestration.
- Approval service.
- Scheduler and worker coordination.
- Event/query service.
- Administration and configuration.

### 6.3 Runtime domain

- Run and attempt state machines.
- Execution loop.
- Planner/action protocol.
- Context assembly.
- Budget and usage accounting.
- Policy evaluation.
- Tool-call lifecycle.
- Checkpoint and recovery coordination.
- Delegation lifecycle.

The runtime domain owns the invariants.

### 6.4 Provider-independent ports

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
- `Clock`
- `IdGenerator`

### 6.5 Adapters

Potential adapters include model providers, native Rust tools, MCP tools, SQLite, PostgreSQL, local trusted-process execution, containers, VMs, remote execution, filesystem/object storage, and external secret managers.

Adapters may fail, time out, return malformed content, or report incomplete usage. Adapter behavior MUST NOT relax runtime policy.

## 7. Run and Attempt Model

### 7.1 Logical run

A `Run` represents the logical unit of work requested by a caller. It owns the overall task identity, caller context, agent version, policy context, and aggregate lifecycle.

### 7.2 Attempt

An `Attempt` represents one concrete execution of a run.

A run MAY contain multiple attempts. Each Attempt has an independent identity, execution history, usage attribution, and terminal outcome. Resource usage is attributed to individual Attempts but accounted against the authoritative aggregate budgets of the parent Run.

A retry creates a new attempt under the same logical run. It MUST NOT overwrite or erase the prior attempt.

The run's aggregate status is derived from its attempts and the applicable retry policy. Exact aggregation rules belong in the run contract.

### 7.3 Run creation

A run records:

- Unique `run_id`.
- Immutable `agent_id` and `agent_version`.
- Caller identity and authorization context.
- Task input reference or bounded payload.
- Policy profile and capability constraints.
- Budget limits.
- Provider/model routing constraints.
- Resolved configuration manifest digest.
- Creation time and correlation/trace identifiers.

A run pins its resolved agent configuration. Updating an agent definition MUST NOT silently change an existing run.

## 8. Execution Loop and Tool Lifecycle

### 8.1 Canonical execution loop

1. Load authoritative run and attempt state.
2. Verify execution ownership and current policy.
3. Load the pinned agent manifest and remaining budgets.
4. Assemble bounded context from policy, instructions, task input, conversation state, and selected memory.
5. Invoke the selected provider through `ModelProvider`.
6. Validate and normalize the provider response.
7. If an action is proposed, validate its schema and resolve the registered tool/version.
8. Evaluate authorization, capabilities, risk, budget, rate limits, and approval requirements.
9. If approval is required, persist a pending approval and suspend execution.
10. Revalidate authorization at dispatch time.
11. Persist a durable dispatch intent before external dispatch.
12. Dispatch through the selected executor with scoped credentials and idempotency information where supported.
13. Persist result metadata, usage, artifacts, and events.
14. Update conversation and run state; apply memory write policy.
15. Continue, finalize, or enter a waiting/terminal state.

A model never directly invokes a tool.

### 8.2 Run lifecycle

Initial states:

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

Terminal states are `completed`, `failed`, `cancelled`, and `timed_out`.

Every authoritative transition MUST validate the expected prior state/version and record its actor, reason, timestamp, and correlation information.

### 8.3 Tool-call lifecycle

Each tool call has a stable identity and a lifecycle including:

- `proposed`
- `validated`
- `denied`
- `approval_pending`
- `authorized`
- `dispatched`
- `succeeded`
- `failed`
- `timed_out`
- `indeterminate`
- `reconciled`

A tool-call record captures tool name/version, normalized argument digest, policy decision, approval reference, executor identity, idempotency key, dispatch state, timing, result digest, and artifact references.

A timeout after dispatch does not prove that the operation failed. The runtime MUST reconcile when possible and otherwise preserve an indeterminate outcome.

### 8.4 Durable dispatch intent

An external side effect MUST NOT be dispatched until a durable authorized dispatch intent exists.

The intent identifies the run, attempt, tool call, normalized argument digest, authorization decision, approval reference, budget reservation references, idempotency key, and executor identity.

Dispatch authorization MUST require a durable active budget reservation bound to the exact run, attempt, tool call, operation, and applicable resource dimensions. A reservation MUST NOT be reused by another operation.

The transaction authorizing dispatch MUST verify the expected run state/version, cancellation state, current policy and capability validity, required approval validity and consumption, and active budget reservations before creating the durable dispatch intent. Approval consumption, reservation binding, and dispatch-intent creation MUST be coordinated atomically with the authoritative state transition. If any prerequisite fails, the transaction MUST NOT authorize dispatch.

The executor may perform the external operation only after the dispatch-intent transaction commits. A committed dispatch intent establishes the operation as authorized and in flight for cancellation and recovery semantics.

Recovery examines incomplete dispatches before allowing new work. It reconciles existing operations before retrying them.

### 8.5 Tool retry contract

Each tool declares one of these operation classes:

- `idempotent`
- `idempotency_keyed`
- `reconcilable`
- `non_reconcilable`

The runtime enforces retry behavior based on that contract. An uncertain non-reconcilable operation MUST NOT be blindly retried.

## 9. Policy, Capabilities, and Approvals

### 9.1 Policy inputs

Policy may consider caller, tenant, agent/version, run/attempt, tool/version, operation, resource scope, argument classification, execution environment, network/filesystem/process permissions, budget, rate limits, approval rules, and current run state.

### 9.2 Policy outcomes

Typed outcomes include:

- `allow`
- `deny`
- `require_approval`
- `allow_with_constraints`

An unresolved or failed policy decision MUST NOT become an allow.

### 9.3 Capability grants

Capabilities are runtime-issued scoped grants. A grant identifies:

- Subject (run, attempt, agent, or child run).
- Resource.
- Permitted operation.
- Scope restrictions.
- Lifetime and expiry.
- Policy provenance.
- Revocation state.

Tools cannot expand grants through their declarations or arguments.

Filesystem authorization must account for path traversal, symlink resolution, and time-of-check/time-of-use risks.

### 9.4 Approval semantics

An approval is bound to the exact action digest, run, tool version, policy version, approver scope, and expiry.

Approval is necessary when policy requires it, but is never sufficient by itself to authorize dispatch.

At dispatch time, the runtime revalidates current run eligibility, policy, capabilities, budgets, resource limits, action digest, expiry, and one-time use. Approval consumption and durable dispatch-intent creation are coordinated atomically.

**Phase 1 boundary:** The approval workflow is not yet implemented. If policy returns `require_approval`, the runtime MUST return an explicit blocked result such as `approval_unavailable`; it MUST NOT dispatch the action, downgrade the decision to `allow`, or treat missing approval infrastructure as approval. This restriction applies even when the tool is otherwise available.

**Phase 2 capability:** Persisted approval requests, suspension and resumption, approver identity/scope validation, expiry, one-time consumption, and dispatch-time revalidation are implemented. The atomic authorization requirements in Section 8.4 remain mandatory.

A material action change invalidates the approval.

### 9.5 Cancellation

Cancellation is a durable cooperative control request, not a rollback guarantee.

Cancellation acceptance and dispatch authorization MUST be serialized against the same authoritative run state/version. The operation that commits first establishes the ordering:

- If cancellation commits first, subsequent dispatch authorization MUST fail.
- If dispatch authorization commits first, its durable dispatch intent establishes that operation as in flight for cancellation purposes, even if the external request has not yet physically left the process.
- Cancellation MUST NOT claim to have prevented an operation whose dispatch intent was already committed.
- After cancellation is accepted, no additional dispatch intent may be authorized for that run. Already-authorized operations are handled through cancellation propagation where supported and outcome reconciliation.

The runtime MUST prevent new work after cancellation is accepted, propagate cancellation where supported, and reconcile already-dispatched operations.

It distinguishes cancellation before dispatch, during execution, after known success, and while an external outcome remains uncertain.

## 10. Execution Bounds and Budgets

### 10.1 Run-level budget ownership

The logical Run is the authoritative aggregate budget boundary.

All execution budgets and ceilings apply to the Run as a whole and aggregate across every Attempt belonging to that Run. Creating a retry or a new Attempt MUST NOT reset, replenish, or independently expand any Run-level budget.

The Run-level account is authoritative for available, reserved, and consumed capacity. Every model request and external operation MUST be accounted for against that aggregate.

### 10.2 Immutable Attempt limit assignment

Every Attempt MUST receive an immutable execution-limit assignment when it is created.

The assignment captures the remaining Run-level allocation available to that Attempt at creation. It is a snapshot of the authoritative Run budget, not a new allocation and not an independent replenishment of capacity.

The assignment MUST include the Run's absolute deadline. An Attempt deadline MUST NOT extend the Run deadline or increase the remaining Run-level allowance.

Recovery MUST retrieve and verify the exact assignment associated with the Attempt. Changes to mutable defaults or later configuration MUST NOT silently alter an existing Attempt's assignment.

### 10.3 Phase 1 default Run profile

The following defaults apply to one logical Run and aggregate across all its Attempts.

| Dimension | Phase 1 default |
|---|---:|
| Total execution steps | 100 per Run |
| Total Attempts | 3, including the initial Attempt |
| Retries | 2 |
| Model requests | 20 per Run |
| Aggregate model input | 100,000 tokens per Run |
| Aggregate model output | 20,000 tokens per Run |
| Input per model request | 16,000 tokens |
| Output per model request | 4,000 tokens |
| Provider/model spend | USD 1.00 per Run |
| Tool calls / dispatches | 40 per Run |
| Concurrent external operations | 1 per Run |
| Failed execution operations | 5 per Run |
| Total Run elapsed time | 15 minutes |
| Aggregate tool execution time | 5 minutes per Run |
| Individual tool operation timeout | 60 seconds |

Payload and serialized-content ceilings:

| Payload | Phase 1 maximum |
|---|---:|
| Serialized model request | 256 KiB |
| Serialized model response | 64 KiB |
| Tool arguments | 64 KiB per call |
| Tool result | 64 KiB per call |
| Aggregate tool results | 1 MiB per Run |

KiB means 1,024 bytes. Token limits and byte-size limits are separate dimensions and MUST both be enforced.

The three-Attempt ceiling includes the initial Attempt. The two-retry ceiling is therefore not additive to three Attempts.

There are no independent per-Attempt aggregate budgets that replenish Run capacity. Per-request, per-operation, and immutable Attempt-assignment constraints remain applicable.

### 10.4 Atomic reservations and enforcement

Budget reservations MUST be atomic under concurrent operations.

Each reservation identifies the Run, Attempt, resource dimension, amount, state, creation/expiry, and exact associated operation or tool-call identity.

The reservation transaction verifies available capacity and updates reserved totals atomically. Concurrent reservations MUST serialize or use equivalent concurrency control so that the same available capacity cannot be allocated more than once.

Dispatch authorization MUST reference an active reservation for the exact operation. Reservation validation and binding and durable dispatch-intent creation are part of the same atomic authorization transaction described in Section 8.4. An operation MUST NOT be dispatched without this durable reservation.

A request that would exceed any hard ceiling MUST be rejected before dispatch. Reservation and authorization MUST fail closed.

Model-request capacity is consumed when durable dispatch authorization and reservation are committed, not only when a successful response arrives. Tool-call capacity is consumed when the durable dispatch intent is committed.

Retries consume additional Run-level capacity. A retry MUST NOT receive fresh Run budgets. An indeterminate external operation MUST be reconciled before a retry can be authorized.

The Run clock starts when execution starts and includes waiting and recovery time. Aggregate tool execution time accumulates across operations. The individual tool timeout does not replace the aggregate tool-time ceiling.

Failed execution operations count toward the failure ceiling. Policy denials and user-requested cancellation do not count as failed execution operations.

The maximum number of concurrent external operations is one. Concurrency control MUST prevent a second external operation from being authorized while the first remains active or unresolved.

Attempt limits assigned at creation are immutable. A provider response, model output, tool argument, or retry request MUST NOT expand an assignment or the Run budget.

Exceeding the defaults requires an explicitly configured and validated alternative profile. A model request or tool argument MUST NOT implicitly escalate limits.

Reservations are settled, released, or marked uncertain. Expiry MUST NOT automatically release a reservation if the associated external operation may still be running or its outcome is unresolved. Uncertain reservations remain charged against available capacity until reconciliation or an explicit, auditable resolution establishes the appropriate settlement.

### 10.5 Cost accounting

Every provider adapter handling paid model requests MUST use a versioned pricing schedule associated with the applicable provider and model.

Before dispatch, the runtime MUST reserve the maximum estimated cost of the request, including its permitted output. The estimate MUST use conservative token estimates and the configured provider rates.

On completion, reported or measured usage settles the reservation. If usage cannot be established, the reservation remains charged against available budget until reconciliation or an audited adjustment resolves it.

A paid model request MUST be rejected when its applicable pricing is unknown. Unknown pricing MUST NOT be treated as zero.

A local provider may use an explicitly configured zero-monetary-cost policy. Such a policy does not remove token, time, execution, payload, or other resource limits.

The USD 1.00 ceiling is a runtime accounting limit based on configured rates. It is not a guarantee that the provider's eventual invoice will exactly match the runtime estimate.

### 10.6 Usage provenance

Usage records distinguish:

- Measured usage.
- Provider-reported usage.
- Estimated usage.
- Unknown usage.
- Reconciled usage.

Records preserve source, unit, amount, timestamp, and confidence/measurement status.

Unknown usage is not zero. Estimated monetary cost is not confirmed billing.

Where actual usage is unavailable before dispatch, conservative reservations are used. Unknown usage MUST retain its reservation or be conservatively accounted for until reconciled.

## 11. Provider Abstraction and Routing

### 11.1 Provider capability descriptor

Providers expose an explicit capability descriptor including:

- Provider and model identity.
- Supported input content types.
- Context window information, when known.
- Structured-output support.
- Tool-call support.
- Streaming support.
- Usage reporting behavior.
- Cancellation behavior.
- Known request/response constraints.

Unknown capabilities MUST NOT be treated as supported.

The runtime selects only providers compatible with the agent's declared requirements and applicable policy.

### 11.2 Normalized provider contract

The `ModelProvider` port normalizes model invocation, messages/content parts, tool schemas and proposals, optional streaming events, usage metadata, finish reasons, and provider errors/retryability.

Provider-specific extensions MAY be preserved as opaque metadata but MUST NOT become required core-domain dependencies.

### 11.3 Fallback and routing

Provider fallback is a runtime routing decision, not a transparent adapter substitution.

Fallback must preserve run/attempt identity, policy profile, authorization rules, remaining budget, context restrictions, data classification, and trace continuity.

A provider change generates an explicit event recording original and selected provider/model, reason, and routing policy.

Fallback MUST NOT silently cross data-handling or trust boundaries.

## 12. Agent Definitions and Immutable Manifests

An agent definition contains:

- Stable `agent_id`.
- Immutable version.
- Instructions and configuration.
- Input/output schemas.
- Provider/model constraints.
- Tool definitions and pinned versions.
- Capability requests.
- Policy profile.
- Memory configuration.
- Execution profile.
- Budget defaults and ceilings.
- Approval policy.
- Metadata and provenance.

An immutable agent version captures or references immutable dependency versions. A resolved configuration manifest records the exact dependency set and content digests used by a run.

Secrets MUST NOT be embedded in agent definitions or manifests.

Changing a dependency MUST NOT silently mutate an existing version. Updating an agent creates a new version.

Capability declarations in agent bundles are requests, not grants.

## 13. Memory and Context

Memory is a pluggable data service, not an authority service.

Separate at least:

- Run-local conversation/state.
- Agent-scoped durable memory.
- User/tenant-scoped memory.
- Retrieved external knowledge/documents.

Memory items carry provenance, scope, timestamps, content type, trust classification, and retention metadata.

Retrieval is bounded and relevance-ranked. Relevance does not imply trust.

Memory writes require explicit policy. Persistent user memory requires defined ownership, retention, and deletion semantics.

Runtime/system policy has deterministic precedence over agent instructions, user input, retrieved memory, and tool output.

## 14. Persistence, Events, Checkpoints, and Recovery

### 14.1 Initial storage

Use SQLite for the initial local-first implementation behind store ports. PostgreSQL is a future adapter.

SQLite requirements include foreign-key enforcement, WAL where appropriate, configured transaction/busy timeouts, short write transactions, explicit schema migrations, and tested crash behavior.

No database transaction is held open across model calls or external tool execution.

### 14.2 Hybrid persistence model

Use a hybrid architecture:

- Relational/current-state records are authoritative for current state.
- Append-only events preserve history and audit evidence.
- Checkpoints accelerate safe recovery.
- Event history supports audit and independent evaluation.

Full event sourcing and mandatory full-stream replay are not required.

### 14.3 Atomic state and event writes

Every authoritative state transition and its corresponding event are committed in one database transaction.

The transaction verifies expected state/version, updates authoritative state, increments version, appends the event, and records its sequence. Failure rolls back the entire operation.

### 14.4 Event ordering and idempotency

Events have monotonic sequence numbers per run.

Enforce uniqueness for `(run_id, sequence_number)` and `event_id`; command/transition idempotency keys are enforced where applicable.

Timestamps are descriptive and MUST NOT be used as authoritative ordering.

Global ordering across unrelated runs is not required.

### 14.5 Checkpoint consistency

A checkpoint includes:

- Run and attempt IDs.
- Checkpoint ID and schema version.
- Associated run-state version.
- Last included event sequence.
- Conversation/context reference.
- Pending tool-call and dispatch states.
- Budget reservations.
- Relevant provider continuation metadata.
- Timestamp and integrity digest.

Checkpoint creation is transactionally consistent with its state version and event sequence.

A checkpoint MUST NOT imply that an uncertain external operation completed.

### 14.6 Recovery workflow

Recovery is a first-class runtime workflow with staged implementation requirements.

**Phase 1 minimum safe restart behavior:**

- Discover incomplete and nonterminal dispatch records during startup.
- Do not blindly redispatch an operation whose prior outcome is unknown.
- Preserve unresolved operations as indeterminate when their outcomes cannot be established.
- Retain associated budget reservations while outcomes remain uncertain.
- Block dependent execution and new model decisions while an unresolved operation could affect the next decision.
- Provide an explicit, auditable operator or reconciliation path for resolving outcomes that cannot be determined automatically.

Phase 1 does not require full checkpoint restoration or automated reconciliation across every external system. It MUST fail safely rather than assume that an interrupted operation did not happen.

**Phase 2 full recovery workflow:**

1. Discover nonterminal runs and attempts.
2. Acquire valid execution ownership.
3. Load authoritative state and latest valid checkpoint.
4. Identify incomplete dispatches.
5. Reconcile known or queryable external outcomes.
6. Mark unresolved outcomes indeterminate.
7. Restore budget reservations and mandatory limits.
8. Revalidate policy and capability validity.
9. Resume only safe operations.

No new model call or dispatch occurs while an unresolved operation could affect the next decision.

### 14.7 Worker ownership and fencing

Initial local execution still uses explicit ownership and state-version checks.

For distributed execution, leases require heartbeat/renewal and monotonically increasing fencing tokens.

State fencing protects internal writes. It does not automatically prevent a stale worker from calling an external API.

External side effects require separate dispatch authorization against current ownership immediately before dispatch. If the external service cannot validate a fencing token, idempotency and reconciliation are required.

Lease expiry does not prove the previous worker stopped.

### 14.8 Artifact integrity

Artifacts carry digest, media/content type, byte length, creation time, producer run/attempt/operation, sensitivity classification, storage reference, and retention metadata.

Reads verify integrity.

Artifact writes use staging, verification, and finalization. An authoritative event MUST NOT reference an artifact until its content is durable and verified. Unreferenced staged objects are garbage-collected after a safe interval.

### 14.9 Backup and restore

A production durability claim requires a documented and tested backup/restore procedure, including consistency across database state, event history, checkpoints, and artifact storage.

## 15. Execution Environments and Isolation

Execution backends are selected by workload trust and risk.

- **In-process/native tools:** trusted, reviewed code only.
- **Local subprocess:** trusted local workflows; not a hostile-code sandbox.
- **Container:** stronger isolation when correctly configured, but shares a kernel.
- **Disposable VM or dedicated remote executor:** preferred direction for hostile or untrusted code.

The backend declares and enforces supported filesystem, network, credential, CPU, memory, PID, timeout, output, and cleanup controls.

Unsupported controls are explicit. The initial release supports trusted workloads only. Arbitrary model-generated shell and untrusted executable plugins are disabled by default.

## 16. Multi-Agent Orchestration and Delegation

Multi-agent delegation is a later phase.

A parent delegates a bounded task to a child with explicit purpose, input, child agent/version, capability scope, budget ceiling, deadline, cancellation policy, and result contract.

### 16.1 Authority derivation

Effective child permissions are the intersection of:

1. Parent authority.
2. Parent delegation grant.
3. Child agent requirements.
4. Current runtime policy.

A child cannot enlarge this intersection.

Child grants specify tools, resource scopes, operations, data access, execution profile, budget ceiling, deadline, and cancellation policy.

### 16.2 Budget inheritance

Child budget allocations are reserved against the parent budget. Concurrent children cannot independently spend the same unreserved parent budget.

### 16.3 Mandatory delegation limits

Delegation is bounded by runtime-enforced limits for:

- Child runs per parent.
- Nesting depth.
- Concurrent children.
- Aggregate delegated budget.
- Child duration.
- Result size.

Parent behavior for child success, failure, timeout, cancellation, and partial results is explicit. Child completion does not automatically complete the parent.

Child output is untrusted input. Shared mutable memory is disabled by default.

## 17. Independent Evaluation Evidence Contract

Conductor and CyberCore Agent Evaluation communicate through a versioned, runtime-independent evidence contract.

Evidence export includes:

- Run and attempt identifiers.
- Agent version and resolved configuration digest.
- Event and trace references.
- Tool execution records.
- Policy decisions.
- Artifact references and content digests.
- Outcome claims.
- Provenance and sensitivity metadata.

Evaluation consumes evidence through an explicit adapter/export interface. It does not depend on Conductor's internal database schema.

Evaluation results include evaluation identity/version, evidence references examined, findings, confidence or verification status, and unresolved evidence gaps.

Conductor may persist and display results but cannot alter or self-certify them. Evaluation remains independently attributable.

## 18. Interfaces and Protocol Boundaries

### CLI

The CLI is an operator/client interface. Initial commands focus on configuration validation, agent inspection, run creation/inspection/cancellation, and local service operation.

### HTTP API

The API exposes authenticated application commands and queries. Application services enforce authorization for every request.

### MCP

MCP is an adapter for approved Conductor capabilities. It uses the same identity, authorization, policy, budget, and audit services as other interfaces.

### SDKs

SDKs wrap stable public contracts and do not duplicate policy logic. Compatibility rules are defined before public release.

## 19. Security and Threat Model

Assume models, tool output, retrieved documents, providers, plugins, and external workers may be malicious, malformed, compromised, or unreliable.

Assume credentials can leak through logs, arguments, crash reports, and artifacts unless explicitly protected.

Required controls include authenticated identity, resource scoping, least-privilege capabilities, secret brokering, strict schemas and size limits, redaction, secret scanning, execution isolation, resource quotas, dependency provenance, durable security events, safe cancellation, and adversarial testing.

Security review is a release gate for hosted execution and untrusted-code execution.

## 20. Observability and Evidence

Runs are inspectable through structured events and correlated traces.

Telemetry includes state transitions, model calls, provider routing, tool proposals and dispatches, policy decisions, approvals, budget changes, retries, timeouts, recovery decisions, and artifact references.

Operational observability records what happened and what the agent claims. It is not independent verification.

## 21. Rust Workspace Direction

Target logical decomposition:

| Crate | Responsibility |
|---|---|
| `conductor-core` | IDs, domain types, errors, state machine, core contracts |
| `conductor-runtime` | Execution loop, context, action lifecycle, checkpoints |
| `conductor-policy` | Capabilities, policy, budget and approval gates |
| `conductor-providers` | Provider traits and adapters |
| `conductor-tools` | Tool registry, schemas, dispatch contracts |
| `conductor-memory` | Memory interfaces, provenance, retrieval/write policy |
| `conductor-store` | Persistence adapters, migrations, events, artifacts |
| `conductor-executor` | Execution backend interfaces and implementations |
| `conductor-orchestrator` | Scheduling, workers, parent/child coordination |
| `conductor-cli` | Operator CLI |

This is a logical target, not a requirement to create every crate immediately. Start with fewer cohesive crates and split when dependency direction and ownership justify it.

Core domain contracts MUST NOT depend on concrete provider SDKs or database drivers. Adapters and interfaces depend inward on domain contracts. Avoid cyclic dependencies.

## 22. Contracts and Compatibility

Versioned contracts are required for:

- Agent definitions and resolved manifests.
- Run and attempt snapshots.
- State transitions and event envelopes.
- Provider capability descriptors and normalized requests/responses.
- Tool definitions, proposals, results, and retry contracts.
- Policy decisions and approvals.
- Capability grants.
- Budget reservations and usage records.
- Checkpoints and recovery records.
- Artifact references.
- Delegation grants and results.
- Evaluation evidence exports and findings.

Contracts specify required fields, validation, compatibility, idempotency, error taxonomy, and sensitivity classification.

## 23. Testing and Quality Gates

Testing is organized around invariants.

Required test categories include:

- Unit tests for state transitions, policy, capability derivation, budget arithmetic, and schemas.
- Property/state-machine tests for replay, invalid transitions, and concurrency.
- Adapter contract tests.
- Fault injection for crashes, timeouts, duplicate delivery, and partial persistence.
- Recovery tests for uncertain external side effects.
- Security tests for prompt injection, authorization bypass, secret leakage, path traversal, and resource exhaustion.
- SQLite transaction, event ordering, checkpoint, and cancellation tests.
- Approval lifecycle, expiry, one-time consumption, and fail-closed Phase 1 behavior tests.
- Artifact integrity and staged-write tests.
- Evaluation evidence contract tests.
- Delegation authority and aggregate budget tests before delegation ships.

Core tests use deterministic fake providers and tools. Live paid-provider tests are opt-in.

## 24. Phased Delivery

### Phase 0 — Architecture and repository foundation

- Accept and version this architecture baseline.
- Establish ADR index and contract schemas.
- Confirm naming/license/toolchain decisions.
- Establish Rust workspace conventions, CI, formatting, linting, and tests.

### Phase 1 — Single-agent runtime foundation

- Minimal domain and state machine with run/attempt semantics.
- Fake provider and one real provider adapter.
- Typed native tool interface.
- Policy and capability gate.
- Explicit fail-closed handling for `require_approval`; approval-required actions cannot be dispatched in Phase 1.
- Bounded budgets and attempt limits.
- SQLite state/event persistence.
- Atomic transition/event writes.
- Durable dispatch intent with atomic approval/budget/state coordination.
- Minimum safe restart handling for incomplete dispatches; no blind redispatch.
- Single-agent execution loop.
- CLI for run creation, inspection, and cancellation.

### Phase 2 — Durable autonomy

- Versioned checkpoints and full checkpoint-based crash recovery.
- Automated idempotency and reconciliation workflows for supported external operations.
- Persisted approval requests, checkpoints, suspension/resumption, and dispatch-time revalidation.
- Memory provenance and policy.
- Complete usage accounting.
- Hardened execution backend appropriate to supported workloads.
- Artifact integrity and backup/restore.

### Phase 3 — Extensible platform

- Provider capability negotiation and controlled routing.
- Immutable agent manifest lifecycle.
- MCP and plugin trust model.
- Versioned evidence export and Agent Evaluation integration.
- HTTP API and SDK contracts.
- Bounded multi-agent delegation after its implementation gates are met.

### Phase 4 — Distributed platform

- Distributed workers and dispatch fencing.
- PostgreSQL and remote artifact storage.
- VM/remote execution.
- Dashboard and administration.
- Deployment, upgrade, backup, and restore operations.

## 25. Open Decisions

The following remain explicit project decisions:

1. Product name and legal/package/domain clearance.
2. License and contribution model.
3. Rust edition and minimum supported Rust version.
4. Initial real provider selection.
5. Initial local identity/authentication model.
6. Event payload retention and protected-payload strategy.
7. Memory ownership, retention, and deletion policy.
8. Public API and schema compatibility policy.
9. Provider data-handling policy and allowed fallback boundaries.

These decisions must be recorded before the affected functionality ships. They do not weaken the accepted architectural invariants.

## 26. Acceptance Criteria

This specification is the architecture baseline when:

- Conductor and Agent Evaluation responsibilities are independent.
- Run and attempt semantics preserve history.
- Model proposals cannot bypass runtime authority.
- Approval-required actions fail closed in Phase 1; when approval workflows are enabled, approvals are revalidated at dispatch.
- Dispatch intent precedes external side effects and is atomically coordinated with approval consumption and durable budget reservations.
- Cancellation acceptance and dispatch authorization have a single authoritative ordering.
- Phase 1 restart handling never blindly redispatches an unresolved operation and preserves its budget reservation.
- Cancellation and retries account for uncertainty.
- Capabilities are runtime-issued and scoped.
- Every attempt has mandatory bounds.
- State and corresponding events are atomic.
- Checkpoints and recovery are consistent and reconciliation-first.
- Provider capabilities and fallback are explicit.
- Budgets are concurrency-safe and usage provenance is retained.
- Agent versions pin immutable dependencies.
- Delegation authority and budgets cannot exceed parent grants.
- Evaluation uses a versioned independent evidence contract.
- Execution isolation claims match actual backend guarantees.

**This specification authorizes architecture-guided Phase 0 work. It does not authorize unrestricted implementation of every later-phase capability.**
