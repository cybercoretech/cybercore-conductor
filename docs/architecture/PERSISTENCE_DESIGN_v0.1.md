# Persistence Design v0.1 — SQLite and Recovery

**Status:** Draft for architecture review
**Scope:** Phase 1 single-process persistence design
**Depends on:** Architecture Specification v0.2, Domain Model v0.1, ADR-001 through ADR-019

## 1. Purpose

This document defines the initial persistence design for CyberCore Conductor.

It translates the accepted architecture and domain model into a proposed relational storage model, transaction contracts, migration strategy, checkpoint rules, recovery behavior, and persistence test requirements.

This is a design artifact. It does not authorize runtime implementation or resolve open decisions identified below.

## 2. Scope and deployment assumptions

### 2.1 Initial target

Phase 1 targets one Conductor process using a local SQLite database.

The process is the sole application-level owner of the database. SQLite remains responsible for transactional integrity and write serialization.

The design MUST NOT rely on multiple application processes concurrently coordinating work through SQLite.

### 2.2 Explicitly deferred

The following are extension points, not Phase 1 requirements:

- Multiple Conductor processes sharing one database.
- Distributed worker ownership and lease coordination.
- Distributed fencing and remote executor ownership.
- PostgreSQL-specific persistence behavior.
- Cross-node transaction coordination.

The persistence port boundary MUST permit a future PostgreSQL adapter without introducing SQLite types into domain contracts.

### 2.3 Core persistence principles

1. Relational records represent authoritative current state.
2. Events preserve append-only history and audit evidence.
3. Checkpoints accelerate recovery; they do not replace authoritative state.
4. State transitions and corresponding events are atomic.
5. External side effects require committed durable dispatch intent.
6. Uncertain external outcomes are reconciled before potentially duplicative work.
7. Database transactions never remain open across model calls or external execution.
8. Schema evolution is explicit and versioned.

## 3. Logical schema

The following tables are proposed logical structures. Exact SQL types, indexes, and column definitions will be finalized after architecture review.

### 3.1 `runs`

Authoritative current state for each logical Run.

Required data:

- `run_id` — immutable primary key.
- `state` — current Run lifecycle state.
- `version` — monotonically increasing Run aggregate version. It advances for every durable Run or Attempt lifecycle transition, including an Attempt-only transition that leaves the Run's own state unchanged.
- `attempt_count` — number of durably created Attempts.
- `next_event_sequence` — next per-Run event sequence to allocate.
- `created_at` and `updated_at` — descriptive timestamps.
- `terminal_at` — nullable terminal-state timestamp.

The Run state and version are authoritative. Event history is not replayed to determine ordinary current-state reads.

### 3.2 `attempts`

One row per concrete execution Attempt.

Required data:

- `attempt_id` — immutable primary key.
- `run_id` — foreign key to `runs`.
- `attempt_number` — unique within the Run.
- `state` — Attempt lifecycle state.
- `input_digest` — digest of immutable execution inputs.
- `manifest_digest` — digest identifying the immutable execution manifest.
- `execution_limits_snapshot_ref` — reference to the immutable execution-limit assignment for this Attempt.
- `execution_limits_digest` — digest of the exact assigned limits.
- `created_at`, `started_at`, and `completed_at`.

The execution-limit assignment MUST capture the remaining allocation from the authoritative Run-level budget at Attempt creation, including applicable step, model-request, tool-call, failure, time, context, and output bounds. This assignment is an immutable snapshot of remaining Run capacity, not a separate Attempt budget or a replenishment of Run capacity. It MUST include the Run's absolute deadline and MUST NOT extend that deadline. Recovery MUST retrieve and verify the exact assignment without consulting mutable defaults. Retries receive only the remaining Run allocation.

Usage accounting MUST remain distinguishable from assigned limits. Unknown or unmeasured usage MUST NOT be represented as zero.

Constraints:

- `(run_id, attempt_number)` MUST be unique.
- An Attempt MUST NOT be reopened after reaching a terminal state.
- A resumed waiting Attempt retains its identity and attempt number.
- A retry creates a new Attempt.
- At most one active Attempt may exist for a Run.

The database-enforced partial unique index MUST treat exactly these states as active:

- `created`
- `running`
- `waiting`
- `reconciling`

The index predicate MUST be kept consistent with the accepted Domain Model state vocabulary. Terminal states (`succeeded`, `failed`, `cancelled`, and `indeterminate`) MUST NOT occupy the active-Attempt slot.

### 3.3 `tool_operations`

One row per logical tool operation within an Attempt.

Required data:

