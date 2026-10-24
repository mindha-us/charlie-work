# Plan 03: Ingestion and triage (no agent yet)

> Turn Cloud Logging Pub/Sub pushes into filtered, fingerprinted, deduped `ErrorEvent`s, stopping short of calling any agent.

Parent: [00-project-plan.md](00-project-plan.md) (build order steps 4-5). Depends on Plan 02.

## PRs

1. **Model + trait.** `ErrorEvent` (source, severity, message, stack frames, resource/repo hint, raw payload ref, timestamp) and a `Source` trait.
2. **Pub/Sub push adapter.** Parse the push envelope + Cloud Logging `LogEntry` into `ErrorEvent`; fixture-based tests; endpoint acks quickly and logs the event.
3. **Store trait + in-memory impl.** Minimal surface: record/lookup attempt by fingerprint, with timestamps.
4. **Filters.** Config-driven rules (min severity, min count, message/resource ignore patterns). Rules are data, not code.
5. **Fingerprinting.** type + normalized message template (strip ids, numbers, uuids) + top N frames; table-driven tests.
6. **Dedupe / cooldown.** Per-fingerprint cooldown using the store.

## Questions

1. Which log shapes must we support first? Plain text `textPayload`, `jsonPayload` with `message`/`stack_trace`, Cloud Error Reporting's `@type` ReportedErrorEvent? Please paste 2-3 real (scrubbed) examples to use as fixtures.
2. Is "min count" evaluated over a time window (needs a counter in the store) or is it just "seen N times ever"? **Recommendation:** windowed count, but ship severity-only first and add count in a later PR.
3. How does an event map to a target repo? Options: label on the log resource, a config map from service name to repo, or a field in the log. This determines the shape of `ErrorEvent`.
4. Fingerprint stability: should deploys (changing line numbers) change the fingerprint? **Recommendation:** use function names, not line numbers.
5. Pub/Sub auth: validate the OIDC token on push requests in this plan or in Plan 06 when infra exists? **Recommendation:** here, as a middleware with a config toggle, so the endpoint is never open once deployed.

Answers:
