# Plan 04: Managed Agents client

> A typed Managed Agents client with a test fake, plus the agent and environment definitions kept in the repo as config.

Parent: [00-project-plan.md](00-project-plan.md) (build order step 6). Depends on Plan 02 (not on Plan 03; can run in parallel).

## PRs

1. **Spike (docs only).** Answer the open questions in the parent plan against the live docs/API: billing coverage, native per-session limits, usage fields on the stream, MCP reachability from the sandbox. Output is a short findings note, written as an addendum plan (parent plan is never edited).
2. **Trait + fake.** `AgentClient` trait (create session, send message, stream events, interrupt, delete) and an in-memory fake for tests.
3. **HTTP impl, part 1.** `reqwest` client with beta header, auth, create session + send `user.message`, typed errors, tests against a mock server.
4. **SSE stream.** Parse the event stream into typed events including usage; reconnect handling; fixture tests.
5. **Definitions + provisioning.** Agent (model, system prompt, tools, MCP servers) and environment (`allowed_hosts`) as files in the repo, plus a script that applies them. Mostly config, so it is cheap on the LoC budget.

## Questions

1. May I read the Managed Agents docs (the five URLs in the parent plan) for the spike? Network access will be needed.
2. Do you already have a Console API key and spend limit set? PR 1 may need a real call; I will not make paid calls without your go-ahead.
3. Provisioning mechanism: `ant` CLI definitions, a small shell script with `curl`, or a Rust subcommand? **Recommendation:** whichever the spike shows is supported; fall back to a script.
4. Should the system prompt live in the repo as a markdown file loaded at provisioning time? **Recommendation:** yes.
5. The Rust client is hand-written against a beta API. Do you want response structs lenient (ignore unknown fields) so API drift doesn't break us? **Recommendation:** lenient.

Answers:
