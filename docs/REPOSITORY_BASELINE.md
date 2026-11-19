# Repository Baseline

**Project:** CyberCore Conductor (provisional working name)
**Repository:** `https://github.com/cybercoretech/cybercore-conductor`
**Intended local workspace:** `/home/raven/Devspace/Cybercore/tooling/cybercore-conductor`
**Baseline record:** 2026-11-18
**Status:** Phase 0 repository foundation established; runtime implementation not yet started

## 1. Repository identity and history

The canonical repository is:

`cybercoretech/cybercore-conductor`

The local clone is located at:

`/home/raven/Devspace/Cybercore/tooling/cybercore-conductor`

The following facts were verified from the local Git repository:

| Item | Verified value |
|---|---|
| Current branch | `main` |
| Current HEAD | `ba5bd6a` |
| Tracked remote branch | `origin/main` |
| `origin/main` at last local verification | `44bbd24` |
| Local branch relationship | One commit ahead of `origin/main` |
| Working tree at last verification | Clean before the current Phase 0 changes |

Known commit history:

- `65ebc89` — Initial repository commit.
- `44bbd24` — `docs: add repository baseline and architecture specification`.
- `ba5bd6a` — `docs: establish conductor architecture v0.2 baseline`.

The current Phase 0 repository foundation changes described below are local working-tree changes. They have not been committed or pushed.

The local `origin/main` reference is the value observed during the last local Git inspection. It does not establish that the remote has not changed since then.

## 2. Historical bootstrap observations

During the original bootstrap, the connected GitHub contents API returned `404` with the explicit message `This repository is empty` when the repository's `README.md` and `Cargo.toml` were requested.

An attempt to create the initial README through the connected GitHub integration was rejected with HTTP `403` (`Resource not accessible by integration`).

These observations describe the earlier integration attempt. They are not the current local repository state: the repository has since been initialized, committed, and cloned locally.

## 3. Current local repository state

The following local repository foundation has been created.

### Root configuration

- `Cargo.toml` — virtual workspace manifest.
- `Cargo.lock` — generated workspace lockfile.
- `rust-toolchain.toml` — pins Rust `1.100.0`.
- `rustfmt.toml` — Rust 2024 formatting edition.
- `.gitignore` — excludes build artifacts, local environment files, and common editor/OS files.
- `LICENSE` — MIT License, copyright `Cybercore Tech`, 2026.

### Initial crate

`crates/conductor-core/`

- `Cargo.toml` — workspace-inherited package configuration.
- `src/lib.rs` — initial library entry point.
- No external dependencies have been introduced.
- Unsafe code is forbidden in this crate.

### Continuous integration

`.github/workflows/ci.yml` defines a Rust CI workflow with separate toolchain jobs for:

- Rust `1.85.0` — declared MSRV.
- Rust `1.100.0` — pinned development toolchain.

Each job checks formatting, workspace compilation, and Clippy with warnings denied.

The workflow exists locally but has not yet been executed by GitHub Actions.

### Architecture documentation

- `docs/architecture/ARCHITECTURE_SPEC_v0.1.md`
- `docs/architecture/ARCHITECTURE_SPEC_v0.2.md`
- `docs/architecture/ADR_INDEX.md`
- `docs/architecture/adr/ADR-024-rust-2024-edition.md`
- `docs/architecture/adr/ADR-025-rust-msrv-and-toolchain.md`
- `docs/architecture/adr/ADR-026-mit-license.md`

The original ADR index backup created during the ADR update is retained locally as:

`docs/architecture/ADR_INDEX.md.pre-ADR-024-026.bak`

It is a temporary recovery artifact, not an architecture document or intended permanent project file.

## 4. Toolchain and validation baseline

The accepted Rust policy is:

| Policy | Value |
|---|---|
| Rust edition | 2024 |
| MSRV | 1.85.0 |
| Pinned development toolchain | 1.100.0 stable |
| Cargo resolver | 3 |
| License | MIT |
| Copyright holder | Cybercore Tech |

The local machine has Rust `1.100.0`, Cargo, rustfmt, and Clippy installed for the pinned toolchain. Rust `1.85.0` is also installed for MSRV validation.

The initial workspace passed these local checks:

- `cargo +1.85.0 check --workspace --all-targets`
- `cargo check --workspace --all-targets` using the pinned `1.100.0` toolchain
- `cargo fmt --all -- --check`
- `cargo clippy --workspace --all-targets -- -D warnings`
- `git diff --check` for the changes inspected at the time

These are local validation results. They do not establish that GitHub Actions has run successfully.

## 5. Project boundary

CyberCore Conductor is the autonomous agent runtime and orchestration platform. It is not the evaluation workbench and is not a wrapper around a single model provider or CLI.

- Conductor executes and governs agent runs.
- CyberCore Agent Evaluation independently evaluates runs, verifies claims, and manages evidence.
- AgentForge remains a separate project, with integration only through explicit contracts.

The governing principle is:

**A model may propose an action; only the runtime may authorize and execute it.**

The architecture specification and accepted decision records define the detailed boundaries, invariants, and implementation gates.

## 6. Architecture decision status

Architecture Specification v0.2 and ADR-001 through ADR-026 establish the current documented architecture baseline.

ADR statuses distinguish accepted direction from implementation-gated and unresolved decisions. An accepted architecture decision does not automatically authorize implementation of every capability it describes.

In particular:

- Approval-required actions remain blocked in the initial Phase 1 scope.
- Phase 1 begins with trusted workloads and does not claim to provide hostile-code isolation.
- Persistence, recovery, dispatch authorization, and evidence contracts must preserve the invariants defined by the architecture.
- CyberCore Agent Evaluation remains an independent evaluation boundary.

The active Open Decision Register in `docs/architecture/ADR_INDEX.md` remains authoritative for unresolved decisions.

## 7. Not yet verified

The following GitHub-side facts have not been verified as part of this baseline update:

- Repository visibility and current default branch configuration.
- Branch protection or rulesets.
- Required status checks.
- GitHub Actions permissions and organization-level restrictions.
- Dependabot, secret scanning, code scanning, and other security settings.
- Whether the remote branch has changed since the last local inspection.
- Whether the newly created CI workflow runs successfully on GitHub.

Do not infer these facts from the local clone.

## 8. Current implementation status

Phase 0 repository foundation work is underway.

Completed locally:

- Repository identity and local Git history inspected.
- Architecture Specification v0.2 established.
- ADR-024 through ADR-026 recorded.
- MIT license added.
- Rust workspace and initial `conductor-core` crate created.
- Rust 2024, MSRV, and pinned toolchain configuration established.
- Initial local MSRV and pinned-toolchain checks passed.
- Initial CI workflow created.

Not yet completed:

- Final repository baseline and Phase 0 audit.
- Resolution of remaining Phase 0 open decisions.
- Domain contract definitions and state-machine specifications.
- Persistence schema and transaction-boundary design.
- Invariant-to-test matrix.
- Ordered Phase 1 implementation backlog.
- GitHub-side CI execution and repository settings review.

No runtime implementation or provider integration has begun.

## 9. Change control

This file records the verified repository and implementation baseline as of its update date. It must be updated when material repository facts or project status change.

Architecture changes belong in decision records or a versioned specification update. Do not silently rewrite accepted architectural invariants.

Local validation must not be described as successful remote CI. Local commits must not be described as pushed or published unless independently verified.
