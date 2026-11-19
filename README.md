# CyberCore Conductor

> **Working project name:** provisional; not legally or commercially cleared.

CyberCore Conductor is the planned provider-independent autonomous agent runtime and orchestration platform in the CyberCore ecosystem.

Its purpose is to run agents as durable, observable, policy-governed processes: accepting tasks, assembling context, interacting with model providers, requesting tools, enforcing permissions and budgets, handling approvals, recording evidence, and recovering from interruption.

## Project status

**Architecture-first / pre-implementation.** The initial repository documentation and Architecture Specification v0.1 define the design before runtime code is introduced. No production-readiness claim is made.

- [Repository baseline](docs/REPOSITORY_BASELINE.md)
- [Architecture Specification v0.1](docs/architecture/ARCHITECTURE_SPEC_v0.1.md)

## Design principles

1. The runtime, not the model, owns authority. Model output is a proposal; policy and runtime services decide what may execute.
2. Runs are durable and observable. State transitions and side effects must be recorded so interruption does not erase operational history.
3. Provider and execution backends are adapters behind stable, provider-independent contracts.
4. Memory and retrieved content are untrusted data, never a source of policy or elevated authority.
5. Verification is independent. Conductor does not certify its own correctness or task success.
6. Security boundaries are enforced at execution and service layers, not merely in prompts, interfaces, or documentation.

## Relationship to other CyberCore projects

- **CyberCore Conductor** owns agent execution, planning loops, tool dispatch, memory access, orchestration, approvals, budgets, and agent lifecycle.
- **CyberCore Agent Evaluation** independently evaluates behavior, verifies outcomes, and stores evidence. It must remain usable with agents other than Conductor.
- **AgentForge** remains a separate coding-agent orchestration project; integration may be considered through explicit interfaces.

## Planned implementation direction

The runtime core will be written in Rust. The initial implementation should favor a small number of cohesive crates and a local-first deployment, with durable persistence and explicit adapter boundaries. HTTP APIs, SDKs, MCP, dashboards, distributed workers, and remote execution are phased capabilities, not assumed initial deliverables.

## Naming and compatibility

The repository name and working product name are provisional. "Conductor" is already used by established workflow/orchestration software. Do not publish packages, domains, marketing assets, or compatibility claims under this name until naming clearance is complete.

## Development

There is no runtime implementation or build/test workflow in this baseline. Architecture decisions are recorded in the specification before implementation begins.
# cybercore-conductor
