# Plan 02: Skeleton service

> A runnable axum service with typed config, structured logging and `/healthz`.

Parent: [00-project-plan.md](00-project-plan.md) (build order step 3). Depends on Plan 01.

## PRs

1. **Typed config.** One `Config` struct (serde), loaded from a file plus env overrides, with validation and tests. Config schema/defaults are excluded from the LoC budget.
2. **HTTP server.** `axum` app on `PORT`, `/healthz`, graceful shutdown on SIGTERM (Cloud Run), `tracing` with JSON output shaped for Cloud Logging (`severity` field).

## Questions

1. Config crate: `figment`, `config`, or hand-rolled serde + env? **Recommendation:** `figment` (file + env layering, typed errors).
2. Config file format: TOML? **Recommendation:** TOML.
3. Secrets (API keys, GitHub token): env only, never in the file? **Recommendation:** env only (Cloud Run maps Secret Manager to env).
4. Error handling shape: `anyhow` at the edges + `thiserror` for typed library errors, with a single global axum error handler (per your exception-handling rules)? Confirm.
5. Log format: JSON always, or pretty locally and JSON when `K_SERVICE` is set?

Answers:
