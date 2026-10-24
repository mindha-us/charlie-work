# Plan 05: Release gate and budget

> Release queued errors to the agent runtime one at a time by hand, then automatically within budget, and prove the whole path against a sandbox repo.

Parent: [00-project-plan.md](00-project-plan.md) (first slice: release to agent if within budget). Depends on Plans 03 and 04. Replaces the old "Dispatch and cost controls" plan; the manual release step is new and comes first.

## PRs

1. **Manual release.** Authenticated "release one" endpoint plus a CLI command that takes the next queued item (or a specific id), builds the first message from its `ErrorEvent`, and starts a run on the runtime. Works with the fake runtime.
2. **Outcome recording.** Watch the run to completion; record result (PR URL, failure, abandoned) on the queue item.
3. **Run caps.** Max runs per day; release is refused over the cap. Over budget means leave queued and log.
4. **Automatic release.** Config flag: release next item whenever within budget and nothing else is running. Off by default.
5. **Dollar / usage budget.** Monthly cap using whatever usage data the runtime provides (cost estimate from a price table in config, or plan-usage equivalent).
6. **End-to-end.** Run against a sandbox repo with a seeded bug; documented as a repeatable manual procedure. No production code if all goes well.
7. **Per-run interrupt** (only if the runtime exposes usage live). Interrupt when a run exceeds its token/cost/time limit.

If the manual control surface proves awkward to use, a small UI becomes its own plan rather than growing PR 1.

## Questions

1. Sandbox repo: is there one, or should one be created (which org, public/private)? Needed for PR 6.
2. Starting limits: runs/day, monthly cap, per-run cap, per-run wall time? **Recommendation:** 3 / (decided after Plan 04, depends on billing model) / small / 15 min.
3. Scrubbing: which fields count as sensitive? Allowlist of fields forwarded plus a regex pass? **Recommendation:** yes.
4. Restart behavior: if the worker dies mid-run, mark the item "unknown" until persistence exists? **Recommendation:** accept for now.
5. How do we tell whether a PR already exists for an error we are about to release? A branch/label convention (`charlie-work/<id>`)? Needed even without fingerprints, to avoid re-releasing the same item.

Answers:
