# Domain Model v0.1 — Run and Attempt

**Project:** CyberCore Conductor
**Status:** Accepted — Phase 0 baseline
**Date:** 2026-11-18
**Scope:** Core execution identity, lifecycle, and invariants

## 1. Purpose

This document defines the initial domain contract for the two foundational execution entities:

- **Run:** the logical unit of agent execution.
- **Attempt:** one concrete execution instance belonging to a Run.

These definitions establish the vocabulary and lifecycle boundaries used by subsequent domain contracts, persistence design, and runtime implementation.

This document does not authorize implementation of capabilities that remain gated by the Architecture Specification or accepted ADRs. It does not supersede Architecture Specification v0.2.

## 2. Core principles

1. A Run represents logical work independently of how many execution attempts it requires.
2. An Attempt represents one concrete execution of a Run.
3. Every Attempt belongs to exactly one Run.
4. A Run may have multiple Attempts, but each Attempt has one immutable identity.
5. Retrying creates a new Attempt. A previous Attempt is never reset or reused.
6. State transitions must be validated by the runtime.
7. State changes and their corresponding events must be committed atomically.
8. Every Run has a monotonically increasing event sequence.
9. Timestamps provide descriptive chronology; sequence numbers establish event ordering.
10. An uncertain external outcome must be reconciled before another potentially duplicative dispatch.
11. A model proposal is not authorization to execute.
12. The runtime remains the authority for lifecycle transitions and execution authorization.

## 3. Run

### 3.1 Definition

A Run is the durable logical record of one requested unit of agent work.

It represents the complete execution lifecycle, including any permitted retries, regardless of the number of Attempts created.

A Run is not synonymous with a provider request, model response, process, or individual tool invocation.

### 3.2 Required identity and metadata

| Field | Meaning |
|---|---|
| `run_id` | Globally unique, immutable Run identifier |
| `status` | Current lifecycle state |
| `created_at` | Run creation timestamp |
| `updated_at` | Most recent successful state-change timestamp |
| `version` | Monotonically increasing Run aggregate version, advanced on every committed Run or Attempt lifecycle transition |
| `next_event_sequence` | Next event sequence to allocate |
| `attempt_count` | Number of Attempts created for this Run |

Additional fields such as owner, tenant, execution manifest, policy version, and input references will be defined by their respective contracts.

The identifier format is intentionally unspecified at this stage.

### 3.3 Run lifecycle states

| State | Meaning | Terminal |
|---|---|---|
| `created` | Run record exists; initial validation or preparation is incomplete | No |
| `ready` | Run is eligible for runtime scheduling | No |
| `running` | An Attempt is actively executing | No |
| `cancelling` | Cancellation has been accepted; previously authorized operations are being stopped or reconciled | No |
| `waiting` | Execution is suspended pending an explicitly defined condition | No |
| `succeeded` | The requested logical work completed successfully | Yes |
| `failed` | The Run ended unsuccessfully under its execution and retry policy | Yes |
| `cancelled` | The Run was cancelled and no further execution is permitted | Yes |
| `indeterminate` | The final outcome cannot safely be established | Yes |

`waiting` does not itself grant permission to dispatch. The condition that caused the wait must be resolved and revalidated before execution resumes.

`indeterminate` is not equivalent to `failed`. It means the system cannot establish whether the requested work or an external side effect completed.

### 3.4 Permitted Run transitions

