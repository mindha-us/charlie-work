# Plan 05: Dispatch and cost controls

> Wire triage to agent sessions behind hard spend limits, then prove the whole path against a sandbox repo.

Parent: [00-project-plan.md](00-project-plan.md) (build order steps 7-8, reordered so caps precede the first live call). Depends on Plans 03 and 04.

## PRs

1. **Run caps.** Max sessions per day (global) and "never while a PR for this fingerprint is open". Over budget means skip and log, not queue.
2. **Async dispatch.** Ack the push immediately, then create the session and send the first message in a spawned task. Includes the first-message builder (error context, scrubbed).
3. **Outcome recording.** Watch the stream to completion; store result (PR URL, failure, interrupted) per fingerprint.
4. **Monthly dollar budget.** Cost estimate from usage events times a price table (config).
5. **Per-session interrupt.** Interrupt when a session exceeds token/cost/time limits (uses native limit if Plan 04's spike found one).
6. **End-to-end.** Run against a sandbox repo with a seeded bug; documented as a repeatable manual procedure. No production code if all goes well.

## Questions

1. Is there a sandbox repo, or should one be created (public/private, which org)? Needed for PR 6.
2. Starting limits: sessions/day, monthly $ cap, per-session $ cap, per-session wall time? **Recommendation:** 3 / $50 / $2 / 15 min until we see real numbers.
3. Scrubbing: which fields count as sensitive (emails, tokens, user ids)? Regex denylist in config, or allowlist of fields forwarded? **Recommendation:** allowlist of fields plus a regex pass.
4. Pending work on restart: Cloud Run can kill the instance mid-session. Acceptable to lose the watcher and mark the attempt "unknown" until Plan 06 persistence, or must it resume now? **Recommendation:** accept for now.
5. How do we detect "PR already open for this fingerprint"? GitHub search by a branch/label convention (`charlie-work/<fingerprint>`)? Confirm the convention.

Answers:
