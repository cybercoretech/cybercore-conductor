# CyberCore Conductor — Architecture Decision Record Index

**Status:** Consolidated decision register
**Date:** 2026-11-18
**Baseline:** Architecture Specification v0.2

This index tracks the architecture decisions established through Reviews 1–3. Review identifiers CR-001 through CR-023 are retained for traceability. ADR identifiers provide stable references for future implementation, amendments, and superseding decisions.

## Decision status definitions

- **Accepted direction:** Approved as architectural direction and incorporated into v0.2.
- **Implementation-gated:** Accepted direction, but implementation is prohibited until its stated prerequisites are satisfied.
- **Open:** Not decided; must be resolved before the affected capability ships.
- **Superseded:** Replaced by a later ADR. No entries are superseded at this baseline.

Acceptance of an architecture direction does not mean that the feature is implemented or that every implementation detail is settled.

## ADR index

| ADR | Review | Decision | Status |
|---|---|---|---|
| ADR-001 | CR-001 | Logical Run and independently identified Attempts | Accepted direction |
| ADR-002 | CR-002 | Durable Dispatch Intent Before External Side Effects | Accepted direction |
| ADR-003 | CR-003 | Approval Revalidation at Dispatch | Accepted direction |
| ADR-004 | CR-004 | Cancellation Is a Durable Control Request, Not Rollback | Accepted direction |
| ADR-005 | CR-005 | Explicit Tool Retry and Reconciliation Classes | Accepted direction |
| ADR-006 | CR-006 | Trusted-Workload-Only Initial Execution Boundary | Accepted direction |
| ADR-007 | CR-007 | Runtime-Issued Scoped Capability Grants | Accepted direction |
| ADR-008 | CR-008 | Mandatory Per-Attempt Execution and Resource Bounds | Accepted direction |
| ADR-009 | CR-009 | Atomic State Transition and Event Persistence | Accepted direction |
| ADR-010 | CR-010 | Per-Run Monotonic Event Ordering and Idempotency | Accepted direction |
| ADR-011 | CR-011 | Hybrid State, Event, and Checkpoint Persistence | Accepted direction |
| ADR-012 | CR-012 | Versioned Checkpoints Anchored to State and Event Sequence | Accepted direction |
| ADR-013 | CR-013 | Reconciliation-First Recovery Workflow | Accepted direction |
| ADR-014 | CR-014 | Separate State Fencing and Dispatch Fencing | Implementation-gated |
| ADR-015 | CR-015 | Integrity-Checked Artifact Provenance and Finalization | Accepted direction |
| ADR-016 | CR-016 | Explicit Provider Capability Descriptors | Accepted direction |
| ADR-017 | CR-017 | Runtime-Controlled, Policy-Preserving Provider Fallback | Accepted direction |
| ADR-018 | CR-018 | Atomic Concurrent Budget Reservations | Accepted direction |
| ADR-019 | CR-019 | Usage Provenance and Explicit Unknown Accounting | Accepted direction |
| ADR-020 | CR-020 | Immutable Agent Manifests with Pinned Dependencies | Accepted direction |
| ADR-021 | CR-021 | Delegated Authority Derived by Intersection | Implementation-gated |
| ADR-022 | CR-022 | Bounded Delegation and Explicit Child Failure Semantics | Implementation-gated |
| ADR-023 | CR-023 | Versioned Independent Evaluation Evidence Contract | Accepted direction |

## Decision summaries

### ADR-001 — Logical Run and Independently Identified Attempts

**Decision:** A Run represents caller-requested logical work. An Attempt represents one concrete execution. Retries create new attempts under the same run and preserve all prior attempt history.

**Rationale:** Retry history, resource accounting, recovery, and evidence must remain independently inspectable.

**Consequences:** Run aggregate status and retry policy require explicit contracts. Attempt identity is mandatory in execution records.

### ADR-002 — Durable Dispatch Intent Before External Side Effects

**Decision:** An external side effect cannot be dispatched until its authorized dispatch intent has been durably persisted.

**Rationale:** A process crash between authorization and dispatch must not create an unrecorded operation.

**Consequences:** Recovery must reconcile incomplete dispatch records. External operations need idempotency or reconciliation contracts where possible.

### ADR-003 — Approval Revalidation at Dispatch

**Decision:** Approval is bound to the exact action and is necessary where policy requires it, but it is never sufficient by itself to authorize dispatch. Current policy, capability, budget, run eligibility, action digest, expiry, and one-time use are revalidated at dispatch.

**Rationale:** Approval may become stale or the proposed action may change.

**Consequences:** Approval consumption and dispatch-intent creation must be coordinated atomically.

### ADR-004 — Cancellation Is a Durable Control Request, Not Rollback