- `operation_id` — immutable primary key.
- `run_id` and `attempt_id` — owning execution identities.
- `operation_number` — stable ordering within the Attempt.
- `tool_name` and `tool_version`.
- `argument_digest` — digest of normalized arguments.
- `retry_class` — declared retry and reconciliation classification.
- `lifecycle_state` — operation lifecycle state.
- `executor_identity`.
- `idempotency_key`, when applicable.
- `result_digest`, when available.
- Creation and update timestamps.

An Attempt may contain multiple operations with independent dispatch and reconciliation lifecycles.

Sensitive argument and result payload storage remains subject to OPEN-006.

### 3.4 `dispatch_intents`

Durable authorization records for external side effects.

Required data:

- `dispatch_intent_id` — immutable primary key.
- `run_id`, `attempt_id`, and `operation_id`.
- `argument_digest`.
- `authorization_decision_reference`.
- `capability_grant_references` — exact immutable identities or versions of the runtime-issued grants authorizing the operation.
- `approval_reference`, when required.
- `budget_reservation_reference`.
- `idempotency_key`, when applicable.
- `executor_identity`.
- `authorized_at`.
- Dispatch and reconciliation status metadata.

A committed dispatch intent means the operation is authorized and in flight for cancellation and recovery semantics, even if the external request has not physically left the process.

For the initial design, each logical tool operation has at most one dispatch intent. The operation's identity represents one authorized dispatch lifecycle. A retry or separately authorized execution MUST create a new operation identity rather than mutate or reuse the previous intent. The database MUST enforce uniqueness of `operation_id` in `dispatch_intents`. Any future support for multiple dispatch authorizations under one operation requires a separately reviewed contract.

A dispatch intent MUST NOT be created unless the transaction verifies the expected Run state/version, cancellation state, current policy and capability validity, required approval validity and consumption, and active budget reservation.

### 3.5 `events`

Append-only event history.

Required data:

- `event_id` — globally unique event identity.
- `run_id` — owning Run.
- `attempt_id` — nullable when not applicable.
- `sequence_number` — monotonically increasing within the Run.
- `event_type`.
- `schema_version`.
- `occurred_at` — descriptive timestamp.
- `state_version` — post-transition Run aggregate version.
- `correlation_id`, where applicable.
- Payload digest and permitted payload/reference metadata.

An event caused by an Attempt-only transition records the newly incremented Run aggregate version even when the Run's own lifecycle state is unchanged. All lifecycle events for one Run therefore share one monotonically increasing aggregate-version domain and one per-Run event-sequence domain.

Constraints:

- `event_id` MUST be unique.
- `(run_id, sequence_number)` MUST be unique.
- Events MUST NOT be rewritten to repair or replace authoritative state.
- No global event ordering across unrelated Runs is required.
- The SQLite adapter MUST enforce event immutability against ordinary application writes. Database triggers MUST reject `UPDATE` and `DELETE` on `events`.
- Any future retention or archival exception requires an explicit policy and migration design; it is not an implicit application-level delete path.

Event payload retention, encryption, redaction, and protected-payload storage remain unresolved under OPEN-006.

### 3.6 `checkpoints`

Versioned recovery snapshots.

Required data:

- `checkpoint_id`.
- `run_id`.
- `attempt_id`, when applicable.
- `schema_version`.
- `run_state_version`.
- `last_included_event_sequence`.
- Pending-operation state references.
- Budget reservation references.
- Context references.
- Integrity digest.
- Creation timestamp.

A checkpoint MUST be transactionally consistent with the Run aggregate version and event sequence it identifies. Its `run_state_version` field identifies the Run aggregate version after all represented Run and Attempt lifecycle transitions. `last_included_event_sequence` identifies the final event included in that snapshot. These are distinct coordinates and MUST both be validated during recovery.

A checkpoint MUST NOT imply that an uncertain external operation completed.

### 3.7 `budget_reservations`

Durable resource reservations.

Required data:

- `reservation_id`.
- `run_id`, `attempt_id`, and operation reference where applicable.
- Resource dimension.
- Reserved amount and unit.
- Lifecycle state.
- Creation and settlement timestamps.
- Usage provenance and reconciliation references where applicable.

Uncertain in-flight operations retain conservative reservations until reconciliation or an explicit auditable resolution establishes the appropriate settlement.

The Phase 1 default Run profile is defined by ADR-027 and Architecture Specification Section 10.3. Uncertain reservations remain charged until reconciliation or an explicit auditable resolution.

### 3.8 `budget_accounts`

Authoritative aggregate accounting for each Run and resource dimension.

Required data:

