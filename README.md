# dependency-hygiene-starter

A small open-source Python utility that starts life with proper dependency-maintenance
habits instead of bolting them on later. This document records the research we did
before writing any code, so that our choices are traceable.

## Why we did this research

Dependency rot is a slow, compounding cost. Two things we wanted to get right from day one:

1. Which automation to trust (Dependabot vs. Renovate) and how to configure it.
2. How real, established projects schedule updates, group them, and cut down noise.

## Example configurations we read

We pulled and read the actual `.github/dependabot.yml` files from three established
open-source projects. Notes below are about scheduling, grouping, ignore rules, and
anything else notable.

### 1. spring-projects/spring-boot

File: `.github/dependabot.yml` in `spring-projects/spring-boot`.

- Two update blocks: `github-actions` at `/` and `npm` at `/antora`.
- **Schedule:** weekly for both ecosystems.
- **Grouping:** none configured.
- **Ignore rules:** none — but it uses `exclude-paths` so release-train workflow files
  are never auto-bumped.
- **Labels:** `type: task` and `status: waiting-for-triage` on every update PR, so
  updates flow into their normal triage queue.

Takeaway: weekly cadence is enough even for a huge project; label updates so they enter
your existing triage process instead of a parallel one.

### 2. pandas-dev/pandas

File: `.github/dependabot.yml` in `pandas-dev/pandas`.

- One update block: `github-actions`, with `directories` covering both `/` and a local
  composite action (`/.github/actions/setup-pixi`).
- **Schedule:** weekly.
- **Grouping:** a `groups:` block named `codeql-action` with pattern `github/codeql-action*`.
  Their inline comment explains why: *"init/autobuild/analyze must stay on the same version
  or CodeQL errors out."* So they group coupled dependencies so they move in one PR.
- **Cooldown:** `default-days: 7` — wait a week after a release before proposing a bump,
  which lets upstream bad releases surface first.
- **Labels:** `CI`, `Dependencies`.

Takeaway: group dependencies that must move together, and use a cooldown to avoid
chasing freshly-published releases.

### 3. pytorch/pytorch

File: `.github/dependabot.yml` in `pytorch/pytorch`.

- One update block: `pip` against a directory of CI commit pins
  (`/.ci/docker/ci_commit_pins`).
- **Schedule:** daily, on `target-branch: main`.
- **Grouping:** none.
- **Allow rules:** only `transformers` is allowed — everything else is excluded.
- **Ignore rules:** ignores `version-update:semver-patch` updates for `*` (all deps), so
  only minor/major bumps generate PRs. Patch-level churn is treated as noise.
- **Commit message:** prefix `[Dependabot] Update` with `include: scope`.
- **Labels:** `dependencies`, `python`, `module: ci`, and more, routing PRs to owning teams.

Takeaway: `allow` + `ignore` let you scope automation to exactly the dependencies you
care about; ignoring patch-level updates for pinned CI deps keeps the queue manageable.

## Public discussion recommendation we found

We looked for a public GitHub discussion where maintainers trade advice on
dependency-update automation (Dependabot vs. Renovate, grouping strategies).

**Found:** [dependabot/dependabot-core discussion #12437 — "🚀 Beta Release: Multi-Ecosystem Updates"](https://github.com/dependabot/dependabot-core/discussions/12437)
(Announcements category, opened by a Dependabot team member, with follow-up questions from
community maintainers).

The concrete recommendation worth adopting came from the discussion's grouping example:
maintainers asked how to keep ecosystems in **separate** groups, while grouping the
**coupled** packages that must move in lock-step — the example given was grouping
`playwright` (npm) with `playwright-ruby-client` (Bundler) *because they should be updated
in sync*, while all other npm and Bundler updates stay in separate groups. That matches the
real-world pattern in pandas' config above (their CodeQL action pins), so it is confirmed
practice rather than a one-off opinion.

**What we're adopting from it:** group by *coupling*, not by convenience. Only dependencies
that must be upgraded together go in one group; everything else gets small, reviewable PRs.

We also checked `renovatebot/renovate` discussions for Renovate-vs-Dependabot comparisons;
the current threads we sampled were narrow troubleshooting questions (regex validation,
token caching, floating wildcards), not general automation-choice advice, so we did not
quote one of those.

## Adoption checklist

1. **Add `.github/dependabot.yml` before the first line of application code** — `version: 2`,
   with two update blocks: `pip` at `/` and `github-actions` at `/`, both on a `weekly`
   schedule (the cadence spring-boot and pandas both use).
2. **Enable Dependabot security updates and secret scanning in repo settings on day one**,
   so vulnerabilities are patched via PR even when we're not actively triaging.
3. **Group by coupling, not by volume:** add a `groups:` block only for dependencies that
   must move in lock-step (e.g. `github/codeql-action*` when we adopt CodeQL), keeping all
   other updates in separate small PRs.
4. **Set a release cooldown of 7 days** (`cooldown: default-days: 7`, as pandas does) so we
   don't chase the first hours of a fresh release.
5. **Use ignore rules deliberately:** ignore `version-update:semver-patch` for CI-only
   pinned dependencies where churn matters more than freshness (the pytorch pattern), and
   re-review that ignore list monthly.
6. **Label every dependency PR** (`dependencies`, plus a module label) so updates enter our
   existing triage workflow with a clear owner.
7. **Validate the config in CI:** run a `dependabot.yml` schema check on PRs that touch the
   file (with a `paths:` filter so it only runs when relevant), per the advice in
   [dependabot-core discussion #13427](https://github.com/dependabot/dependabot-core/discussions/13427).

## Automation starting point

This repository also carries `.github/workflows/smoke.yml`, a minimal manually-triggered
Actions workflow that echoes the repository name, so the CI foundation exists from the
first commit.