| Current state | Next state | Condition |
|---|---|---|
| `created` | `ready` | Initial validation and preparation succeed |
| `created` | `failed` | Initial validation fails terminally |
| `created` | `cancelled` | Cancellation is accepted |
| `ready` | `running` | Runtime resumes an existing waiting Attempt, or creates a new eligible Attempt when no resumable Attempt exists |
| `ready` | `waiting` | A required execution condition is unavailable |
| `ready` | `cancelled` | Cancellation wins before execution starts |
| `running` | `waiting` | Execution pauses for a defined condition |
| `running` | `ready` | The current Attempt has a known terminal outcome, and retry policy permits another Attempt |
| `running` | `succeeded` | The current Attempt succeeded and the logical work is complete |
| `running` | `failed` | The current Attempt failed with a known outcome and no retry is permitted |
| `running` | `cancelling` | Cancellation is accepted; previously authorized operations may remain in flight |
| `running` | `indeterminate` | The current Attempt has completed reconciliation, is terminal `indeterminate`, and no reconciliation process remains active |
| `cancelling` | `cancelled` | All previously authorized operations have known settled outcomes, the active Attempt is terminal, and cancellation stopped execution before logical completion |
| `cancelling` | `succeeded` | All previously authorized operations have known settled outcomes, the active Attempt is terminal and succeeded, and the requested logical work is complete |
| `cancelling` | `failed` | All previously authorized operations have known settled outcomes, the active Attempt is terminal, a known unsuccessful outcome is established, and no retry is permitted |
| `cancelling` | `indeterminate` | Reconciliation has completed, the affected Attempt is terminal `indeterminate`, and no reconciliation process remains active |
| `waiting` | `ready` | The wait condition is resolved and revalidation succeeds |
| `waiting` | `cancelled` | Cancellation is accepted, the active Attempt is terminal, and no authorized operation remains unresolved |
| `waiting` | `cancelling` | Cancellation is accepted while an authorized operation remains unresolved |
| `waiting` | `failed` | The wait condition fails terminally, any existing Attempt is terminal, and no authorized operation remains unresolved |
| `waiting` | `indeterminate` | Reconciliation has completed, the affected Attempt is terminal `indeterminate`, and no reconciliation process remains active |

Any transition not listed here is rejected.

Terminal Run states cannot transition back to an executable state. A new request requiring execution must create a new Run.

### 3.5 Run invariants

- `run_id` never changes.
- `version` increases on every committed Run or Attempt lifecycle transition, including Attempt-only transitions. It is the aggregate concurrency version for the Run and all its Attempts.
- `attempt_count` increases only when a new Attempt is durably created.
- A Run cannot have more than one active Attempt unless a future, explicit concurrency contract permits it.
- A terminal Run cannot create another Attempt.
- A Run cannot become terminal while an Attempt is active or reconciliation is still in progress.
- A Run may become `indeterminate` only after the affected Attempt is terminal `indeterminate` and all applicable reconciliation has completed; no reconciliation process may remain active. This records unresolved external uncertainty, not an active recovery process.
- A Run cannot become `succeeded` while an Attempt remains unresolved.
- A Run cannot become `failed` merely because an external outcome is unknown.
- A Run in `cancelling` cannot authorize new work. Its final state reflects the established outcome: `succeeded` if logical work completed, `failed` if a known unsuccessful outcome is established and no further execution will occur, `cancelled` if execution stopped before logical completion, or `indeterminate` if reconciliation cannot establish the outcome. All authorized operations must have known settled outcomes, except when the final state is `indeterminate`.
- A Run in `indeterminate` cannot be automatically retried.
- Cancellation and dispatch authorization must serialize against the same Run state/version.
- A state transition and its corresponding event are one atomic persistence operation.

## 4. Attempt

### 4.1 Definition

An Attempt is one concrete execution instance of a Run.

It records the execution identity, inputs, and outcome of that specific execution. It is distinct from both the logical Run and individual external tool dispatches.

A retry always creates a new Attempt with a new identifier and a new attempt number.

### 4.2 Required identity and metadata

| Field | Meaning |
|---|---|
| `attempt_id` | Globally unique, immutable Attempt identifier |
| `run_id` | Parent Run identifier |
| `attempt_number` | Monotonically increasing number within the Run |
| `status` | Current Attempt lifecycle state |
| `created_at` | Attempt creation timestamp |
| `started_at` | Execution start timestamp, when applicable |
| `finished_at` | Terminal completion timestamp, when applicable |
| `input_digest` | Digest identifying the immutable execution input snapshot |

Additional metadata will include the execution manifest, policy context, and relevant provider/tool references as those contracts are defined.

The exact digest algorithm and identifier encoding remain implementation decisions.

### 4.3 Attempt lifecycle states

| State | Meaning | Terminal |
|---|---|---|
| `created` | Attempt identity and immutable input snapshot have been recorded | No |
| `running` | Concrete execution is underway | No |
| `waiting` | This Attempt is suspended pending resolution of a defined execution condition | No |
| `reconciling` | Execution outcome is being established after interruption or uncertainty | No |
| `succeeded` | This Attempt completed successfully | Yes |
| `failed` | This Attempt completed unsuccessfully with a known outcome | Yes |
| `cancelled` | This Attempt was stopped with a known cancellation outcome | Yes |
| `indeterminate` | Its outcome cannot safely be established | Yes |

