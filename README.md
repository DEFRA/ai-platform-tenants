# ai-platform-tenants

Machine-written GitOps state for the AI Platform: one team file per team per environment, at `environments/{env}/{slug}.json`. Nobody edits it by hand.
The GitOps worker dispatches `submit.yml` with the whole document for the Portal API; the workflow validates it at
the environment's pinned `ai-platform-infra` release and commits it to `main` as the GitHub App: a signed commit with
`Operation-Id`, `Requested-By` and `Traceparent` trailers, and no pull request. A push to `main` applies through
`apply.yml` as the environment's deploy identity. Every run sends its status to the Portal API as events. No step
waits for a person (D09, revised 25 October 2026).

The contract is `docs/gitops-contract.md` in `ai-platform-infra`; the workflows are in
`ai-platform-workflows`.

| Workflow            | Trigger                                     | Does                                                                                    |
| ------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------- |
| `submit.yml`        | Dispatch by the worker, or an admin by hand | Validate, commit to `main` as the App; `write`, `remove` or `revert`                    |
| `validate.yml`      | Pull request, review                        | `validate-desired-state / validate`                                                     |
| `apply.yml`         | Push to `main`, or a dispatch               | Apply with the safeguards, a run report and events                                      |
| `reconcile-all.yml` | Nightly, or an engineer's dispatch          | Apply every environment from `main` (`all: true`, no break-glass input); writes no file |
| `pause.yml`         | Dispatch by an admin                        | Read `AIP_PAUSED` in `{env}-events` and publish `paused` or `resumed`; writes nothing   |

A hand `submit.yml` run, any hand dispatch of `apply.yml`, `pause.yml`, and `allow_destroy` or `force_unlock` on
`apply.yml` need the repository admin role (`aice-platform-admins`), for the person who started the run and for whoever
re-ran it; anyone else's dispatch fails the admin check before any change. Only the App's own dispatches are exempt, and
an engineer's `reconcile-all.yml` run. Leave `operation_id`
empty on a hand run: it makes its own and refuses one that is passed in. `.github/` is code-owned by
`@DEFRA/aice-platform-admins` (`CODEOWNERS`), so a change to a caller needs their review.

## Break-glass

When the portal is down and a change cannot wait (design B `#break-glass`):

1. Branch `break-glass/{incident}`; edit the whole file in canonical form
   (`node ai-platform-infra/checks/bin/aip-checks.js canonical --file <file>`); advance `revision`.
2. Commit with the trailer `Break-glass: {incident}`; open a pull request titled with the incident and
   labelled `break-glass`.
3. `validate` runs as normal and needs two approvals from people other than the author.
4. The merge applies like any other change.
5. Close the incident only when the file matches what the portal would have written.

A failed apply is retried in one of two ways. An engineer re-runs the failed jobs of the latest portal or push run
(a re-run of a run that is no longer the head of `main` is refused as `superseded`), or dispatches `reconcile-all.yml`.
A fresh hand dispatch of `apply.yml` is not a retry for an engineer: the admin check refuses anyone who is neither an
exempt App nor a repository admin (AC18). `allow_destroy` and `force_unlock` on `apply.yml` are break-glass inputs for
repository admins only (`aice-platform-admins`) and are reported in the run summary.

The callers in `.github/workflows/` reference `ai-platform-workflows` at `@v2`, so every release within a
major reaches them unchanged. A new major arrives as a Dependabot pull request. It changes only `.github/`, so
`validate` passes it once someone has approved its head, and the ruleset's code-owner review makes that approval
one of `aice-platform-admins` (AC23); there is no bypass.
