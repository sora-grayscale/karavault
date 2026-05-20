# Karavault CI verification

Status: documents the fork's PR CI strategy for issue #13.

This note exists so future maintainers understand how PR CI is expected to
work in the `sora-grayscale/karavault` fork, and what is intentionally
inherited from upstream `karakeep-app/karakeep` without modification.

## Workflow inventory

Inherited workflows currently in `.github/workflows/`:

- `ci.yml` — primary PR CI. Runs lint, format, typecheck, package tests, and
  the OpenAPI spec drift check on `pull_request` and `push` to `main`.
- `claude.yml` — GitHub `@claude` integration. Out of scope for #13; tracked
  separately under #19 because it hard-codes a single triggering actor and
  does not work for this fork as shipped.
- `docker.yml`, `android.yml`, `cli.yml`, `extension.yml`, `ios.yml`,
  `mcp.yml`, `sdk.yml` — release-oriented workflows inherited from upstream.
  Out of scope for #13.

## Decision: keep `ci.yml` unchanged for the MVP

- The inherited `ci.yml` already covers the checks we want enforced on every
  PR (lint, format, typecheck, tests, open-api spec).
- Adding a Karavault-specific PR workflow would duplicate coverage. A
  dedicated workflow will be added only when a Karavault-only check exists
  (for example, a vault-mode plaintext-leakage gate introduced under #11).
- This decision is revisitable; later PRs may swap or extend the workflow set.

## How to verify PR CI on the fork

1. Push a branch to the fork: `git push -u origin <branch>`.
2. Open a PR against `main` of the fork only (never upstream):
   `gh pr create --repo sora-grayscale/karavault --base main --head <branch>`.
3. In the GitHub UI, confirm `CI / lint`, `CI / format`, `CI / typecheck`,
   `CI / tests`, and `CI / open-api-spec` appear as checks on the PR and
   reach a conclusion.
4. If the checks do not appear, the fork's GitHub Actions need a one-time
   manual activation. Visit
   `https://github.com/sora-grayscale/karavault/actions` and click
   "I understand my workflows, go ahead and enable them". The GitHub API
   does not expose this toggle.

## Initial pre-activation state

- `gh api repos/sora-grayscale/karavault/actions/permissions` reports
  Actions enabled.
- `gh api repos/sora-grayscale/karavault/actions/workflows` reports zero
  workflows, even though the files exist on `main`. This was the gap #13
  is intended to close.

## Verified after activation

After the one-time UI activation on 2026-05-19, the gap closed:

- `gh api repos/sora-grayscale/karavault/actions/workflows` now reports
  `total_count: 9` with all inherited workflows in `state: "active"`.
- An empty re-trigger commit on PR #20 triggered CI run
  https://github.com/sora-grayscale/karavault/actions/runs/26081008839
  (event `pull_request`, conclusion `success`).
- All five PR check contexts passed:

  | check                | duration |
  |----------------------|----------|
  | `CI / lint`          | 2m12s    |
  | `CI / format`        | 2m28s    |
  | `CI / open-api-spec` | 2m18s    |
  | `CI / typecheck`     | 4m05s    |
  | `CI / tests`         | 10m50s   |

  These are the CI jobs that #12 should connect as required status checks
  on `main`; GitHub may display them either as job names (`lint`, `format`,
  ...) or as workflow-qualified check names (`CI / lint`, `CI / format`,
  ...).

## Follow-up

- #12 — once CI is reliably green, connect the relevant CI jobs as required
  status checks on `main` branch protection.
- #19 — fix the `claude.yml` actor restriction so the GitHub `@claude`
  integration is usable for fork maintainers.