Dispatch authorization is not an Attempt lifecycle state. It belongs to the individual external operation or tool-call record. An Attempt may contain multiple independently authorized operations, each with its own durable dispatch intent and outcome.

`reconciling` is a recovery state, not permission to repeat an operation. An Attempt may leave `reconciling` for `indeterminate` only after all applicable reconciliation steps have completed and its outcome still cannot be established. No reconciliation process may remain active for that Attempt.

### 4.4 Permitted Attempt transitions

| Current state | Next state | Condition |
|---|---|---|
| `created` | `running` | The runtime starts the concrete execution |
| `created` | `cancelled` | Cancellation occurs before execution begins |
| `running` | `succeeded` | Execution completes successfully and all operations have known outcomes |
| `running` | `failed` | Execution fails with a known outcome and no operation remains unresolved |
| `running` | `cancelled` | Execution stops; all authorized operations have known settled outcomes |
| `running` | `waiting` | Execution is suspended pending a defined condition; no new dispatch is authorized while waiting |
| `running` | `reconciling` | Interruption, cancellation, or uncertainty leaves an authorized operation unresolved |
| `waiting` | `running` | The wait condition is resolved and revalidated; the same Attempt resumes |
| `waiting` | `cancelled` | Execution stops and all authorized operations have known settled outcomes |
| `waiting` | `reconciling` | An authorized operation remains unresolved and requires reconciliation |
| `reconciling` | `succeeded` | Reconciliation establishes successful completion |
| `reconciling` | `failed` | Reconciliation establishes unsuccessful completion |
| `reconciling` | `cancelled` | All authorized operations have known settled outcomes and execution has stopped |
| `reconciling` | `indeterminate` | All applicable reconciliation steps have completed, but the outcome still cannot be established; no reconciliation process remains active |

Any transition not listed here is rejected.

### 4.5 Attempt invariants

- `attempt_id` never changes.
- `run_id` never changes.
- `attempt_number` is unique within its Run and increases monotonically.
- Execution inputs are immutable after Attempt creation.
- An Attempt in `waiting` remains the same Attempt; resumption does not create a retry or increment the Run's attempt count.
- A terminal Attempt is never reopened or reused.
- A retry creates a new Attempt rather than modifying an earlier one.
- An Attempt cannot be considered successful solely because a request was submitted.
- Dispatch authorization belongs to individual operation/tool-call records, not the Attempt.
- An Attempt may contain multiple operations with independent dispatch and reconciliation lifecycles.
- Cancellation cannot mark an Attempt cancelled while an authorized operation remains unresolved.
- An unknown external outcome must enter reconciliation rather than trigger blind redispatch.
- Attempt completion and its corresponding Run transition must be coordinated transactionally.
- Attempt history remains attributable to its original execution identity.

### 4.6 Execution-limit assignment

Every Attempt receives an immutable execution-limit assignment at creation.

The assignment records the remaining allocation from its parent Run's authoritative aggregate budget. It is a snapshot of the Run's remaining capacity, not a separate Attempt budget and not a replenishment of Run capacity.

Run-level limits aggregate across all Attempts. Creating a retry MUST NOT reset or expand the Run's budgets.

The Attempt assignment includes the Run's absolute deadline. An Attempt cannot extend that deadline or increase the remaining Run-level allowance.

The Phase 1 Run profile permits at most three Attempts, including the initial Attempt, and at most two retries. A retry creates a new Attempt and requires the previous execution outcome to be sufficiently established for retry authorization. An indeterminate external operation MUST be reconciled before retry.

The Attempt assignment is immutable after creation. Recovery uses the original assignment rather than consulting mutable defaults.

## 5. Run–Attempt relationship

The relationship is one-to-many:

- One Run may own zero or more Attempts.
- Every Attempt belongs to exactly one Run.
- Attempts are ordered by `attempt_number`.
- At most one Attempt may be active for a Run under the initial execution model.
- Historical Attempts remain independently inspectable after the Run becomes terminal.

A Run is the aggregate boundary for scheduling, retry policy, cancellation, and logical completion.

An Attempt is the execution-history boundary for one concrete run of the work.

