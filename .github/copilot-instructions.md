# ai-platform-tenants

Machine-written GitOps state: one team file per team per environment, at `environments/{env}/{slug}.json`. The Portal API writes team files through `submit.yml`; a merge applies the team. From the infra release that adds the research tier (ai-platform-infra#18), one file is platform-owned: `environments/{env}/research.json`. Contract: `docs/gitops-contract.md` in `ai-platform-infra`.

## Review first

- **One whole file, canonical form** (keys sorted, two-space indent, LF), under 60 KB, `revision` increased, file name equals `team.slug`, folder equals `environment`. No secret, email address or name: `updatedBy` is one principal (`entra:`, `github:` or `app:`).
- **No resource names** in a team file. Only `research.json` carries Azure deployment names.
- **OAuth only for a team.** `subscription-key` is allowed only in `research.json`, which has no `oauth` block. `research` is not a team name, and a writer App never writes that file: an engineer does, by break-glass pull request. Before the environment's pin includes the research tier, `research.json` fails validation: do not add it.
- **Capacity and quota.** Deployments name a catalogue model offered to that tier, a non-Global SKU and capacity inside the model's `capacityPolicy`; the sum across teams stays within `quota`. 32 deployments per Foundry account at most.
- **Removal.** A team leaves through `pending-removal` (`removeAfter` at least the environment's cooling-off away) and then `removed`; where the cooling-off is zero, as in Infradev, an active team may go straight to `removed`. Only a removed team's file may be deleted. A change that removes more than one team in one apply needs an admin's `allow_destroy`.
- **One environment per pull request**, only under `environments/`.

## Conventions

- Nobody edits a team file by hand. Use `submit.yml`, or break-glass: branch `break-glass/{incident}`, whole file in canonical form (`node ai-platform-infra/checks/bin/aip-checks.js canonical --file <file>`), advance `revision`, trailer `Break-glass: {incident}`, label `break-glass`, two approvals from other people.
- Retry a failed apply by re-running its failed jobs (an engineer may, for the latest portal or push run; a run that is no longer the head of `main` is refused as `superseded`) or by dispatching `reconcile-all.yml`. A fresh hand dispatch of `apply.yml`, a hand `submit.yml`, `pause.yml` and `allow_destroy`/`force_unlock` need the repository admin role, for the actor and whoever re-ran it; anyone else fails the admin check (AC18).
- Workflow callers reference `ai-platform-workflows` at `@v2`; a new major arrives as a Dependabot PR reviewed by an engineer.

## Pull requests

`submit.yml` commits to `main` as the App with no pull request, so the pull requests here are break-glass, engineer or `.github/` changes (a `.github/` change needs a code-owner review by `aice-platform-admins`). The steps below are for them.

1. Branch `<type>/<description>` (`break-glass/{incident}` for incidents); conventional commits.
2. Validate against the environment's pinned infra release before opening the PR.
3. State the team, the environment and what the apply will create, change or remove.
4. **Request Copilot review**: `gh pr edit <n> --add-reviewer "@copilot"`. When it lands, fix every comment or reply with the reason it does not apply, push, then resolve each thread (GraphQL `resolveReviewThread`); re-request after a substantial push and repeat until nothing is unresolved. Then request a human reviewer. Copilot is advisory and never replaces the required approvals.
5. Do not merge your own PR.
