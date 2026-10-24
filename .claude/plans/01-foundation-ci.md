# Plan 01: Foundation and CI

> Bootstrap a Rust crate with lint, test and Docker checks so every later PR lands green.

Parent: [00-project-plan.md](00-project-plan.md) (build order steps 1-2, minus deploy).

## PRs

1. **Cargo project.** `cargo init`, `rustfmt.toml`, clippy config, minimal README (what it is, how to build/test). Deployable: trivially (nothing runs).
2. **CI: checks.** GitHub Actions running `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`, with cargo caching.
3. **Dockerfile + CI build.** Multi-stage, static binary, runs as non-root. CI builds on every PR, no push.
4. **Supply chain.** Dependabot config (cargo, actions, docker) and a `cargo audit` job.

Push to Artifact Registry / Cloud Run deploy is deliberately deferred to Plan 06 (needs infra).

## Questions

1. Static target: `musl` + `scratch`/`distroless/static`, or `gnu` + `distroless/cc`? (musl is simplest to make portable, but some crates, e.g. TLS/allocator, are slower or fiddly.) **Recommendation:** musl + distroless/static, use `rustls` everywhere.
2. Rust toolchain pinning: `rust-toolchain.toml` with a fixed stable version, or float on stable? **Recommendation:** pin, let Dependabot/manual bumps move it.
3. Should `cargo audit` fail the build or only warn on advisories? **Recommendation:** fail on PRs, plus a scheduled run.
4. Is a required status check list on `main` something you'll configure, or should a PR include it (e.g. via `gh api`/settings-as-code)? 
5. Edition / MSRV policy: latest edition, no MSRV promise?

Answers:
