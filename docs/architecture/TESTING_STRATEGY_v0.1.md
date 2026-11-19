# Testing Strategy v0.1 — Invariant-Driven Verification

**Status:** Accepted baseline
**Phase:** Phase 0 — Architecture and design
**Scope:** Testing principles, test layers, verification boundaries, and invariant-to-test traceability

## 1. Purpose

This document defines how CyberCore Conductor verifies its architectural invariants.

Testing is organized around observable guarantees and failure behavior, not merely code coverage. A passing test suite must provide evidence that the runtime enforces its authority, lifecycle, durability, resource, and recovery contracts.

This strategy complements:

- `ARCHITECTURE_SPEC_v0.2.md`, especially Section 23.
- `DOMAIN_MODEL_v0.1.md`.
- `PERSISTENCE_DESIGN_v0.1.md`.
- Applicable accepted ADRs.

It does not authorize implementation of features that remain phase-gated or implementation-gated.

## 2. Testing principles

1. **Invariant-first:** Every critical invariant must map to one or more executable tests before the affected functionality ships.
2. **Deterministic by default:** Core tests use controlled clocks, deterministic identifiers where appropriate, fake providers, fake tools, and explicit test fixtures.
3. **Test failure behavior:** Tests verify rejection, rollback, blocking, and recovery behavior—not only successful execution.
4. **Verify durable evidence:** Persistence tests inspect both authoritative state and corresponding event history.
5. **Isolate external systems:** Ordinary CI tests must not require live paid providers, production credentials, or uncontrolled external services.
6. **Exercise real boundaries:** Fakes verify orchestration contracts; integration tests verify actual adapters, database transactions, and filesystem behavior.
7. **No false exactly-once claims:** Tests must distinguish local transactional guarantees from uncertain external side effects.
8. **Independent verification:** Tests must not treat an agent's self-reported success as proof of successful execution.
9. **Phase-aware coverage:** Tests for later-phase capabilities become release gates before those capabilities ship, not prerequisites for the initial trusted-workload Phase 1.
10. **Traceable failures:** Test names and failure output should identify the violated contract or invariant.

## 3. Test layers

### 3.1 Unit tests

Test isolated domain behavior without external infrastructure.

Required subjects include:

- Run and Attempt transition validation.
- Retry and reconciliation classifications.
- Cancellation rules.
- Capability derivation and scope checks.
- Budget arithmetic and limit enforcement.
- Schema and input validation.
- Usage provenance classification.
- Immutable execution-limit assignment validation.

Unit tests should cover valid transitions, invalid transitions, boundary values, and prohibited actions.

### 3.2 State-machine and property-oriented tests

Test sequences of operations against the domain invariants.

Required subjects include:

- Invalid Run and Attempt transition rejection.
- Deterministic replay of recorded transition and command sequences, verifying resulting domain state, event ordering, and idempotency. This does not require full event sourcing or make event replay authoritative for current state.
- At most one active Attempt per Run.
- Attempt identity and attempt-number monotonicity.
- Resumption of a waiting Attempt without creating a new Attempt.
- Retry creating a new Attempt only after the previous Attempt has a known terminal outcome and policy permits it.
- Run terminal-state restrictions.
- Aggregate Run version advancement on every durable Run or Attempt lifecycle transition.
- Event sequence monotonicity and uniqueness.
- Cancellation and dispatch ordering.

Generated operation sequences may be used when they improve coverage. A property-testing dependency is not mandated at this stage; select one when implementation complexity and test value justify it.

### 3.3 Adapter contract tests

Verify that provider, tool, executor, and persistence adapters obey their declared interfaces.

Tests must cover:

- Input and output schema compliance.
- Timeout and cancellation behavior.
- Error classification.
- Idempotency and reconciliation declarations.
- Capability requirements.
- Usage and provenance reporting.
- Handling of unsupported or unknown capabilities.

Contract tests use controlled fakes for core CI. Adapter-specific tests may use local test services where necessary.

### 3.4 Persistence integration tests

Use a real temporary SQLite database when the persistence implementation exists.

Verify:

