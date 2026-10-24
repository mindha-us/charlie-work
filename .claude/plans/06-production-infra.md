# Plan 06: Production infra

> Persist state in Firestore and codify the log sink, Pub/Sub subscription and Cloud Run deploy.

Parent: [00-project-plan.md](00-project-plan.md) (build order step 9 plus the deferred CI deploy from step 2). Depends on Plan 05.

## PRs

1. **Firestore store.** Implement the `Store` trait; integration tests against the emulator.
2. **Infra as code.** Log sink (severity >= ERROR), Pub/Sub topic + push subscription with OIDC, service account, Secret Manager entries. Schema/config-heavy, so it fits the LoC budget.
3. **Cloud Run service.** Service definition, min instances 0, env from Secret Manager.
4. **CI publish + deploy.** On merge to `main`: build and push to Artifact Registry, deploy to Cloud Run; Workload Identity Federation, no long-lived keys.

## Questions

1. Firestore vs Postgres (existing Supabase?). Firestore is the default in the parent plan. Any reason to switch?
2. Terraform or `gcloud` scripts? **Recommendation:** Terraform, if you already use it elsewhere. Do you?
3. Which GCP project(s) and region? One project hosting charlie-work, separate from the projects whose logs it watches? Cross-project log sinks need explicit IAM.
4. Environments: only prod, or a staging deploy as well?
5. Who applies Terraform: you manually, or CI? **Recommendation:** manual apply at first.

Answers:
