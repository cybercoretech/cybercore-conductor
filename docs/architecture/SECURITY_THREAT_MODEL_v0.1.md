# CyberCore Conductor — Security and Threat Model v0.1

**Status:** Draft for architecture review
**Date:** 2026-11-18
**Scope:** Trust boundaries, threat actors, protected assets, attack surfaces, and security invariants

## 1. Purpose

This document establishes the security model for CyberCore Conductor before security-sensitive implementation decisions are made.

It identifies:

- Trust boundaries and authority relationships.
- Threat actors and assumed adversarial capabilities.
- Protected assets and security objectives.
- Attack surfaces and entry points.
- Required security invariants.
- Phase-specific security constraints and unresolved decisions.
- Security test requirements traceable to the invariants.

This document supplements Architecture Specification v0.2. It does not supersede accepted architecture decisions, resolve open decisions, select implementation technologies, or authorize implementation of deferred capabilities.

ADR-027 accepts the Phase 1 default Run budgets and execution ceilings, as specified in Architecture Specification Sections 10.3–10.6. That acceptance establishes the governing architectural limits; it does not mean that budget reservation, usage metering, cost accounting, or limit enforcement controls have been implemented or verified. Those controls remain subject to implementation and the applicable tests.

## 2. Security objectives

Conductor MUST preserve the following objectives:

1. **Authority integrity:** Only the runtime may authorize privileged execution.
2. **Identity integrity:** Actions remain attributable to the caller, Run, Attempt, operation, and applicable executor.
3. **Least privilege:** Every executable capability is explicitly scoped and limited.
4. **Confidentiality:** Credentials and protected data are not disclosed to unauthorized actors or unintended destinations.
5. **Integrity:** Authoritative state, authorization records, events, artifacts, and evidence cannot be silently altered or misattributed.
6. **Availability within bounds:** Execution, resource consumption, retries, and external operations remain bounded.
7. **Recoverability:** Interrupted or uncertain operations do not silently become successful, failed, or safe to repeat.
8. **Auditability:** Security-relevant decisions and actions produce attributable, durable evidence.
9. **Independent verification:** Conductor cannot turn its own claims into independent proof of success.

Security controls MUST fail closed when required identity, authorization, policy, capability, approval, budget, or integrity state cannot be established.

## 3. Foundational trust principles

### 3.1 Runtime authority

A model may propose an action. Only the Conductor runtime may authorize and execute it.

Prompts, agent instructions, provider responses, tool declarations, memory contents, retrieved documents, and tool outputs are not authorization mechanisms.

### 3.2 Data is not authority

Untrusted content MUST NOT:

- Grant or expand capabilities.
- Select arbitrary credentials.
- Change runtime policy.
- Increase execution or budget limits.
- Bypass approval requirements.
- Authorize a dispatch.
- Change the authoritative identity or ownership of a Run.

### 3.3 Explicit trust

Trust MUST be based on an explicit contract, identity, policy decision, provenance, and applicable scope. Familiarity, successful prior execution, relevance, provider reputation, or a declared capability is not sufficient by itself.

### 3.4 No implicit trust escalation

Crossing a boundary requires an explicit, policy-governed transition. Provider fallback, tool invocation, memory retrieval, artifact access, delegation, and execution-backend selection MUST NOT silently increase authority or weaken data-handling restrictions.

## 4. Trust boundaries

The following boundaries are logical security boundaries. They do not prescribe a process layout or deployment topology.

### TB-01: Caller and interface boundary

**Participants:** CLI, future HTTP API, SDKs, MCP adapters, and callers.

**Untrusted inputs:** Commands, identities, task inputs, parameters, uploaded content, and caller-supplied metadata.

**Required properties:**

- Authenticate callers before exposing authenticated operations.
- Authorize each command against its caller and resource scope.
- Validate schemas, sizes, identifiers, and permitted state transitions.
- Never trust caller-supplied ownership, policy decisions, capability grants, or execution results.
- Prevent one caller from accessing another caller's Runs or protected artifacts unless explicitly authorized.

The initial local identity and authentication model remains OPEN-005.

### TB-02: Runtime authority boundary