- Foreign-key enforcement and schema migrations.
- Atomic state and event updates.
- Run aggregate version advancement.
- Per-Run event sequence allocation.
- Conditional state-version checks.
- Cancellation and dispatch serialization.
- Atomic budget reservation and settlement.
- Command deduplication.
- Checkpoint consistency and integrity.
- Event immutability.
- Recovery behavior after reopening the database.

Tests must inspect both the resulting database state and the event history.

### 3.5 Fault-injection tests

Inject failures at durable and external boundaries.

At minimum, cover:

- Failure before a state-transition transaction commits.
- Failure after state and event commit.
- Crash after dispatch-intent commit but before request transmission.
- Crash after request transmission but before result persistence.
- Duplicate delivery.
- Timeout with uncertain external outcome.
- Partial or failed persistence.
- Checkpoint creation interrupted by a state transition.
- Recovery with multiple independently unresolved operations.

The test must establish what is known, what remains uncertain, and whether new work is correctly blocked.

### 3.6 Security tests

Before the relevant capability ships, test:

- Prompt injection and untrusted memory content.
- Authorization bypass attempts.
- Capability scope expansion.
- Approval misuse, expiry, and replay.
- Secret leakage through logs, arguments, artifacts, and errors.
- Path traversal.
- Resource exhaustion and limit bypass.
- Unsafe tool and executor inputs.

Security tests must verify fail-closed behavior wherever authorization or protected resources are involved.

### 3.7 End-to-end and acceptance tests

Test complete workflows through public runtime boundaries.

Use deterministic fake providers and tools for ordinary CI. Verify externally observable outcomes, durable history, authorization decisions, and recovery behavior.

Live paid-provider tests are opt-in and must not be required for ordinary pull-request validation.

## 4. Phase-specific requirements

### 4.1 Phase 0

- Establish test architecture and invariant traceability.
- Preserve the existing Rust formatting, compilation, and Clippy checks.
- Do not introduce runtime dependencies solely to support a future test design.
- Identify test seams and contracts before implementation.

### 4.2 Phase 1

Before the corresponding functionality ships, require:

- Domain and state-machine unit tests.
- Deterministic fake provider and tool tests.
- SQLite transaction and event-ordering integration tests.
- Cancellation and dispatch ordering tests.
- Durable dispatch-intent tests.
- Budget reservation concurrency tests.
- Run-level aggregate budget enforcement across all Attempts.
- Maximum execution steps, Attempts, retries, model requests, and tool dispatches.
- Aggregate input/output token limits and per-request token ceilings.
- Run-level monetary ceiling, versioned pricing, and unknown-pricing rejection.
- Atomic pre-dispatch reservations and concurrent-operation ceiling.
- Failed-operation counting and exclusion of policy denials and user cancellation.
- Run elapsed-time, aggregate tool-time, and individual tool-time ceilings.
- Serialized request/response and tool argument/result payload-size ceilings.
- Immutable Attempt snapshots of remaining Run allocation and the absolute Run deadline.
- Retry aggregation: a new Attempt never replenishes Run-level budgets.
- Unknown usage reservation retention and reconciliation.
- Rejection of implicit budget escalation through model output or tool arguments.
- Reconciliation-first recovery tests.
- Fail-closed approval behavior tests.
- Crash and fault-injection tests for implemented durable boundaries.
- CI execution of the workspace test suite.

Phase 1 must not silently treat approval-required actions as authorized. Tests must verify that actions requiring the unavailable Phase 2 approval workflow remain blocked.

### 4.3 Later phases

The following become mandatory gates before their respective capabilities ship:

- Persisted approval lifecycle and one-time consumption.
- Full checkpoint-based crash recovery.
- Provider routing and fallback behavior.
- Persistent user memory and its ownership/retention rules.
- Artifact staging, integrity, and finalization.
- Delegation authority, child budgets, concurrency, and failure semantics.
- Distributed ownership, leases, and fencing.
- PostgreSQL adapter behavior.
- Production backup and restore verification.
- Adversarial security testing for expanded execution environments.

A later-phase test requirement does not authorize that feature for Phase 1.

## 5. Invariant-to-test matrix