The Run's current state must agree with its Attempt history. Persistence and recovery design must define how this consistency is maintained across transactions and process restarts.

## 6. Retry semantics

Retry is a Run-level decision that creates a new Attempt.

Before a retry is permitted:

1. The previous Attempt must have a known terminal outcome.
2. The retry policy must permit another Attempt.
3. The Run must remain nonterminal and eligible for execution.
4. Required budgets and execution limits must be available.
5. Required authorization and policy checks must be repeated.

An Attempt in `reconciling` or `indeterminate` must not be retried automatically.

Retry safety depends on the action's declared execution semantics:

- `idempotent`
- `idempotency_keyed`
- `reconcilable`
- `non_reconcilable`

These classifications govern dispatch and recovery behavior. They do not weaken the requirement to establish the previous Attempt's outcome before a potentially duplicative operation.

## 7. Cancellation and dispatch ordering

Cancellation acceptance and each operation's dispatch authorization are competing transitions against the same authoritative Run state/version.

They must serialize so that:

- If cancellation commits first, subsequent dispatch authorization MUST fail.
- If dispatch authorization commits first, its durable dispatch intent establishes that operation as in flight, even if the external request has not physically left the process.
- Once cancellation is accepted, no additional dispatch intent may be authorized for the Run.
- Previously authorized operations are handled through cancellation propagation where supported and outcome reconciliation.
- A cancellation request does not prove that an external operation was cancelled.
- An unresolved external outcome cannot be represented as a confirmed cancellation.

The Run enters `cancelling` when cancellation is accepted while authorized operations may remain unresolved. Cancellation prevents new dispatch authorization but does not rewrite outcomes already established by authorized work. After all authorized operations settle and the active Attempt becomes terminal, the Run's final state reflects the established result: `succeeded` if the logical work completed, `failed` if a known unsuccessful outcome is established and no further execution will occur, or `cancelled` if execution stopped before logical completion. If all applicable reconciliation steps have completed but the outcome remains unknown, the affected Attempt becomes terminal `indeterminate` and the Run may then become `indeterminate`; no reconciliation process may remain active. A cancellation request alone does not determine the final state.

The persistence contract will define the transaction and concurrency mechanism that enforces this ordering.

## 8. Events and state history

Every committed Run or Attempt state transition must produce a corresponding durable event.

Events must include, at minimum:

- Event identifier, unique across the event store.
- Run identifier.
- Attempt identifier when the event concerns an Attempt.
- Per-Run sequence number.
- Event type.
- Schema version.
- Recorded timestamp.
- State version or transition context.
- Correlation information sufficient to trace the originating operation.

The state change and event append must be atomic. The transaction must coordinate the authoritative state update, state-version increment, event append, and per-Run sequence allocation. A partially committed transition or event sequence is not permitted.

Per-Run sequence numbers establish authoritative event ordering. The pair `(run_id, sequence_number)` must be unique, and event identifiers must be unique. Command idempotency keys, when applicable, must also be unique within their defined scope. Wall-clock timestamps are descriptive and must not be used as the sole ordering mechanism. No global event ordering across unrelated Runs is required.

Event payload retention, encryption, redaction, and protected-payload storage remain governed by OPEN-006 and are not decided here.

## 9. Explicitly out of scope

This draft does not define:

- Task, Action, or Tool schemas.
- Provider request and response contracts.
- Authentication or tenant identity mechanisms.
- Approval persistence or approval consumption.
- Event payload retention or encryption policy.
- Memory ownership or retention.
- Public API compatibility guarantees.
- Database schema, SQL migrations, or transaction implementation.
- Distributed scheduling or concurrent Attempts.
- Runtime implementation details.

These will be addressed in subsequent contracts or decision records.

## 10. Review and acceptance criteria

This draft is ready for architecture review when the following questions have been resolved:

- Are the Run and Attempt state names and meanings unambiguous?
- Are all legal transitions represented?
- Are terminal states genuinely terminal?
- Can retries occur only after a known prior outcome?
- Can cancellation and dispatch authorization be serialized without a race?
- Can recovery distinguish known failure from an indeterminate outcome?
- Can persistence maintain Run/Attempt consistency and atomic event history?

Acceptance requires explicit review against Architecture Specification v0.2 and the applicable ADRs.

Until accepted, this document is a proposed domain contract, not an implementation authority.
