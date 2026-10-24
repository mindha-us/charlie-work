# charlie-work: plan

Charlie Work: the thankless, invisible work that keeps the bar running. This service listens for production errors, hands each one to a Claude Managed Agent that reproduces it with a failing test, fixes it, and opens a **draft PR** for a human to review. It never merges.

## Goals

1. Listen for errors from many sources (start with GCP Cloud Logging; Sentry etc. later).
2. For each accepted error, run an agent that captures the error, writes a failing test, patches the code, and proves the test passes.
3. Give the agent MCP servers to diagnose (GCP logs, GitHub, read-only data sources).
4. File a draft PR.
5. Hard cost controls so we never spend unbounded tokens.
6. Portable: ships as a Docker image that runs on any serverless container host (Cloud Run first).

## Decisions so far

| Decision | Choice | Why |
|---|---|---|
| Name | `charlie-work` | Always Sunny reference; fits unglamorous bug fixing. |
| Language | Rust | Portable static binary, good fit for the dispatcher's I/O and state logic, and the author wants to learn it. |
| Agent runtime | **Claude Managed Agents** (REST, beta) | Anthropic hosts the harness and sandbox. No runner image with Node/git/gh to build. Called via `reqwest`; no Rust SDK needed. |
| First deploy target | GCP Cloud Run | Existing infra; scale-to-zero. |
| Merge policy | Draft PRs only, human merges | The agent token cannot merge. |

Rejected: shelling out to the `claude` CLI in a Cloud Run Job (needs our own sandbox and a heavier image); GitHub Actions + `claude-code-action` (viable fallback, less control over ingestion and budgets); off-the-shelf tools (Sentry Seer, Better Stack AI SRE are tied to their own error sources; the open-source options found, rutvej/DAA and GabsFranke/claude-code-github-agent, don't match our ingestion path).

## Architecture

```
Cloud Logging (severity >= ERROR)
   -> log sink -> Pub/Sub push subscription
      -> charlie-work (Rust, Cloud Run)
           1. parse + normalize into an ErrorEvent
           2. filter (noise rules, min severity/count)
           3. fingerprint (type + message template + top frames)
           4. dedupe / cooldown / budget check against state store
           5. create Managed Agent session (repo mounted, GitHub token)
           6. send first user.message with the error context
           7. watch the event stream; enforce budget; record outcome
      -> Anthropic Managed Agents API
           agent: model, system prompt, MCP servers, tools
           environment: cloud sandbox (restricted allowed_hosts)
           session: one per error fingerprint
      -> GitHub: branch + draft PR
```

The service answers the Pub/Sub push quickly (ack), and does session management asynchronously so Pub/Sub does not redeliver.

### Components (single crate, subcommands later if needed)

- `source/`: error source adapters producing `ErrorEvent`. First: Pub/Sub push payload from a Cloud Logging sink. Trait-based so Sentry webhooks etc. slot in.
- `triage/`: filtering rules, fingerprinting, dedupe/cooldown.
- `budget/`: spend and run-count accounting.
- `agent/`: thin typed client for the Managed Agents endpoints (agents, environments, sessions, events, SSE stream).
- `store/`: state persistence behind a trait (Firestore first; in-memory for tests).
- `config`: one typed config struct, loaded from a file plus env.

## Cost controls

Enforced in the service (the first two are cheap and prevent most runaway cost):

- Per-fingerprint cooldown: at most one attempt per fingerprint per N hours; never while a PR for it is open.
- Global caps: max sessions per day and a monthly dollar budget. Over budget means skip and log, not queue.
- Per-session cap: watch usage on the event stream and send an interrupt when a session exceeds its token/cost/time limit. **Open question:** whether the API offers a native per-session limit; check the sessions and reference docs.
- Model choice per run in config (cheaper model first, optional escalation).
- Minimum error severity/count before an agent is started.

## Safety

- Fine-grained GitHub token scoped to specific repos: push branches and open PRs; no merge, no admin.
- Restricted environment networking via `allowed_hosts`.
- Diagnostic MCPs are read-only.
- Agent output is untrusted input: PRs are drafts and go through normal review and CI on the target repo.
- Managed Agents sessions are stored server-side and are not eligible for Zero Data Retention or HIPAA BAA. Be deliberate about what log content is forwarded; scrub known-sensitive fields before sending. Sessions can be deleted via the API.

## Open questions

- **Billing.** Managed Agents is an API product (Claude Console API key, beta header `managed-agents-2026-04-01`). The docs I read do not say it is covered by a Pro/Max subscription, and API usage is billed separately from those plans. Confirm pricing in the Console before relying on it, and set a Console spend limit.
- Does the sessions API support a native token/cost/turn limit? Which usage fields appear on the stream?
- Can the sandbox reach a GCP logs MCP? Options: public MCP behind auth, a self-hosted environment, MCP tunnels (research preview), or have the dispatcher include the relevant log lines in the first message (simplest; start here).
- State store: Firestore vs a small Postgres (existing Supabase?).
- Where does the agent get repo-specific guidance? Mounted repos load `.claude/skills` from their root, so each target repo can carry its own conventions.

## Build order

1. Repo bootstrap: Cargo project, `rustfmt`/`clippy` config, Dockerfile (multi-stage, static binary), README.
2. **CI** (GitHub Actions; `main` is already protected so everything lands via PR):
   - `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`
   - Docker build on every PR (no push)
   - On merge to `main`: build and push the image to Artifact Registry; deploy to Cloud Run once infra exists
   - Dependabot / `cargo audit`
3. Skeleton service: `axum` server, `/healthz`, typed config, structured logging.
4. `ErrorEvent` model and Pub/Sub source with fixture-based tests.
5. Triage: filters, fingerprinting, dedupe against the in-memory store, with tests.
6. Managed Agents client plus a fake for tests; create the agent and environment (script or `ant` CLI definition kept in the repo).
7. Wire it end to end against a sandbox repo with a seeded bug.
8. Budget enforcement and interrupt on overspend.
9. Firestore store, Terraform or gcloud scripts for the log sink, Pub/Sub, Cloud Run.
10. Second source (Sentry webhook), then iterate on the agent prompt using real incidents.

## References

- Managed Agents overview: https://platform.claude.com/docs/en/managed-agents/overview
- Accessing GitHub from an agent: https://platform.claude.com/docs/en/managed-agents/github
- Sessions: https://platform.claude.com/docs/en/managed-agents/sessions
- Events and streaming: https://platform.claude.com/docs/en/managed-agents/events-and-streaming
- Reference (events, rate limits): https://platform.claude.com/docs/en/managed-agents/reference
