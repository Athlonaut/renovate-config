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
| All routine non-major updates grouped into **one** `non-major` pull request | Production, dev, GitHub Actions, override floors, Node and pnpm bumps share one weekly pull request. Renovate drove 84% of PR CI minutes in October 2026 (657 runs), and each separate group cost its own CI run plus a re-run on every conflict rebase; one group is one run. `next`+`react`, Clerk and Prisma keep their own groups (below) because they break in ways worth isolating. Majors stay separate, one pull request each, behind dashboard approval. |
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
| Lock file maintenance on the **first Sunday**, same Saturday-evening-to-Sunday window, **automerged** | Cron `* 20-23 1-7 * 6` plus `* 0-6 1-7 * 0` — days 1–7 that are also a Sunday is the first Sunday, so the monthly refresh lands on a review day rather than up to six days before one. The Saturday slot carries the same day-of-month filter; it can occasionally pick a Saturday that is not the one before the first Sunday (when the 1st falls on a Sunday, the Saturday slot matches the 7th instead), which is harmless. Automerge is gated on the branch's CI and, like every other automerge here, only reaches `dev` (`baseBranchPatterns`), so promotion to production stays a manual `dev`→`main` merge. |
| Majors blocked in `pnpm-workspace.yaml` | Overrides pin a security floor for a transitive dependency, not a version to track. A major forces an incompatible API on whichever package pinned the old line. In-range updates still flow. |
| Non-major override floors, the `packageManager` pin and Node **automerged** | These are one-line, mechanical, and covered by the full CI run, and they sat open for weeks in every repo waiting for a human (seven open PRs per repo on 2026-10-14), eating the concurrent-PR limit that lock file maintenance needs. Matching `pnpm-workspace.yaml` also lets the `next+react` group automerge: its branch carries the `next@<x` override line, and a group only automerges when every update in it may. Majors stay held by the rules above. |
| Engine ranges are floors (`rangeStrategy: replace` for `engines`) | With the global `bump`, every Node and pnpm PR rewrote `node: 24.x` to `24.21.0` and `pnpm: 11.x` to `11.28.5`, turning a compatibility statement into an exact pin that drifts out of date the day it merges. `replace` leaves a satisfied range alone, so engines only move on a new major. Same rule as the GreenJimmy preset. |
| Security alerts bypass schedule and release-age | A fix you're waiting on shouldn't sit until the weekly window. |
| Weekly, **Saturday 8pm to Sunday 7am** (cron `* 20-23 * * 6` and `* 0-6 * * 0`) | Aligned with the automated weekly security review, which runs at 7am Sunday America/Phoenix. The window closes before the review starts, so everything Renovate opens is already on the Dependency Dashboards and in the PR lists when the review runs. It was Sunday 2–6am, five hours wide, which Renovate's docs call enough — but the Mend-hosted app visits each repo every few hours on its own clock, and a five-hour window is missed whenever one visit lands just before it opens and the next lands after it closes. That happened fleet-wide on 2026-10-11 (trysyncom's visit at 1:07am, nothing opened all week) and left every weekly review refreshing lock files by hand. Eleven hours spans at least two visits. Cron is used because Later text syntax is deprecated, and the minute field must be `*` — Renovate has no minute granularity. |
| Rebase only on conflict (`rebaseWhen: "conflicted"`) | The default rebases every open pull request each time `dev` moves, and every rebase re-runs CI. This was about a third of the org's Actions minutes. |
| `pnpm-workspace.yaml` overrides in the `non-major` group | One pull request per run instead of one per override line, each with its own CI run. The `next@<x` override line still lands in `next+react`, because that rule comes later and wins. |
| No `updateNotScheduled: false` | It would stop Renovate touching open pull requests outside the weekend window, but the `renovate/stability-days` status is only refreshed when Renovate processes the branch. A group opened on Saturday with a release under a day old would stay pending until the next weekend, and automerge would stall for a week. `rebaseWhen: "conflicted"` already removes the routine re-runs. |
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

## Changing the preset

Same flow as the apps: branch off `dev`, open a pull request into `dev`, then
promote `dev` into `main`. Consumers resolve the preset from `main`, so nothing
takes effect fleet-wide until that second merge.

Two rulesets enforce it, matching the product repos:

| Ruleset | Branch | What it does |
| ------- | ------ | ------------ |
| `dev stays` | `dev` | No deletion, no force-push. Without it, auto-delete-on-merge removes `dev` every time a promotion merges — which is exactly what happened on 2026-09-02 |
| `main via dev only` | `main` | Pull request required, no deletion or force-push, and two checks must pass |

The checks are `Source branch must be dev` (from `branch-policy.yml`) and
`Lint & Test` (from `ci.yml`, which runs `renovate-config-validator`). The apps
get the first from `Athlonaut/.github/actions/require-dev-source`; this repo
inlines the same four lines instead, because that action lives in a private repo
and a public repo cannot resolve it — it fails as "not found", not as a
permissions error. Renaming
either job locks `main` until the ruleset is updated to match. Validation earns
its place here because a malformed `default.json` breaks Renovate in every repo
at once, with no pull request anywhere to revert.

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
