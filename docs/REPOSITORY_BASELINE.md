# Repository Baseline

**Project:** CyberCore Conductor (provisional working name)
**Repository:** `https://github.com/cybercoretech/cybercore-conductor`
**Intended local workspace:** `/home/raven/Devspace/Cybercore/tooling/cybercore-conductor`
**Baseline record:** 2026-11-18
**Status:** Documentation bootstrap; runtime not yet started

## 1. Confirmed observations

The GitHub contents API returned `404` with the explicit message `This repository is empty` when the repository's `README.md` and `Cargo.toml` were requested. This establishes that no readable root README or Cargo workspace manifest existed at the time of inspection.

An attempt to create the initial README through the connected GitHub integration was rejected with HTTP `403` (`Resource not accessible by integration`). Therefore this documentation package has been prepared for placement in the repository, but has **not** been committed or pushed.

## 2. Not verified

The connected integration did not provide a successful repository metadata or branch listing. The following must be verified from a local clone or an appropriately authorized GitHub session before treating them as facts:

- default branch name and current branch HEAD;
- whether the repository has any commits, tags, releases, or non-default refs;
- repository visibility and GitHub settings;
- branch protection, required checks, Actions permissions, and security settings;
- local workspace contents and whether the intended local directory already exists;
- whether another collaborator has initialized the repository since this inspection.

Do not infer that the default branch is `main`, that the working tree is clean, or that no remote changes exist.

## 3. Initial documentation layout

This bootstrap package defines the following initial layout:

```text
cybercore-conductor/
├── README.md
└── docs/
    ├── REPOSITORY_BASELINE.md
    └── architecture/
        └── ARCHITECTURE_SPEC_v0.1.md
```

This is intentionally documentation-only. It does not introduce a Cargo workspace, Rust crates, CI workflow, license, code formatting policy, or runtime implementation. Those should be added after the repository metadata and project policy are confirmed.

## 4. Project boundary

CyberCore Conductor is the autonomous agent runtime and orchestration platform. It is not the evaluation workbench and is not a wrapper around a single model provider or CLI.

- Conductor executes and governs agent runs.
- CyberCore Agent Evaluation independently evaluates runs, verifies claims, and manages evidence.
- AgentForge remains a separate project, with integration only through explicit contracts.

## 5. Baseline acceptance checklist

Before implementation begins:

- [ ] Confirm repository owner/name and canonical remote URL.
- [ ] Confirm default branch, HEAD SHA, tags, and working-tree status.
- [ ] Confirm local workspace path and inspect its contents.
- [ ] Add this baseline record and the architecture specification to the repository.
- [ ] Review and accept or amend the v0.1 architecture decisions.
- [ ] Confirm the project name and license strategy before public package publication.
- [ ] Establish Rust toolchain, workspace conventions, CI, formatting, linting, and test policy in a separate implementation-bootstrap change.

## 6. Change control

This file records the observed starting point, not a guarantee about the repository's future state. Update it when the repository is initialized or when a material baseline fact changes. Architecture changes belong in decision records or a versioned specification update; do not silently rewrite accepted invariants.
