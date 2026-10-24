# Plan 03: Ingestion and queue

> Turn Cloud Logging Pub/Sub pushes into parsed `ErrorEvent`s sitting in a queue, with no filtering, fingerprinting, or agent involved.

Parent: [00-project-plan.md](00-project-plan.md) (first slice: ingest, parse, enqueue). Depends on Plan 02. Triage (filters, fingerprints, dedupe) is deliberately out of scope; see Plan 08.

## PRs

1. **Model.** `ErrorEvent` (source, severity, message, stack frames, resource/repo hint, raw payload, timestamp) and a queue item wrapper with release state (`queued`, `released`, `done`).
2. **Pub/Sub push adapter.** Parse the push envelope + Cloud Logging `LogEntry` into `ErrorEvent`; fixture-based tests; endpoint acks quickly.
3. **Queue.** `Queue` trait + in-memory implementation: enqueue, list, take-one-for-release. The endpoint now enqueues what it parses.
4. **Inspect.** Authenticated read endpoint listing queued items, so we can see what is collecting at volume.

The `Source` trait is introduced only if it falls out naturally from PR 2; otherwise it waits for the second source (Plan 07).

## Questions

1. Which log shapes must we support first? Plain `textPayload`, `jsonPayload` with `message`/`stack_trace`, Cloud Error Reporting's `ReportedErrorEvent`? Please paste 2-3 real (scrubbed) examples to use as fixtures.
2. How does an event map to a target repo? Label on the log resource, a config map from service name to repo, or a field in the log? This shapes `ErrorEvent`.
3. Pub/Sub auth: validate the OIDC token on push requests here or in Plan 06? **Recommendation:** here, as middleware with a config toggle, so the endpoint is never open once deployed.
4. With no dedupe, the queue grows without bound at volume. For the in-memory version, is a hard max length with drop-oldest acceptable until the persistent store exists? **Recommendation:** yes.

Answers:
