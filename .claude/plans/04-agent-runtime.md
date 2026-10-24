# Plan 04: Agent runtime

> Decide which agent runtime works with the plans we already have (Claude Team, Google Workspace), then build a small client for it with a test fake.

Parent: [00-project-plan.md](00-project-plan.md) (open question on runtime and billing). Depends on Plan 02; can run in parallel with Plan 03. This plan was previously "Managed Agents client"; the billing constraint made the runtime itself the first thing to settle.

## PRs

1. **Spike (docs only).** Compare the candidates against the constraint and write the result as an addendum note in this plan's directory. For each of: Claude Managed Agents, Claude Code / Agent SDK headless on the Team plan, Gemini CLI under Workspace, GitHub Actions + `claude-code-action`:
   - Does it bill against an existing plan or a new metered account?
   - Are unattended/automated runs permitted under the plan's terms?
   - Where must it run (laptop, Cloud Run, hosted)?
   - Can it read the codebase, take MCP servers, and open PRs?
   - What usage data does it expose for budgeting and interrupts?
2. **Trait + fake.** `AgentRuntime` trait (start run with an error context, observe completion, report usage if available, cancel) and an in-memory fake. Shape follows what the spike shows is common, not the first candidate.
3. **First implementation.** The chosen runtime, with the smallest useful surface: start a run, wait for the outcome, surface the PR URL. May split into two PRs depending on the runtime.
4. **Definitions as config.** Agent prompt, tools, MCP servers, and (if applicable) sandbox network rules as files in the repo, plus any provisioning script.

## Questions

1. May I research the candidates' docs for the spike? Network access is needed.
2. Is running the worker on your laptop acceptable for the first version, if that is the only way to use the Team plan? **Recommendation:** yes. It makes Cloud Run portability a later goal.
3. Are there constraints on the Gemini side, e.g. which Workspace edition and whether agent/CLI access is included?
4. I will not make any paid or plan-consuming calls during the spike without your go-ahead. Okay?

Answers:
