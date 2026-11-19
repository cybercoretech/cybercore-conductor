# ADR-027 — Phase 1 Default Run Budgets and Execution Ceilings

**Status:** Accepted
**Date:** 2026-11-18

## Context

The architecture requires bounded execution, aggregate resource accounting, immutable Attempt assignments, and atomic reservations. OPEN-009 left the concrete Phase 1 defaults unresolved.

The runtime needs one explicit, testable default profile before budget enforcement and runtime bootstrap can proceed. Limits must apply to the logical Run rather than replenishing with each retry or Attempt.

## Decision

Accept the Phase 1 default Run profile below. All Run-level totals aggregate across every Attempt belonging to the logical Run.

### 1. Execution and request limits

| Dimension | Accepted default |
|---|---:|
| Total execution steps | 100 per Run |
| Total Attempts | 3, including initial Attempt |
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

The three-Attempt ceiling includes the initial Attempt; two retries are the maximum additional Attempts.

### 2. Payload and serialized-content limits

| Payload | Accepted maximum |
|---|---:|
| Serialized model request | 256 KiB |
| Serialized model response | 64 KiB |
| Tool arguments | 64 KiB per call |
| Tool result | 64 KiB per call |
| Aggregate tool results | 1 MiB per Run |

KiB means 1,024 bytes. Token and byte ceilings are independent and both apply.

### 3. Run and Attempt accounting

The Run owns the authoritative aggregate budget. Retries and new Attempts do not reset or replenish any Run-level limit.

Every Attempt receives an immutable snapshot of the remaining Run-level allocation at Attempt creation. The snapshot is not a new allocation or an independent Attempt budget. The Run account remains authoritative.

The Attempt assignment includes the Run's absolute deadline. An Attempt cannot extend that deadline or increase the remaining Run allowance.

No separate per-Attempt aggregate ceilings are introduced by this decision beyond the immutable assignment of the remaining Run allocation and the applicable per-request and per-operation limits.

A retry creates a new Attempt and consumes the remaining Run allocation. An indeterminate external operation must be reconciled before retry authorization.

### 4. Reservation and enforcement semantics

A request that would exceed any hard ceiling is rejected before dispatch.

Reservation and authorization are atomic. Concurrent operations cannot reserve the same remaining capacity. Dispatch requires a durable reservation bound to the exact operation.

Model-request capacity is consumed when durable dispatch authorization and reservation are committed. Tool-call capacity is consumed when the durable dispatch intent is committed.

Failed execution operations count toward the failure ceiling. Policy denials and user-requested cancellation do not.

The Run clock starts when execution starts and includes waiting and recovery time. Tool execution time accumulates across calls. The individual 60-second timeout does not replace the aggregate five-minute tool-time limit.

The concurrent external-operation limit is one. A second operation cannot be authorized while the first remains active or unresolved.

Attempt limits are immutable after creation. Model output, provider responses, tool arguments, and retry requests cannot expand them.

Exceeding the default profile requires an explicitly configured and validated alternative profile. There is no implicit escalation through model output or tool arguments.

### 5. Cost accounting

Every provider adapter handling paid model requests must use a versioned pricing schedule for the applicable provider and model.

Before dispatch, the runtime reserves the maximum estimated request cost, including permitted output, using conservative token estimates and configured provider rates.

Reported or measured usage settles the reservation. If usage cannot be established, the reservation remains charged against available capacity until reconciliation or an audited adjustment.

Unknown paid-request pricing causes rejection. Unknown pricing is never treated as zero.

A local provider may use an explicitly configured zero-monetary-cost policy, but remains subject to all token, time, execution, payload, and other resource ceilings.

The USD 1.00 ceiling is a runtime accounting limit based on configured rates, not a guarantee that a provider invoice will match the estimate.

### 6. Usage provenance and uncertainty

Usage records distinguish measured, provider-reported, estimated, unknown, and reconciled usage.

Unknown usage is not zero. Conservative reservations remain charged until reconciliation or an explicit, auditable resolution establishes the appropriate settlement.

Reservation expiry does not automatically release capacity while an associated external operation may still be running or its outcome is unresolved.

### 7. Counting rules

- A model request counts when its durable dispatch authorization and reservation are committed, not only when a successful response arrives.
- A tool call / dispatch counts when its durable dispatch intent is committed.
- Retries consume additional Run-level capacity.
- The three-Attempt limit includes the initial Attempt.
- Failed execution operations count toward the failure ceiling; policy denials and user-requested cancellation do not.
- The Run elapsed-time clock includes waiting and recovery after execution starts.
- Tool execution time accumulates across calls.
- An indeterminate external operation does not authorize an automatic retry.
- A new Attempt receives only the remaining Run allocation.

## Rationale

Run-level accounting prevents retries from multiplying resource limits. Immutable Attempt assignments make recovery deterministic and auditable. Atomic reservations prevent concurrent over-allocation. Conservative cost and usage accounting prevents unknown values from being interpreted as free capacity.

The defaults provide explicit Phase 1 ceilings that can be tested consistently across domain, persistence, runtime, provider, and tool boundaries.

## Consequences

- Architecture Specification Section 10 defines the Run budget, Attempt assignment, profile, reservation, cost, and provenance contracts.
- Domain Model and Persistence Design must preserve aggregate Run ownership and immutable Attempt assignments.
- Testing Strategy must verify every hard ceiling, aggregation rule, reservation boundary, and unknown-usage behavior.
- Security Threat Model may rely on the accepted limits as architectural requirements but must not imply that enforcement is already implemented or verified.
- Additional profiles and override policies remain outside this decision and require explicit future decisions.
- OPEN-009 is resolved. Unrelated open security and provider decisions remain open.