**Participants:** Runtime orchestration, policy evaluation, capability issuance, budget accounting, approval checks, and dispatch authorization.

**Security role:** This is the authoritative control boundary.

**Required properties:**

- Runtime policy determines whether an action is permitted.
- Only runtime-issued grants confer execution authority.
- Authorization is checked against current authoritative state.
- Dispatch authorization binds the exact action, operation, Run, Attempt, capability grants, applicable approval, budget reservations, and executor.
- Authorization failure, missing state, or uncertainty MUST NOT become permission.

### TB-03: Model and provider boundary

**Participants:** Model providers, model endpoints, provider adapters, and provider-returned content.

**Threat assumptions:** Providers may be compromised, malicious, unavailable, inconsistent, or incorrect. Responses may contain prompt injection, malformed structures, fabricated claims, or unauthorized action proposals.

**Required properties:**

- Provider output is untrusted input.
- Provider responses are schema-validated and normalized.
- Provider claims do not establish successful execution.
- Provider-specific behavior cannot relax runtime policy.
- Provider credentials are supplied only through approved secret handling.
- Provider changes and fallback preserve identity, policy, budget, context restrictions, data classification, and trace continuity.

Provider data-handling and fallback boundaries remain OPEN-010.

### TB-04: Tool and executor boundary

**Participants:** Registered tools, native code, subprocesses, containers, VMs, and remote execution backends.

**Threat assumptions:** Tool implementations may contain defects, misuse granted authority, return hostile content, or perform unintended side effects.

**Required properties:**

- Tool declarations request capabilities; they do not grant them.
- Arguments are validated and bound to an exact operation identity.
- Executors receive only the authority and credentials required for the operation.
- External side effects require committed durable dispatch intent.
- Tool output is untrusted data.
- Executor guarantees MUST accurately describe actual isolation and enforcement.

Phase 1 supports trusted, reviewed workloads only. In-process native code and ordinary local subprocesses are not hostile-code sandboxes.

### TB-05: Memory and context boundary

**Participants:** Conversation state, agent memory, user or tenant memory, retrieved documents, and external knowledge sources.

**Threat assumptions:** Memory may be stale, poisoned, cross-scope, malicious, incorrectly attributed, or contain instructions intended to manipulate execution.

**Required properties:**

- Memory is a data service, not an authority service.
- Retrieved content retains provenance, scope, and trust classification.
- Relevance does not imply trust.
- Runtime policy takes precedence over memory and agent instructions.
- Memory writes require explicit policy.
- Cross-user and cross-tenant access is prohibited unless explicitly authorized.

Persistent user-memory ownership, retention, and deletion remain OPEN-007.

### TB-06: Persistence and event boundary

**Participants:** Run and Attempt state, operation records, dispatch intents, budget records, events, checkpoints, and migrations.

**Threat assumptions:** Data may be corrupted, replayed, duplicated, partially written, accessed outside its intended scope, or manipulated through application defects.

**Required properties:**

- Authoritative state transitions and their events commit atomically.
- Event history is append-only under ordinary application operation.
- State versions and event sequences preserve their defined ordering.
- Recovery validates persisted identities, versions, schemas, and integrity.
- An uncertain external operation is reconciled before dependent new decisions.
- Database access does not independently grant application-level authority.

Payload protection, encryption, redaction, and retention remain OPEN-006.

### TB-07: Secret and credential boundary

**Participants:** Secret broker, credential sources, provider adapters, executors, and authorized consumers.

**Threat assumptions:** Secrets may leak through logs, arguments, errors, crash reports, environment inheritance, artifacts, or compromised adapters.

**Required properties:**

- Secrets MUST NOT be embedded in agent definitions or immutable manifests.
- Credentials MUST NOT be selected by model output or tool arguments.
- Access is limited to the authorized operation and intended destination.
- Secret values MUST NOT be exposed in ordinary telemetry or error messages.
- Credential use must remain attributable to the authorized operation.
- Missing or invalid credentials MUST fail closed.

The concrete secret-management mechanism is not selected here.

### TB-08: Artifact and evidence boundary

**Participants:** Artifact storage, artifact producers, consumers, evidence export, and CyberCore Agent Evaluation.

**Threat assumptions:** Artifacts may contain malicious content, sensitive information, forged provenance, or content that changes after validation.

**Required properties:**

- Artifacts carry integrity digests, content metadata, sensitivity classification, and producer provenance.
- Content is durable and verified before an authoritative event references it.
- Artifact access is subject to caller and resource authorization.
- Evidence exports preserve provenance and integrity.
- Conductor cannot modify, impersonate, or self-certify independent evaluation results.

### TB-09: External systems and network boundary

**Participants:** Remote APIs, provider services, external tools, network services, and remote executors.

**Threat assumptions:** External systems may be unavailable, compromised, misconfigured, or return incomplete or deceptive results.

**Required properties:**

- Network access is explicit and policy-governed.
- External responses remain untrusted.
- Timeouts do not prove that an operation failed.
- External side effects are represented by durable operation and dispatch records.
- Unknown outcomes are reconciled or preserved as indeterminate.
- Provider fallback MUST NOT silently cross trust or data-handling boundaries.

### TB-10: Independent evaluation boundary

**Participants:** Conductor evidence export and CyberCore Agent Evaluation.

**Required properties:**

- Evidence crosses through a versioned, explicit contract.
- Evaluation remains runtime-independent.
- Findings retain their evaluation identity and provenance.
- Conductor may store and display findings but MUST NOT alter or self-certify them.
- A Run's claimed outcome is not proof of correctness.

## 5. Threat actors

Threat actors are modeled by capability, not by presumed identity or intent. Multiple actors may cooperate or control different parts of one attack.

| ID | Threat actor | Assumed capabilities |
|---|---|---|
| TA-01 | Malicious or compromised caller | Submits crafted inputs, replays commands, attempts unauthorized resource access, or abuses exposed interfaces |
| TA-02 | Prompt-injection content author | Places adversarial instructions in task input, documents, web content, memory, or tool output |
| TA-03 | Malicious or compromised model/provider | Proposes unauthorized actions, emits deceptive content, leaks supplied data, or returns malformed responses |
| TA-04 | Malicious or compromised tool | Misuses granted authority, accesses unintended resources, returns hostile output, or performs unintended side effects |
| TA-05 | Malicious dependency or supply-chain actor | Introduces compromised packages, build artifacts, tool implementations, or update content |
| TA-06 | Local unprivileged process or user | Attempts to access local files, databases, sockets, credentials, or artifacts outside its authority |
| TA-07 | Privileged operator or insider | Misuses legitimate administrative access, changes configuration, or accesses protected data |
| TA-08 | Compromised executor or worker | Misuses execution privileges, forges results, interferes with operations, or attempts to exceed assigned scope |
| TA-09 | Malicious external service | Returns deceptive results, replays requests, delays completion, or creates ambiguous side-effect outcomes |
| TA-10 | Accidental failure or environmental fault | Causes crashes, corruption, clock anomalies, resource exhaustion, duplicate delivery, or partial execution |

The threat model does not assume that every actor can compromise the host operating system, database administrator, or deployment control plane. Those capabilities require explicit deployment-specific analysis.

## 6. Protected assets

| ID | Asset | Security objective |
|---|---|---|
| A-01 | Caller and tenant identity | Prevent impersonation and unauthorized cross-scope access |
| A-02 | Run and Attempt state | Preserve lifecycle integrity, ownership, and attribution |
| A-03 | Policy definitions and decisions | Prevent unauthorized policy changes or forged allow decisions |
| A-04 | Capability grants | Prevent forgery, expansion, replay, or use outside scope |
| A-05 | Approval records | Prevent action substitution, replay, expiry bypass, or unauthorized consumption |
| A-06 | Credentials and secrets | Prevent disclosure and unauthorized use |
| A-07 | Task input and contextual data | Preserve confidentiality, integrity, provenance, and scope |
| A-08 | Memory records | Prevent poisoning, unauthorized access, and cross-scope contamination |
| A-09 | Budget accounts and reservations | Prevent over-allocation and unauthorized resource consumption |
| A-10 | Tool operations and dispatch intents | Preserve authorization, operation identity, and side-effect history |
| A-11 | Events and checkpoints | Preserve ordering, integrity, and recovery correctness |
| A-12 | Artifacts | Preserve confidentiality, integrity, classification, and provenance |
| A-13 | Agent manifests and dependencies | Prevent unauthorized modification and dependency substitution |
| A-14 | Provider and executor credentials/configuration | Prevent unauthorized access and policy bypass |
| A-15 | Evaluation evidence and findings | Preserve independent attribution and integrity |
| A-16 | Runtime availability and host resources | Prevent unbounded execution and resource exhaustion |
| A-17 | Security telemetry and audit history | Preserve useful evidence without leaking protected values |

