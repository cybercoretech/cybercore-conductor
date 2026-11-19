# ADR-025 — Rust MSRV and Pinned Development Toolchain

**Status:** Accepted

## Decision

The project's initial Minimum Supported Rust Version (MSRV) is Rust `1.85.0`.

The development toolchain is pinned to Rust `1.100.0`, using the stable
release channel. The pinned toolchain includes the `rustfmt` and `clippy`
components.

Continuous integration must validate both:
- The declared MSRV (`1.85.0`).
- The exact pinned development toolchain (`1.100.0`).

Dependencies must remain compatible with the declared MSRV. If dependency
requirements make that impossible, the MSRV must be deliberately raised and
the decision documented before changing the project policy.

## Rationale

Rust `1.85.0` is the minimum compiler version supporting the Rust 2024 edition.
A separate pinned development toolchain provides reproducible local development
and CI behavior without making the development compiler the project's minimum.

## Consequences

- The workspace manifest declares `rust-version = "1.85"`.
- `rust-toolchain.toml` pins `1.100.0` and the required components.
- CI tests the MSRV and pinned toolchain separately.
- Dependency updates must be checked for MSRV compatibility.
- Raising the MSRV requires an explicit decision and corresponding documentation.
