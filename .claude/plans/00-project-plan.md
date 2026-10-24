# charlie-work: plan

The thankless, invisible work that keeps the bar running. This service listens for production errors, hands each one to an AI agent that reproduces it with a failing test, fixes it, and opens a PR for a human to review. It never merges.

## Goals

1. Listen for errors from many sources. Define a clear interface which can be implemented by each connector (start with GCP Cloud Logging; Sentry etc. later).
2. Categorize each reported error into an incident. The first instance of it should create an incident, future ones should ++ an existing incident. Incidents will need to be combined and managed over time. Incidents need to be stored somewhere. The source error data could themselves be the incidents and the store (i.e. PagerDuty incidents). So the incident store itself will be an interface with many connectors. Multiple incident stores should be able to be used at the same time.
3. For each open incident, run an agent that captures the error, writes a failing test, patches the code, and proves the test passes.
4. Give the agent MCP servers to diagnose (GCP logs, GitHub, read-only data sources).
5. File a PR.
6. Hard cost controls so we never spend unbounded tokens.
7. Portable: ships as a Docker image that runs on any serverless container host (Cloud Run first).

### Proposed additional goals

Marked for discussion in the PR. None need to be built perfectly at the start.

8. **Human release gate.** Collection can run at volume while an agent run is released one at a time by hand. Needed for development and testing, and a useful safety valve in production.
9. **Runs on the plans we already pay for.** Must work with a Claude Team plan or Google Workspace (Gemini) entitlement, not only a metered API key. See Open questions.
10. **Auditable.** Every agent run records what it was given, what it did, what it cost, and what came of it (PR, failure, abandoned).
11. **Scrubbed inputs.** Known-sensitive fields never leave our boundary toward the agent.
12. **Outcome feedback.** Track whether each PR merged, was closed, or was ignored, so prompts and filters can be tuned on evidence.

### Principle

A single working implementation beats a clean interface with zero implementations. Interfaces (sources, incident stores, agent runtimes) are introduced when the first implementation is done and a second is genuinely planned, not before.

## Decisions so far

| Decision | Choice | Why |
|---|---|---|
| Name | `charlie-work` | Fits unglamorous bug fixing. |
| Language | Rust | Portable static binary, good fit for the dispatcher's I/O and state logic, and the author wants to learn it. |
| Agent runtime | **Open** (see Open questions) | Managed Agents is technically attractive but is API-billed; we need something that works with a Claude Team plan or Google Workspace. |
| First deploy target | GCP Cloud Run | Existing infra; scale-to-zero. |
| Merge policy | Open PRs only, human merges | The agent token cannot merge. |

Previously rejected, now to be revisited because of the billing constraint: shelling out to the `claude` CLI / Agent SDK (needs our own sandbox and a heavier image, but can use a subscription login). Still rejected: off-the-shelf tools (Sentry Seer, Better Stack AI SRE are tied to their own error sources; the open-source options found, rutvej/DAA and GabsFranke/claude-code-github-agent, don't match our ingestion path). GitHub Actions + `claude-code-action` remains a viable fallback with less control over ingestion and budgets.

## Architecture

### First slice (build this first)

```
Cloud Logging (severity >= ERROR)
   -> log sink -> Pub/Sub push subscription
      -> charlie-work (Rust, Cloud Run)
           1. ingest + parse into an ErrorEvent
           2. enqueue
           3. release to agent, only when released and within budget
                 - automatic when enabled
                 - manual "release one" for development/testing
           4. agent works the item; record outcome
      -> agent runtime (TBD) -> GitHub: branch + PR
```

No filtering, fingerprinting, or dedupe in this slice. The queue can grow large; releasing is the only thing that spends money. Manual "release one" needs a control surface: start with an authenticated endpoint / CLI command, and a small UI only if that proves too awkward.

### Later (deliberately deferred)

Noise filters and severity/count thresholds, fingerprinting, incident creation and merging, multiple incident stores, second error source, persistent store, per-session interrupts. See plans 07 and 08.

The service answers the Pub/Sub push quickly (ack), and does agent work asynchronously so Pub/Sub does not redeliver.

### Components (single crate, subcommands later if needed)

- `source/`: error source adapters producing `ErrorEvent`. First: Pub/Sub push payload from a Cloud Logging sink.
- `queue/`: pending error events and their release state.
- `budget/`: spend and run-count accounting; the release gate consults it.
- `agent/`: thin client for whichever runtime is chosen.
- `store/`: state persistence behind a trait (in-memory first, then Firestore or Postgres).
- `config`: one typed config struct, loaded from a file plus env.
- Later: `triage/` (filters, fingerprints) and `incident/` (incident stores).

## Cost controls

Enforced in the service:

- Release gate: nothing reaches an agent unless released (manually or by an enabled automatic policy) and within budget.
- Global caps: max sessions per day and a monthly dollar budget (or usage-equivalent, if on a subscription). Over budget means leave queued and log.
- Per-session cap: watch usage and interrupt when a session exceeds its limit, if the chosen runtime exposes the data.
- Model choice per run in config (cheaper model first, optional escalation).
- Later: per-incident cooldown, minimum severity/count.

## Safety

- Fine-grained GitHub token scoped to specific repos: push branches and open PRs; no merge, no admin.
- Restricted network access for the agent where the runtime allows it.
- Diagnostic MCPs are read-only.
- Agent output is untrusted input: PRs go through normal review and CI on the target repo.
- If a hosted runtime is used, sessions may be stored server-side and may not be eligible for Zero Data Retention or HIPAA BAA. Be deliberate about what log content is forwarded; scrub known-sensitive fields before sending.

## Open questions

- **Agent runtime and billing.** This must work with a Claude Team plan or Google Workspace (Gemini), not a new metered subscription. Candidates, all unverified:
  - Claude Managed Agents (REST, beta; Console API key, header `managed-agents-2026-04-01`). The docs read so far do not say it is covered by a Team/Pro/Max plan, and API usage is billed separately. Likely does not meet the constraint.
  - Claude Code / Agent SDK run headless, authenticated with the Team plan. Needs our own sandbox. May need to run on a laptop rather than Cloud Run. Whether unattended automation is allowed under the plan's terms must be confirmed.
  - Gemini CLI / agent tooling under the Workspace entitlement.
  - GitHub Actions + `claude-code-action`.
  A laptop-hosted runner is acceptable if it is the only way to use an existing plan, which makes "portable to Cloud Run" a later goal rather than a first-slice constraint. Resolved by the spike in plan 04.
- Can the agent read the codebase itself, or does the dispatcher have to send it? Is there a tool the runtime provides for the agent to request information about the codebase?
- State store: Firestore vs a small Postgres (existing Supabase?).
- Where does the agent get repo-specific guidance? Repos that are checked out load `.claude/skills` from their root, so each target repo can carry its own conventions.

## Build order

1. Repo bootstrap: Cargo project, `rustfmt`/`clippy` config, Dockerfile (multi-stage, static binary), README.
2. **CI** (GitHub Actions; `main` is already protected so everything lands via PR):
   - `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`
   - Docker build on every PR (no push)
   - Dependabot / `cargo audit`
3. Skeleton service: `axum` server, `/healthz`, typed config, structured logging.
4. `ErrorEvent`, Pub/Sub source, and an in-memory queue, with fixture-based tests.
5. Agent runtime spike: decide the runtime against the billing constraint; then a client plus a fake for tests.
6. Release gate and budget: manual "release one", caps, then wire to the runtime end to end against a sandbox repo with a seeded bug.
7. Production: persistent store, log sink, Pub/Sub, Cloud Run, deploy-on-merge.
8. Later: triage (filters, fingerprints), incidents and incident stores, second source (Sentry), per-session interrupts, prompt iteration on real incidents.

## References

- Managed Agents overview: https://platform.claude.com/docs/en/managed-agents/overview
- Accessing GitHub from an agent: https://platform.claude.com/docs/en/managed-agents/github
- Sessions: https://platform.claude.com/docs/en/managed-agents/sessions
- Events and streaming: https://platform.claude.com/docs/en/managed-agents/events-and-streaming
- Reference (events, rate limits): https://platform.claude.com/docs/en/managed-agents/reference