## 7. Attack surfaces

The following surfaces require validation, authorization, and adversarial testing appropriate to their phase.

| ID | Attack surface | Representative threats |
|---|---|---|
| AS-01 | CLI, API, SDK, and MCP commands | Forged identity, unauthorized access, malformed commands, replay, oversized inputs |
| AS-02 | Task input and conversation context | Prompt injection, malicious serialization, content-based policy manipulation |
| AS-03 | Provider requests and responses | Data disclosure, malformed responses, tool-call manipulation, deceptive usage reports |
| AS-04 | Tool registration and schemas | Capability overstatement, schema confusion, version substitution |
| AS-05 | Tool arguments and results | Injection, path traversal, oversized payloads, secret disclosure |
| AS-06 | Policy and capability evaluation | Scope confusion, stale grants, authorization bypass, fail-open behavior |
| AS-07 | Approval lifecycle | Action substitution, expiry bypass, replay, wrong approver scope |
| AS-08 | Filesystem and process execution | Path traversal, symlink attacks, TOCTOU, ambient credentials, process escape |
| AS-09 | Network and external services | SSRF, unauthorized destinations, response spoofing, ambiguous side effects |
| AS-10 | Memory retrieval and writes | Poisoning, cross-tenant retrieval, unauthorized persistence |
| AS-11 | Database and migrations | Unauthorized reads/writes, corruption, migration abuse, event mutation |
| AS-12 | Recovery and retry paths | Blind redispatch, duplicate side effects, stale authorization, reservation loss |
| AS-13 | Secrets and configuration | Environment leakage, secret exposure, insecure defaults, credential substitution |
| AS-14 | Artifact storage and retrieval | Content substitution, forged provenance, unauthorized disclosure |
| AS-15 | Dependencies and build pipeline | Dependency confusion, compromised releases, malicious build scripts |
| AS-16 | Logs, telemetry, and crash reporting | Secret leakage, sensitive payload exposure, forged or incomplete audit evidence |
| AS-17 | Checkpoints and evidence export | Replay, tampering, identity mismatch, integrity failure |
| AS-18 | Cancellation and concurrency | Race conditions, post-cancellation dispatch, duplicated authorization |
| AS-19 | Budget and resource controls | Resource exhaustion, accounting bypass, concurrent over-allocation |
| AS-20 | Provider fallback and routing | Trust-boundary crossing, policy downgrade, unintended data disclosure |

## 8. Required security invariants

These invariants are mandatory architecture properties. An implementation MUST demonstrate them through tests and review before the affected capability ships.

### SI-01: Runtime-only execution authority

No model, provider, tool, memory record, artifact, or caller-supplied field can independently authorize privileged execution.

### SI-02: Authenticated and scoped caller authority

Every authenticated command is attributable to an identity and authorized against the requested resource and operation. Unknown identity or authorization state is denied.

### SI-03: Runtime-issued capabilities only

Executable authority comes only from valid runtime-issued grants. Grants are scoped to subject, resource, operation, lifetime, provenance, and revocation state. Declarations and requests are not grants.

### SI-04: Exact-action authorization

Authorization is bound to the exact normalized action and its Run, Attempt, operation, tool/version, policy context, executor, and applicable approval and budget reservations. Substitution invalidates authorization.

### SI-05: Dispatch requires durable authorization

No external side effect occurs before the durable dispatch-intent transaction commits with all required authorization and reservation checks.

### SI-06: Current-state revalidation

Dispatch authorization uses current authoritative Run state, cancellation state, policy, capability validity, required approval, and budget availability. Stale prior authorization is insufficient.

### SI-07: Cancellation and dispatch serialize

Cancellation acceptance and dispatch authorization share one authoritative ordering. If cancellation commits first, subsequent dispatch authorization fails. If dispatch commits first, the operation remains recorded as in flight.

### SI-08: Approval cannot override policy

An approval is necessary when policy requires it but is never sufficient by itself. It is bound to the exact action and is checked for scope, expiry, policy version, and one-time consumption.

### SI-09: Untrusted content cannot change policy

Prompt injection and hostile content in inputs, provider responses, memory, documents, artifacts, and tool output cannot alter runtime policy, grant authority, or bypass validation.

### SI-10: Least-privilege execution

Tools and executors receive only the capabilities, resource scope, credentials, and execution permissions required for the authorized operation.

### SI-11: Secrets are not ordinary data

Secrets are not embedded in agent definitions, manifests, ordinary logs, tool arguments, or unprotected artifacts. Secret access is explicit and attributable.

### SI-12: Input and output validation

External inputs and adapter outputs are schema-validated, bounded, and classified before use. Malformed or oversized content is rejected or handled through an explicit policy result, never silently trusted.

### SI-13: Memory is non-authoritative

Memory retrieval cannot grant permissions or override runtime policy. Memory access and writes are scope-checked and policy-controlled.

### SI-14: State and audit atomicity

Every authoritative state transition and its corresponding event are committed atomically. Event history is immutable under ordinary application operation.

### SI-15: Recovery preserves uncertainty

An uncertain external outcome is not silently classified as success or failure. It is reconciled or remains indeterminate. Dependent new decisions remain blocked while unresolved operations could affect them.

### SI-16: No unsafe automatic retry

An operation with an uncertain outcome is not blindly retried. Retry behavior follows its explicit idempotency and reconciliation contract.

### SI-17: Resource exhaustion is bounded

Execution steps, attempts, requests, tool calls, elapsed time, output sizes, concurrency, and other configured resources are bounded by runtime-enforced limits. Untrusted input cannot expand those limits.

### SI-18: Artifact integrity and provenance

An authoritative record cannot reference an artifact until the artifact is durable and integrity-verified. Artifact producer, scope, classification, and digest remain attributable.

### SI-19: Dependency and manifest integrity

Agent manifests pin immutable dependency versions and preserve provenance. Secrets are excluded. Dependency changes cannot silently mutate an existing execution identity.

### SI-20: Isolation claims match enforcement

A backend MUST NOT be described as isolating hostile code unless the required isolation properties are actually implemented and tested. Native code and ordinary local subprocesses are trusted-workload execution only.

### SI-21: Provider neutrality preserves security

Provider adapters cannot weaken authorization, confidentiality, data classification, resource limits, or audit requirements. Unknown provider capabilities are not assumed supported.

### SI-22: Independent evaluation remains independent

Conductor cannot alter, impersonate, or self-certify Agent Evaluation findings. Evidence and findings remain separately attributable.

### SI-23: Security failures are observable

Security-relevant authorization failures, policy denials, capability changes, dispatches, credential-use metadata, integrity failures, and recovery decisions produce attributable security events without exposing secret values.

### SI-24: Fail closed on uncertainty

If a security-critical prerequisite cannot be established, the affected privileged action is blocked. Missing policy, unavailable approval infrastructure, invalid grants, corrupt integrity data, and unknown authorization state never imply permission.

## 9. Phase boundaries and security gates

### 9.1 Phase 1 trusted workload boundary

Phase 1 MUST remain limited to trusted, reviewed workloads.

- Arbitrary model-generated shell execution is disabled by default.
- Untrusted executable plugins are disabled by default.
- Native tools are trusted code, not sandboxed code.
- Ordinary local subprocess execution is not a hostile-code sandbox.
- Approval-required actions remain blocked while the Phase 2 approval workflow is unavailable.
- Recovery MUST NOT blindly redispatch unresolved operations.
- Provider and tool output remain untrusted even when the provider or tool is approved for use.

### 9.2 Later capability gates

The following require separate security review and tests before implementation or release:

- Authenticated multi-user or tenant-facing APIs.
- Persisted approvals and approver identity management.
- Persistent user memory.
- Provider routing and fallback across different data-handling boundaries.
- MCP and third-party plugin integrations.
- Hosted or remote execution.
- Arbitrary or hostile-code execution.
- Containers or VMs making stronger isolation claims.
- Distributed workers, leases, and dispatch fencing.
- Multi-agent delegation.
- Public SDKs and compatibility guarantees.

### 9.3 Unresolved decisions

This threat model does not resolve:

| Decision | Security relevance |
|---|---|
| OPEN-005 — Local identity and authentication | Caller identity, access control, and authenticated APIs |
| OPEN-006 — Event payload protection and retention | Confidentiality, privacy, audit retention, and deletion |
| OPEN-007 — Memory ownership and retention | Memory access, poisoning scope, deletion, and privacy |
| OPEN-010 — Provider data handling and fallback | Data disclosure and cross-provider trust boundaries |

The relevant decision MUST be resolved before the capability that depends on it is implemented or released.

## 10. Threat-to-test traceability

| Security invariant | Required adversarial or failure test |
|---|---|
| SI-01 | Model and tool propose privileged actions without runtime authorization |
| SI-02 | Missing, forged, or out-of-scope caller identity |
| SI-03 | Forged, expired, revoked, or expanded capability grant |
| SI-04 | Change normalized action after authorization |
| SI-05 | Crash before and after dispatch-intent commit |
| SI-06 | Revoke or change policy/capability before dispatch |
| SI-07 | Race cancellation against dispatch authorization in both commit orders |
| SI-08 | Expired, replayed, wrong-scope, or mismatched approval |
| SI-09 | Prompt injection in task input, memory, documents, and tool output |
| SI-10 | Tool attempts access beyond its granted resource scope |
| SI-11 | Canary secrets in logs, errors, arguments, environment, and artifacts |
| SI-12 | Malformed schemas, oversized payloads, and invalid encodings |
| SI-13 | Memory attempts to override policy or cross identity scope |
| SI-14 | Crash during state/event transaction and attempted event mutation |
| SI-15 | Timeout or crash after external transmission with unknown outcome |
| SI-16 | Uncertain non-idempotent operation followed by retry request |
| SI-17 | Boundary and over-limit tests, concurrent resource allocation, and exhaustion |
| SI-18 | Tamper with staged artifact before finalization or reference |
| SI-19 | Change dependency content without changing pinned identity |
| SI-20 | Verify that unsupported isolation guarantees are not claimed or enabled |
| SI-21 | Provider returns unsupported capability, malicious output, or changed routing target |
| SI-22 | Attempt to alter or impersonate independent evaluation findings |
| SI-23 | Verify durable security events and ensure sensitive values are redacted |
| SI-24 | Remove or corrupt each required authorization prerequisite and verify denial |

Security tests MUST verify externally observable outcomes and durable evidence, not merely internal helper return values.

Ordinary CI MUST use deterministic fake providers and tools. Live provider credentials and production services MUST NOT be required for ordinary security tests.

## 11. Explicitly out of scope

This document does not select or implement:

- An authentication protocol or identity provider.
- A tenant model or authorization product.
- A secret manager or key-management system.
- Encryption algorithms, key rotation, or payload-retention rules.
- A hostile-code sandbox implementation.
- A network firewall or egress-control implementation.
- A provider allowlist or fallback policy.
- A security monitoring product or incident-response platform.
- A certification or compliance framework.
- Additional profiles and override policies remain outside this document.

These require their own decisions, contracts, and implementation gates.

## 12. Review and acceptance criteria

This document is ready for acceptance when review confirms:

- Trust boundaries reflect the accepted Architecture Specification.
- Threat actors and assumptions are explicit and appropriately scoped.
- Protected assets and attack surfaces cover all Phase 1 execution paths.
- Security invariants do not conflict with accepted ADRs or domain contracts.
- Phase 1 restrictions are distinguishable from later-phase requirements.
- Open decisions remain open and are not silently resolved here.
- Each invariant has a meaningful adversarial or failure test.
- No statement implies that an unimplemented control already exists.

Acceptance establishes the security architecture baseline. It does not authorize implementation of deferred capabilities.
