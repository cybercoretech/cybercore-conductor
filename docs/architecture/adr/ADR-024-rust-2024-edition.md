# ADR-024 — Rust 2024 Edition

**Status:** Accepted

## Decision

The CyberCore Conductor workspace uses the Rust 2024 edition.

All workspace crates inherit the edition from the workspace root unless a
specific compatibility requirement is documented and approved.

## Rationale

The project is a new Rust codebase. Establishing the current language edition
at the workspace boundary avoids inconsistent crate defaults and makes
language-version expectations explicit.

## Consequences

- The workspace manifest declares `edition = "2024"`.
- Contributors and CI must use a compiler supporting Rust 2024.
- Edition migration or replacement requires an explicit architectural decision.