- `run_id`.
- Resource dimension and unit.
- Configured limit.
- Currently reserved amount.
- Settled/consumed amount.
- Aggregate version or equivalent concurrency guard.

The logical key is `(run_id, resource_dimension, unit)`.

`budget_accounts` is the serialization point for capacity decisions. `budget_reservations` remains the auditable per-reservation ledger. Every reservation creation, adjustment, settlement, or release MUST update the corresponding account aggregate and reservation record in the same transaction.

Available capacity is calculated from the account's configured limit minus its reserved and consumed amounts, subject to the resource's defined accounting semantics. The Phase 1 dimensions, units, ceilings, and usage treatment are defined by ADR-027 and Architecture Specification Sections 10.3–10.6. The Run-level account remains authoritative across all Attempts; an Attempt snapshot MUST NOT create new capacity. Unknown usage MUST NOT be silently treated as zero.

### 3.9 `command_deduplication`

Idempotency records for applicable state-changing commands.

Required data:

- Command identity and idempotency key.
- Defined uniqueness scope.
- Associated Run and operation references, where applicable.
- Request digest.
- Recorded outcome/reference.
- Creation timestamp.

The same key with a different request digest MUST NOT silently reuse a previous result.

### 3.10 `schema_migrations`

Migration history.

Required data:

- Migration version.
- Stable migration identifier.
- Migration checksum.
- Applied timestamp.

Applied migrations MUST NOT be silently rewritten. Migration ordering and compatibility checks are explicit.

## 4. Relationships and integrity

The logical relationships are:

- One Run has many Attempts.
- One Run has many Events.
- One Attempt belongs to exactly one Run.
- One Attempt has many tool operations.
- One tool operation belongs to exactly one Attempt.
- One operation may have multiple dispatch/reconciliation records only where the operation contract explicitly permits distinct authorized dispatches.
- Budget reservations belong to their defined Run, Attempt, and operation scope.
- Checkpoints belong to a Run and identify a precise state version and event sequence.

Foreign-key enforcement MUST be enabled for every SQLite connection.

Deleting a Run MUST NOT cascade-delete audit events, dispatch intents, or evidence needed to explain external side effects. Retention and deletion policy must be designed explicitly before implementing destructive lifecycle operations.

## 5. Transaction contracts

### 5.1 State transition and event append

The Run version is the version of the Run aggregate, not merely a counter for changes to the Run's top-level state.

Every durable Run or Attempt lifecycle transition MUST advance the owning Run's aggregate version exactly once. An Attempt-only transition therefore increments the Run version even when the Run's own state value remains unchanged.

One transaction MUST:

1. Load or conditionally verify the expected Run aggregate version.
2. Validate the permitted Run or Attempt transition and domain invariants.
3. Update the authoritative Run or Attempt state.
4. Increment the owning Run aggregate version exactly once.
5. Allocate the next per-Run event sequence.
6. Append the corresponding event with the resulting post-transition Run aggregate version.
7. Commit all changes together.

Any failure MUST roll back the entire transition.

Attempt completion and the corresponding Run transition MUST be coordinated transactionally. If one logical operation requires multiple lifecycle events in a single transaction, each transition MUST advance the Run aggregate version exactly once, and each event MUST record the aggregate version immediately after its corresponding transition. Event sequence numbers MUST advance consecutively in that same order. All state changes, aggregate-version increments, sequence allocations, and event appends MUST commit atomically. The design MUST NOT allow an event to claim a version or sequence that was not committed with its corresponding state.

### 5.2 Cancellation and dispatch authorization

Cancellation acceptance and dispatch authorization MUST serialize against the same authoritative Run state/version.

If cancellation commits first, subsequent dispatch authorization MUST fail.

If dispatch authorization commits first, the durable dispatch intent establishes the operation as in flight.

After cancellation is accepted, no new dispatch intent may be authorized for that Run.

The database transaction MUST NOT remain open while the executor performs the external operation.

### 5.3 Dispatch authorization

The authorization transaction MUST verify:

- Expected Run state and version.
- Run is eligible for new work.
- Cancellation has not been accepted.
- Current policy validity.
- Exact runtime-issued capability grant identities, scope, resource, operation, lifetime, provenance, and revocation status.
- Required approval validity and one-time consumption.
- Active budget reservation for the exact operation.
- Exact normalized argument digest.
- Executor identity and applicable idempotency information.

Approval consumption, reservation binding, dispatch-intent creation, and the authoritative state transition MUST be coordinated atomically.

### 5.4 Budget reservation

Reservation creation MUST serialize concurrent allocation attempts against the corresponding `budget_accounts` row.

The transaction MUST:

1. Read the authoritative account limit, reserved amount, and consumed amount.
2. Calculate available capacity using the resource's defined accounting semantics.
3. Conditionally update the account's reserved aggregate.
4. Insert the reservation ledger record.
5. Commit both changes atomically.

Reservation settlement, adjustment, release, or uncertain status MUST update the account aggregate and reservation ledger in the same transaction and remain auditable.

The account update MUST use a conditional concurrency guard or equivalent serialized write transaction. It MUST NOT permit concurrent operations to allocate the same available capacity more than once.

Expiry MUST NOT automatically release a reservation while its associated external operation may still be running or unresolved. Unknown usage MUST NOT be silently treated as zero.

For paid model requests, the runtime MUST reserve the maximum estimated cost before dispatch using a versioned provider/model pricing schedule and conservative token estimates. Reported or measured usage settles the reservation. Unknown usage retains the reservation until reconciliation or an audited adjustment. Unknown pricing causes a paid request to be rejected; it MUST NOT be treated as zero. An explicitly configured local zero-cost policy remains subject to all non-monetary limits.

### 5.5 Command idempotency

Applicable commands MUST enforce uniqueness within a defined scope.

A repeated command with the same key and same request digest may return its previously recorded outcome.

A repeated key with a different request digest MUST be rejected as an idempotency conflict.

## 6. SQLite connection and transaction policy

The initial adapter MUST:

- Enable foreign-key enforcement on every connection.
- Use WAL mode where appropriate for the deployment.
- Configure an explicit busy timeout.
- Keep write transactions short.
- Use explicit transaction boundaries.
- Use immediate write transactions for operations requiring serialized authorization or reservation decisions.
- Use conditional updates against expected state versions.
- Avoid holding transactions across model calls, provider requests, or external tool execution.
- Handle busy/locked errors through bounded, explicit retry behavior.
- Verify database integrity as part of the documented maintenance and recovery procedure.

SQLite write serialization is not distributed worker fencing. The single-process assumption MUST remain explicit.

## 7. Migration strategy

Migrations are ordered, versioned, and explicit.

Each migration has:

- A stable identifier.
- A monotonically increasing version.
- A checksum.
- An application timestamp.
- A defined transaction policy.

Migration execution MUST:

1. Read the current schema version.
2. Validate the integrity of previously applied migration records.
3. Apply only unapplied migrations in order.
4. Record each successful migration atomically with its schema changes where SQLite permits.
5. Stop on unexpected version or checksum mismatch.

Destructive migrations require explicit review and a documented data-preservation or backup strategy.

The initial implementation should use embedded, repository-controlled SQL migration files. Migration tooling selection is deferred until implementation planning.

## 8. Checkpoint consistency

Checkpoint creation MUST coordinate:

- The authoritative Run state version.
- The last included per-Run event sequence.
- The Attempt identity and state represented.
- Pending operation and dispatch references.
- Budget reservation references.
- Context and artifact references.
- Checkpoint schema version and integrity digest.

Checkpoint creation MUST NOT race with a state transition in a way that produces a snapshot claiming a state version or event sequence it does not represent.

Recovery MUST validate checkpoint identity, schema compatibility, and integrity before using it.

Authoritative relational state remains the source of truth.

## 9. Recovery behavior

Recovery is reconciliation-first.

The initial recovery design MUST:

1. Discover nonterminal Runs and Attempts.
2. Identify incomplete dispatch intents and unresolved external operations.
3. Restore applicable budget reservations conservatively.
4. Reconcile known or queryable external outcomes.
5. Preserve indeterminate outcomes where certainty cannot be established.
6. Revalidate policy, capabilities, budgets, and the Attempt's persisted immutable execution-limit assignment before resuming.
7. Resume only operations that are safe under their declared retry and reconciliation contract.

### 9.1 Recovery decision gate

Recovery MUST NOT issue a new model call, authorize a new dispatch, or make a new execution decision while an unresolved external operation could affect that decision.

Before resuming an Attempt or proceeding to subsequent work, the runtime MUST establish one of the following for every relevant incomplete operation:

- Its external outcome has been reconciled and recorded.
- Its operation contract establishes that proceeding is safe despite the unresolved outcome.
- An explicit auditable resolution has been recorded under the applicable policy.

If none of these conditions is met, the affected work MUST remain blocked pending reconciliation or explicit resolution.

This gate applies independently to each operation within an Attempt. An Attempt containing multiple operations MUST NOT proceed past a decision point that depends on an unresolved operation merely because another operation has completed.

Recovery MUST NOT assume an interrupted external operation did not happen.

