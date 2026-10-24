# Plan 07: Sentry source

> Add a Sentry webhook adapter behind the existing `Source` trait, proving the ingestion abstraction.

Parent: [00-project-plan.md](00-project-plan.md) (build order step 10). Depends on Plan 03; deploys via Plan 06.

## PRs

1. **Sentry adapter.** Webhook signature verification, payload to `ErrorEvent` mapping, fixtures, route registration.

Prompt iteration on real incidents is ongoing work, not a planned PR.

## Questions

1. Is Sentry actually in use anywhere you care about, or should this plan be dropped until needed?
2. Sentry issue alerts webhook vs internal integration? Which does your org have?
3. Do Sentry events carry enough to find the target repo (project slug mapping)?

Answers:
