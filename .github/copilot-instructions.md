# ai-platform-tenants

Machine-written GitOps state: one team file per team per environment, at `environments/{env}/{slug}.json`. The Portal API writes team files through `submit.yml`; a merge applies the team. One file is platform-owned: `environments/{env}/research.json`. Contract: `docs/gitops-contract.md` in `ai-platform-infra`.

## Review first

- **One whole file, canonical form** (keys sorted, two-space indent, LF), under 60 KB, `revision` increased, file name equals `team.slug`, folder equals `environment`. No secret, email address or name: `updatedBy` is one principal (`entra:`, `github:` or `app:`).
- **No resource names** in a team file. Only `research.json` carries Azure deployment names.
- **OAuth only for a team.** `subscription-key` is allowed only in `research.json`, which has no `oauth` block. `research` is not a team name, and a writer App never writes that file: an engineer does, by break-glass pull request.
- **Capacity and quota.** Deployments name a catalogue model offered to that tier, a non-Global SKU and capacity inside the model's `capacityPolicy`; the sum across teams stays within `quota`. 32 deployments per Foundry account at most.
- **Removal.** A team leaves through `pending-removal` and then `removed`; only a removed team's file may be deleted. A change that removes more than one team in one apply needs an engineer's `allow_destroy`.
- **One environment per pull request**, only under `environments/`.

## Conventions

- Nobody edits a team file by hand. Use `submit.yml`, or break-glass: branch `break-glass/{incident}`, whole file in canonical form (`node ai-platform-infra/checks/bin/aip-checks.js canonical --file <file>`), advance `revision`, trailer `Break-glass: {incident}`, label `break-glass`, two approvals from other people.
- Retry a failed apply with a fresh dispatch of `apply.yml`, never a re-run.
- Workflow callers reference `ai-platform-workflows` at `@v1`; a new major arrives as a Dependabot PR reviewed by an engineer.

## Pull requests

1. Branch `<type>/<description>` (`break-glass/{incident}` for incidents); conventional commits.
2. Validate against the environment's pinned infra release before opening the PR.
3. State the team, the environment and what the apply will create, change or remove.
4. **Request Copilot review**: `gh pr edit <n> --add-reviewer "@copilot"`. Answer every comment, then request a human reviewer. Copilot is advisory and never replaces the required approvals.
5. Do not merge your own PR or use the engineers' bypass except as documented.