Recovery MUST NOT blindly redispatch an uncertain non-idempotent operation.

An Attempt may become terminal indeterminate only after applicable reconciliation is complete and its outcome still cannot be established. No reconciliation process may remain active for that Attempt.

Phase 1 may use an explicit operator/reconciliation path rather than automated reconciliation across every external system.

## 10. Worker ownership and fencing boundary

Phase 1 assumes one Conductor process owns the database and coordinates execution.

Internal state updates still use expected state versions and transactional validation.

External side effects require dispatch authorization against current authoritative state immediately before execution.

This design does not claim to protect against multiple independent processes sharing the same SQLite database or against a stale distributed worker.

A future distributed deployment requires explicit ownership, leases, fencing, and external dispatch protection. Database write serialization alone is insufficient.

## 11. Failure and crash scenarios

The implementation test plan MUST cover at least:

1. Process crash before a state-transition transaction commits.
2. Process crash after state and event commit.
3. Process crash after dispatch-intent commit but before external request transmission.
4. Process crash after external request transmission but before result persistence.
5. Timeout with unknown external outcome.
6. Cancellation racing with dispatch authorization.
7. Concurrent budget reservations competing for the same capacity.
8. Duplicate command delivery.
9. Duplicate event insertion.
10. Event sequence allocation under concurrent writers.
11. Checkpoint creation racing with a state transition.
12. Invalid or incompatible checkpoint schema.
13. Migration interruption and restart.
14. Database busy/locked errors.
15. Foreign-key violation and rollback behavior.
16. Attempt completion racing with Run transition.
17. Recovery of an Attempt with multiple independently unresolved operations.

Tests MUST verify both the resulting authoritative state and corresponding durable event history.

## 12. Persistence test matrix

| Invariant | Required test |
|---|---|
| State/event atomicity | Inject failure between update and event append; verify complete rollback |
| Run version monotonicity | Competing stale-version transitions; only valid transition succeeds |
| Event sequence uniqueness | Concurrent event allocation; no duplicate per-Run sequence |
| Attempt identity | Resume waiting Attempt without creating a new Attempt |
| Active Attempt uniqueness | Attempt to create two active Attempts for one Run |
| Terminal Attempt immutability | Reject reopening or reusing a terminal Attempt |
| Dispatch durability | External execution cannot begin before intent commit |
| Cancellation ordering | Cancellation-first blocks dispatch; dispatch-first remains recorded in flight |
| Approval atomicity | Failed dispatch authorization does not consume approval partially |
| Budget safety | Concurrent reservations cannot over-allocate |
| Uncertain reservation | Recovery does not release unresolved reservation |
| Command idempotency | Same key/digest is deduplicated; changed digest is rejected |
| Checkpoint consistency | State version and event sequence match the represented snapshot |
| Reconciliation-first recovery | Uncertain operation is reconciled before new potentially duplicative dispatch |
| Crash safety | Inject crashes at each durable/external boundary |
| Migration integrity | Reject checksum mismatch and unexpected schema version |
| Foreign-key integrity | Invalid references fail and roll back |
| SQLite configuration | Verify foreign keys, journal mode, and busy timeout |

## 13. Open decisions and deferred policy

This design does not resolve:

- OPEN-006: event payload retention, encryption, redaction, and protected-payload storage.
- OPEN-005: initial local identity and authentication model.
- OPEN-010: provider data handling and allowed fallback boundaries.

The schema may define references, digests, classifications, and metadata needed to support later decisions. It MUST NOT silently establish unresolved retention, privacy, identity, or execution policies.

## 14. Implementation boundary

This document is a proposed persistence design, not implementation authorization.

Runtime implementation requires:

- Review and acceptance of this design.
- Resolution of any decisions that block the relevant implementation.
- An implementation plan mapped to the persistence test matrix.
- Explicit confirmation of the Phase 1 scope.

No database dependency, migration, runtime persistence adapter, or schema is introduced by this document alone.

## 15. Review checklist

- [ ] Logical tables and relationships match the approved Run/Attempt model.
- [ ] State/event atomicity is enforceable.
- [ ] Cancellation and dispatch authorization serialize correctly.
- [ ] Budget reservations cannot over-allocate.
- [ ] Dispatch intents are durable before external execution.
- [ ] Checkpoints identify exact state versions and event sequences.
- [ ] Recovery is reconciliation-first.
- [ ] SQLite assumptions are explicit and limited to single-process Phase 1.
- [ ] Multi-process and PostgreSQL extensions remain behind persistence ports.
- [ ] OPEN-006 remains unresolved.
- [ ] Failure scenarios map to executable tests.