| Invariant | Required verification | Expected failure behavior |
|---|---|---|
| Runtime owns execution authority | Model proposes unauthorized action | Proposal is rejected; no dispatch occurs |
| Run and Attempt transitions are valid | Exercise permitted and prohibited transitions | Invalid transition is rejected without partial state change |
| Attempt identity is immutable | Retry and resume scenarios | Resume retains identity; retry creates a new Attempt |
| Only one active Attempt exists | Competing Attempt creation | At most one creation succeeds |
| Run aggregate version is monotonic | Run and Attempt lifecycle transitions | Every committed transition increments exactly once |
| State and event are atomic | Inject failure between state update and event append | Entire transaction rolls back |
| Event sequence is per-Run and monotonic | Competing event allocations | No duplicate or missing committed sequence |
| Cancellation and dispatch serialize | Race cancellation against dispatch authorization | Commit order determines the authorized outcome |
| Dispatch intent precedes side effects | Crash around dispatch boundary | No external execution without committed intent |
| Approval is exact and fail-closed | Expired, mismatched, replayed, or unavailable approval | Dispatch is rejected |
| Capability grants are scoped | Attempt broader resource or operation access | Authorization is rejected |
| Execution limits are immutable | Change mutable defaults after Attempt creation | Recovery uses the original verified assignment |
| Budget reservations are atomic | Competing reservations for the same capacity | No over-allocation |
| Unknown usage is not zero | Provider omits or cannot establish usage | Usage remains unknown or explicitly estimated |
| Run-level budget aggregation | Consume capacity across multiple Attempts and retries | No Attempt or retry replenishes Run capacity |
| Attempt snapshot immutability | Change defaults after Attempt creation and recover | Original remaining allocation and absolute Run deadline remain authoritative |
| Hard execution ceilings | Exercise each Phase 1 count and time ceiling at and beyond its boundary | Over-limit work is rejected before dispatch |
| Token and payload ceilings | Exceed aggregate and per-request token and byte limits | Request is rejected before dispatch |
| Cost reservation and pricing | Exercise maximum-cost reservation, unknown pricing, and settlement | Unknown paid pricing is rejected; reservations remain conservative |
| Concurrent external-operation ceiling | Attempt overlapping external dispatches | A second operation is not authorized while one remains active or unresolved |
| Failed-operation ceiling | Exercise failed operations, policy denials, and user cancellation | Only failed execution operations consume the failure allowance |
| Unknown usage reservation | Leave provider usage unavailable or indeterminate | Reservation remains charged until reconciliation or audited adjustment |
| No implicit profile escalation | Request larger limits through model output or tool arguments | Runtime rejects unauthorized expansion |
| Retry respects operation contract | Exercise all retry classifications | Unsafe or non-reconcilable uncertain work is not blindly repeated |
| Recovery is reconciliation-first | Leave an external operation unresolved | Dependent new decisions remain blocked |
| Checkpoints represent exact state | Race checkpoint creation with transitions | Inconsistent checkpoint is rejected |
| Events are append-only | Attempt event update or deletion | Mutation is rejected |
| Artifacts are verified before reference | Interrupt staged artifact write | No authoritative reference to unverified content |
| Evaluation is independent | Agent claims success without supporting evidence | Claim is not accepted as independent proof |
| Child authority is bounded | Child attempts to exceed parent grants or budget | Excess authority or allocation is rejected |

### 5.1 Persistence failure-scenario coverage

The following scenarios are mandatory persistence integration or fault-injection
tests. They trace directly to the failure scenarios in
`PERSISTENCE_DESIGN_v0.1.md`, Section 11.

| Failure scenario | Required verification | Expected failure or recovery behavior |
|---|---|---|
| Crash before state-transition commit | Interrupt the transaction before commit | No partial state or event change becomes visible |
| Crash after state and event commit | Restart after successful commit | Both authoritative state and event history remain durable |
| Crash after dispatch-intent commit but before transmission | Interrupt before sending the external request | Recovery reconciles the operation; no blind redispatch |
| Crash after transmission but before result persistence | Interrupt after the external request is sent | Outcome remains uncertain until reconciled; no assumed success or blind retry |
| Timeout with unknown external outcome | Return a timeout without establishing the remote result | Operation remains unresolved; affected decisions remain blocked |
| Cancellation racing with dispatch authorization | Exercise both transaction orderings | Shared serialization determines the outcome; no unauthorized dispatch |
| Concurrent budget reservations | Compete for the same remaining capacity | Reservations remain atomic; capacity cannot be over-allocated |
| Duplicate command delivery | Repeat the same command key and digest, then vary the digest | Identical delivery is deduplicated; changed content is rejected |
| Duplicate event insertion | Attempt to insert an existing event identity or Run sequence | Uniqueness constraints reject the duplicate without corrupting history |
| Event sequence allocation under concurrent writers | Compete to append events to one Run | Committed sequence numbers remain unique and consecutive |
| Checkpoint creation racing with a transition | Interleave checkpoint and state-transition transactions | Checkpoint represents one consistent state/version/sequence boundary |
| Invalid or incompatible checkpoint schema | Load an unsupported or malformed checkpoint | Checkpoint is rejected; recovery does not trust partial or incompatible state |
| Migration interruption and restart | Interrupt migration, then reopen and migrate again | Database remains recoverable; migration state is consistent and repeatable |
| Database busy or locked errors | Inject lock contention and transaction timeout | Operation fails or retries within policy; no partial transaction is exposed |
| Foreign-key violation and rollback | Attempt an invalid relational write | Constraint rejects the write and the entire transaction rolls back |
| Attempt completion racing with Run transition | Interleave Attempt and Run lifecycle transitions | Aggregate version, state, and events remain serialized and consistent |
| Multiple independently unresolved operations in one Attempt | Recover with more than one incomplete external operation | Each operation is reconciled or explicitly resolved; dependent decisions remain blocked until all are handled |

These tests must verify both authoritative persisted state and the corresponding
durable event history. A test that checks only an API response or mocked result
does not establish persistence correctness.

## 6. Test fixtures and determinism

Core tests should use reusable fixtures with explicit ownership and cleanup.

The test harness should support:

- Controllable clocks and timeout progression.
- Deterministic fake model/provider responses.
- Fake tools with configurable outcomes and side effects.
- Simulated duplicate delivery and delayed responses.
- Injected failures at named persistence and dispatch boundaries.
- Temporary isolated SQLite databases.
- Explicit resource and budget limits.
- Captured event histories and authorization decisions.

Tests must not depend on wall-clock timing where a controllable clock can provide deterministic behavior.

## 7. CI and quality gates

The existing CI toolchain matrix remains authoritative:

- Rust 1.85.0 MSRV.
- Rust 1.100.0 pinned project toolchain.

For each matrix toolchain, CI should run:

1. Formatting check.
2. Workspace compilation and all-target check.
3. Clippy with warnings denied.
4. `cargo test --workspace --all-targets`.

Additional integration or security jobs may be separated when runtime cost or infrastructure requirements justify it. Required pull-request checks must remain deterministic and must not require paid provider credentials.

The current CI workflow does not yet execute `cargo test`; adding that step is an implementation task, not part of this architecture-only document.

## 8. Coverage and release criteria

Line or branch coverage may help identify untested code, but coverage percentage alone does not establish architectural correctness.

Before a capability ships:

- Its applicable invariants must map to executable tests.
- Critical failure and rejection paths must be exercised.
- Persistence behavior must be verified at actual transaction boundaries.
- Recovery tests must establish safe behavior under uncertainty.
- Required tests must run in CI or an explicitly documented release gate.
- Known test gaps must be recorded and accepted by the appropriate review authority.

No feature may claim a tested guarantee when its corresponding test only verifies a mock or declaration rather than the relevant enforcement boundary.

## 9. Deferred decisions

This strategy intentionally does not select:

- A property-testing framework.
- A mocking framework.
- A dedicated fault-injection framework.
- A coverage threshold.
- A benchmark framework or performance budget.
- A live-provider test schedule.
- A particular CI job-splitting strategy.

These choices should be made when implementation needs and measurable constraints are known.

## 10. Review and acceptance criteria

This draft is ready for architectural review when reviewers can confirm:

- All required test categories in Architecture Specification v0.2 are represented.
- Persistence failure scenarios map to executable test expectations.
- Critical domain and authority invariants have explicit test coverage.
- Phase 1 requirements are separated from later-phase gates.
- Deterministic CI behavior is preserved.
- No test strategy statement silently authorizes a deferred feature.

Acceptance of this strategy establishes verification expectations. It does not mean the tests or the functionality have been implemented.
