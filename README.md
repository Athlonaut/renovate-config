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
| Renovate merges its own pull requests (`platformAutomerge: false`) | GitHub's auto-merge cannot be enabled on a pull request that is already mergeable, and `dev` carries no required status checks — so platform auto-merge silently never completed and green pull requests sat open for a week. Renovate merging through the API works regardless of branch protection. |
| No `transitiveRemediation` | See below — it took the whole fleet offline. |
| Majors need dashboard approval | The Next/Clerk/Prisma stack breaks on majors; each deserves reading. |
| `eslint` majors blocked outright | `eslint-config-next` declares `eslint >=9` as its peer, but the plugins it pulls in (`eslint-plugin-react`, `eslint-plugin-import`, `eslint-plugin-jsx-a11y`) cap at `^9` and their newest releases still do. Nothing sets `strict-peer-dependencies`, so pnpm installs v10 with a warning and lint breaks afterwards. Blocked rather than left pending so the dashboard's approve-all button can't sweep it up. |
| `next` + `react` + types grouped | They move as a set; splitting them produces unbuildable intermediate states. |
| `@clerk/*` grouped | Clerk ships auth changes across several packages at once. |
| `prisma` + `@prisma/*` grouped | Client and CLI must match or `generate` breaks. |
| `minimumReleaseAge: 1 day` | Catches bad publishes before they reach you. Was 3 days, which quarantined security patches too: every exclusion it accumulated was a fix we wanted, and it is what silently held `nanoid` at 3.3.17. One day is pnpm's own default. |
| Lock file maintenance exempt from release age | A lockfile refresh touches hundreds of packages, so one of them is always younger than a day and `renovate/stability-days` never goes green — the monthly pull request sat unmergeable instead of quarantining anything useful. CI is the gate that matters for a bulk refresh. |
| Security alerts bypass schedule and release-age | A fix you're waiting on shouldn't sit until Monday. |
| Weekly, Monday before 6am | Updates are waiting when the week starts, not landing mid-flow. |
| Dependency Dashboard | One issue listing everything pending, instead of triaging pull requests. |

## Don't re-add `transitiveRemediation`

It was added 2026-08-10 and every hosted run after it died with
`kernel-out-of-memory`. No repo got a Renovate run for the following week — no
dependency dashboard was updated, no branch was pushed, and six high-severity
advisories (including the `nanoid` one it was added to catch) sat untouched
until someone went looking. A dead Renovate is worse than the gap it closes.

The option makes Renovate resolve full dependency trees to find fixes that exist
only in the lockfile. On the hosted app's memory ceiling, the large lockfiles
here don't fit — and because this preset is shared, one setting took down every
repo at once rather than the one that overflowed.

The lockfile-only gap is real, and is covered instead by version-scoped
`overrides` in each app's `pnpm-workspace.yaml`. If you want the option back,
scope it to one repo's own `renovate.json` first and confirm that repo still
gets a run — do not put it back in this preset on the strength of the docs.

## Running it

The **Mend-hosted Renovate app** is installed on the Athlonaut org and runs it.
No token or scheduled workflow is needed — the app authenticates as itself.

A self-hosted workflow lived here previously as a fallback. It was removed when
the app was installed: running both would mean two Renovates opening duplicate
pull requests on the same branches. If you ever switch back to self-hosting,
recover it from git history and add a `RENOVATE_TOKEN` secret.

**This repo is public** so the app (and Renovate's preset resolution) can read
`github>Athlonaut/renovate-config` without extra access grants. It holds only
dependency policy — no secrets. Keep it that way.

## Migrating an app off Dependabot

1. Add `renovate.json` with the extends line above.
2. Let one Renovate cycle run and confirm the pull requests look right.
3. Only then delete `.github/dependabot.yml`.
4. **Also turn off Dependabot security updates** — they are a separate repo
   setting that `dependabot.yml` does not control:
   `gh api -X DELETE repos/Athlonaut/<repo>/automated-security-fixes`

Running both briefly is noisy but safe. Deleting Dependabot first leaves a gap
where nothing is watching for security updates.

Step 4 is easy to miss. Every app in the org had no `dependabot.yml` and was
still running Dependabot security updates months later, opening pull requests
alongside Renovate's — and failing, because they target the default branch that
the `main via dev only` ruleset rejects.

Leave Dependabot **alerts** enabled. They are the detection feed Renovate's
`vulnerabilityAlerts` rule reads, so disabling them makes Renovate blind to
security issues rather than tidier:

```sh
gh api -X PUT repos/Athlonaut/<repo>/vulnerability-alerts   # alerts: keep on
gh api -X DELETE repos/Athlonaut/<repo>/automated-security-fixes  # the PR bot: off
```
