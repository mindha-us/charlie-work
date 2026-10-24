# Plan 08: Triage and incidents

> Group queued errors into incidents (filter, fingerprint, count, merge), store them behind an interface, and release incidents rather than raw events.

Parent: [00-project-plan.md](00-project-plan.md) (goal 2). Depends on Plans 03 and 05. Deliberately deferred until the first slice (ingest, queue, release to agent) works; this is a sketch to be split into smaller plans when it is picked up.

## Rough PR sequence (to be refined)

1. **Incident model.** `Incident` (id, fingerprint, count, first/last seen, status, linked events or source ref, PR link).
2. **Fingerprinting.** type + normalized message template + top frames; table-driven tests.
3. **One incident store.** The simplest that works (in-memory, then the persistent store from Plan 06). First occurrence creates an incident, later ones increment it.
4. **Release by incident.** The release gate takes open incidents, not raw events; "never while a PR is open" lives here.
5. **Filters.** Config-driven noise rules, min severity, windowed min count.
6. **Incident management.** Combine/split incidents, close them when the PR merges.
7. **Second incident store** (e.g. PagerDuty) and an `IncidentStore` trait, only once the first store is solid. Several stores may be active at once.

## Questions

1. Which external incident store do you actually want second: PagerDuty, something else, or none yet?
2. When the source system already has incidents (PagerDuty), is it the source of truth, with our store a cache, or do we mirror both ways?
3. Should merging two incidents be manual only at first? **Recommendation:** yes.
4. Does fingerprint stability across deploys matter (function names rather than line numbers)? **Recommendation:** yes.

Answers:
