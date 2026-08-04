# Athlonaut Renovate preset

One dependency policy for every Athlonaut app. Change it here, every repo picks
it up — including apps that don't exist yet.

Each app's entire Renovate config is:

```json
{ "extends": ["github>Athlonaut/renovate-config"] }
```

Saved as `renovate.json` in the app's root.

## What the policy does

| Behaviour | Why |
|---|---|
| Pull requests target **`dev`** | The org ruleset rejects any pull request into `main` whose head isn't `dev`. Targeting `main` would make every update unmergeable — the exact breakage Dependabot hit. |
| Non-major grouped into `production` / `development` | One pull request a week for routine bumps instead of a dozen. |
| Non-major **automerged** | CI is the gate. Majors always wait for a human. |
| Majors need dashboard approval | The Next/Clerk/Prisma stack breaks on majors; each deserves reading. |
| `next` + `react` + types grouped | They move as a set; splitting them produces unbuildable intermediate states. |
| `@clerk/*` grouped | Clerk ships auth changes across several packages at once. |
| `prisma` + `@prisma/*` grouped | Client and CLI must match or `generate` breaks. |
| `minimumReleaseAge: 3 days` | Catches bad publishes before they reach you. |
| Security alerts bypass schedule and release-age | A fix you're waiting on shouldn't sit until Monday. |
| Weekly, Monday before 6am | Updates are waiting when the week starts, not landing mid-flow. |
| Dependency Dashboard | One issue listing everything pending, instead of triaging pull requests. |

## Running it

Renovate needs to actually run somewhere. Two options:

**Self-hosted (free, no third-party terms):** `.github/workflows/renovate.yml`
in this repo runs it on a schedule across the org. It needs a `RENOVATE_TOKEN`
secret — a fine-grained PAT with, on the target repos: `Contents: read & write`,
`Pull requests: read & write`, `Issues: read & write` (for the dashboard),
`Workflows: read & write` (only if it should update `.github/workflows`), and
organization `Metadata: read`.

**Mend-hosted app:** install from the GitHub Marketplace. Check its current terms
for private repositories before relying on it — that has not been verified here.

## Migrating an app off Dependabot

1. Add `renovate.json` with the extends line above.
2. Let one Renovate cycle run and confirm the pull requests look right.
3. Only then delete `.github/dependabot.yml`.

Running both briefly is noisy but safe. Deleting Dependabot first leaves a gap
where nothing is watching for security updates.