**Decision:** Cancellation prevents new work and propagates where supported. Already-dispatched operations are reconciled; cancellation does not promise rollback.

**Rationale:** External side effects cannot generally be undone by changing local run state.

**Consequences:** The runtime distinguishes pre-dispatch cancellation, in-flight cancellation, known completion, and uncertain outcomes.

### ADR-005 — Explicit Tool Retry and Reconciliation Classes

**Decision:** Tools declare whether operations are idempotent, idempotency-keyed, reconcilable, or non-reconcilable. Runtime retry behavior follows that declaration.

**Rationale:** Generic automatic retries can duplicate external side effects.

**Consequences:** Uncertain non-reconcilable operations cannot be blindly retried.

### ADR-006 — Trusted-Workload-Only Initial Execution Boundary

**Decision:** Initial execution supports trusted, reviewed workloads. In-process native tools and ordinary local subprocesses are not hostile-code sandboxes. Arbitrary model-generated shell and untrusted executable plugins are disabled by default.

**Rationale:** A capability declaration cannot constrain arbitrary native code running with process privileges.

**Consequences:** Hostile-code execution requires a separately reviewed isolation backend.

### ADR-007 — Runtime-Issued Scoped Capability Grants

**Decision:** Capabilities are runtime-issued grants identifying subject, resource, operation, scope, lifetime, provenance, and revocation state.

**Rationale:** Tools and agent definitions cannot be trusted to grant themselves authority.

**Consequences:** Filesystem and other resource checks must account for traversal, symlinks, and TOCTOU risks.

### ADR-008 — Mandatory Per-Attempt Execution and Resource Bounds

**Decision:** Every attempt has mandatory step, model request, tool call, failure, time, context, output, and applicable executor limits.

**Rationale:** Caller omission or model behavior must not create unbounded execution.

**Consequences:** Limit exhaustion produces an explicit result and is covered by runtime tests.

### ADR-009 — Atomic State Transition and Event Persistence

**Decision:** An authoritative state transition and its corresponding event are committed in one database transaction.

**Rationale:** State and audit history must never disagree due to partial persistence.

**Consequences:** State updates, version increments, event append, and sequence assignment are one atomic operation.

### ADR-010 — Per-Run Monotonic Event Ordering and Idempotency

**Decision:** Event sequence numbers are monotonic within each run. Event IDs, run sequence pairs, and applicable command idempotency keys are unique.

**Rationale:** Timestamps cannot reliably order concurrent or replayed operations.

**Consequences:** No global event ordering across unrelated runs is required.

### ADR-011 — Hybrid State, Event, and Checkpoint Persistence

**Decision:** Current relational records are authoritative state; append-only events preserve history; checkpoints accelerate recovery. Full event sourcing is not required.

**Rationale:** This provides auditability without requiring all reads and recovery to replay the complete event stream.

**Consequences:** Events are not rewritten to match projections. Schema evolution and migrations remain explicit.

### ADR-012 — Versioned Checkpoints Anchored to State and Event Sequence

**Decision:** Checkpoints include schema version, run/attempt identity, state version, last included event sequence, pending operation state, budget reservations, context references, and integrity digest.

**Rationale:** Recovery must know exactly which durable state a checkpoint represents.

**Consequences:** Checkpoint creation is transactionally consistent. A checkpoint cannot turn an uncertain external operation into a presumed success.

### ADR-013 — Reconciliation-First Recovery Workflow

**Decision:** Recovery discovers nonterminal work, acquires ownership, loads authoritative state/checkpoints, reconciles incomplete dispatches, restores reservations, revalidates policy, and resumes only safe operations.

**Rationale:** Restarting a conversation is not equivalent to recovering an autonomous execution.

**Consequences:** No new model call or dispatch occurs while an unresolved operation could affect the next decision.

### ADR-014 — Separate State Fencing and Dispatch Fencing

**Decision:** State fencing protects internal writes. External side effects require separate dispatch authorization against current ownership. Where an external service cannot validate fencing, idempotency and reconciliation are required.

**Rationale:** A database fencing token cannot automatically stop a stale worker from calling an external API.

**Consequences:** Lease expiry does not prove the old worker stopped. Distributed worker leases, fencing, and dispatch fencing are implementation-gated until the distributed platform phase. This decision does not authorize premature cluster machinery in Phase 1.

### ADR-015 — Integrity-Checked Artifact Provenance and Finalization

**Decision:** Artifacts have content digests, size/type metadata, producer provenance, sensitivity classification, storage reference, and retention metadata. Staged content is verified and finalized before an authoritative event references it.

**Rationale:** Evidence must be durable, attributable, and verifiable.

**Consequences:** Reads verify integrity; unreferenced staged objects are garbage-collected after a safe interval.

### ADR-016 — Explicit Provider Capability Descriptors

**Decision:** Providers explicitly declare supported input types, context information, structured output, tool calling, streaming, usage reporting, cancellation behavior, and constraints. Unknown capabilities are not assumed supported.

**Rationale:** Provider differences must not leak into core assumptions or cause unsupported features to be selected.

**Consequences:** Runtime selection checks agent requirements against provider capabilities.

### ADR-017 — Runtime-Controlled, Policy-Preserving Provider Fallback

**Decision:** Provider fallback is an explicit runtime routing decision. It preserves policy, budget, context restrictions, data classification, identity, and trace continuity.

**Rationale:** Providers may differ in trust, data handling, context capacity, and accounting behavior.

**Consequences:** Provider changes are events. Fallback cannot silently cross trust or data-handling boundaries.

### ADR-018 — Atomic Concurrent Budget Reservations

**Decision:** Budget reservations are atomic and associated with a run, attempt, resource dimension, operation, amount, and lifecycle state.

**Rationale:** Concurrent operations must not spend the same remaining budget.

**Consequences:** Uncertain in-flight operations retain conservative reservations until safely reconciled.

### ADR-019 — Usage Provenance and Explicit Unknown Accounting

**Decision:** Usage distinguishes measured, provider-reported, estimated, unknown, and reconciled values, with source and confidence metadata.

**Rationale:** Missing provider usage is not zero, and estimated costs are not confirmed billing.

**Consequences:** Hard limits use conservative reservations when actual usage is unavailable before dispatch.

### ADR-020 — Immutable Agent Manifests with Pinned Dependencies

**Decision:** Agent versions capture or reference immutable versions of instructions, schemas, tools, policy, memory configuration, provider constraints, execution profile, and budget settings. Runs record a resolved manifest digest.

**Rationale:** Reproducibility requires more than pinning a top-level agent version.

**Consequences:** Dependency changes create new versions or manifests. Secrets are never embedded in definitions.

### ADR-021 — Delegated Authority Derived by Intersection

**Decision:** Child authority is the intersection of parent authority, parent delegation grant, child requirements, and current runtime policy.

**Rationale:** A child agent cannot gain authority through its own configuration or provider/tool selection.

**Consequences:** Child budget allocations are reserved against parent budget. Delegated authority intersection is implementation-gated until the delegation phase and its security and budget prerequisites are satisfied.

### ADR-022 — Bounded Delegation and Explicit Child Failure Semantics

**Decision:** Delegation is limited by child count, nesting depth, concurrency, aggregate budget, duration, and result size. Parent behavior for child failure, timeout, cancellation, and partial results is explicit.

**Rationale:** Unbounded delegation creates resource exhaustion and ambiguous completion semantics.

**Consequences:** Child completion does not automatically complete a parent. Bounded delegation and its child failure semantics are implementation-gated until the delegation phase.

### ADR-023 — Versioned Independent Evaluation Evidence Contract

**Decision:** Conductor exports versioned evidence through a runtime-independent contract. Evaluation returns separately attributable findings and evidence gaps.

**Rationale:** Evaluation must not depend on Conductor's internal database or implementation and must remain independent.

**Consequences:** Conductor cannot modify or self-certify evaluation results. Evidence and finding schemas require explicit versioning.

## Open Decision Register

These are not resolved by ADR-001 through ADR-023.

| ID | Open decision | Required before |
|---|---|---|
| OPEN-001 | Product name and legal/package/domain clearance | Public naming and publication |
| OPEN-002 | License and contribution model | Public source release |
| OPEN-003 | Rust edition and MSRV | Toolchain bootstrap |
| OPEN-004 | Initial real provider selection | Provider adapter implementation |
| OPEN-005 | Initial local identity/authentication model | Authenticated run APIs |
| OPEN-006 | Event payload retention and protected-payload strategy | Persistent event payloads |
| OPEN-007 | Memory ownership, retention, and deletion policy | Persistent user memory |
| OPEN-008 | Public API/schema compatibility policy | Public API or SDK publication |
| OPEN-009 | Concrete default budget and execution ceilings | Runtime bootstrap |
| OPEN-010 | Provider data-handling and allowed fallback boundaries | Provider routing/fallback |

## Change Control

- Accepted decisions are amended through new or superseding ADRs, not silent edits.
- Each ADR should eventually have its own file under `docs/architecture/adr/`.
- The index links decisions to their originating review findings.
- Implementation PRs should identify the ADRs they implement.
- If implementation reveals a conflict with an accepted decision, record the conflict and proposed amendment before changing the invariant.
- Architecture Specification v0.2 remains the consolidated baseline until a later version is explicitly accepted.
