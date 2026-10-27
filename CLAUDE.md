# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `yarn dev` — run the Next.js dev server locally
- `yarn build` — static-export build (writes `./out` because `next.config.js` sets `output: 'export'`)
- `yarn lint` — ESLint via `next lint`
- `bash ./scripts/health-check.sh` — run the monitoring probe locally. The script early-exits the commit path unless `git remote get-url origin` matches `statsig-io/statuspage`, so running it in a Spendbase checkout only prints results (set that remote in a scratch clone if you need to exercise the commit branch).

There is no test suite in this repo.

## Architecture

This is a fork of the Fettle status page, re-skinned for Spendbase. The whole system is three decoupled pieces that only communicate through files committed to this repo:

1. **Monitored URL list** — `public/urls.cfg`, a `KEY=URL` file. Adding a service means adding a line here; the key becomes the filename stem for its log and the display name on the page.
2. **Health probe** — `scripts/health-check.sh`, invoked hourly by `.github/workflows/health-check.yml` (`cron: "0 * * * *"`). For each entry it curls the URL up to 3 times, treats `200/202/301/302/307` as success, and appends `YYYY-MM-DD HH:MM, success|failed, <seconds>` to `public/status/<KEY>_report.log`. Each log is tail-trimmed to 2000 lines before being committed back to `main` (pushed via SSH using the `DEPLOY_KEY` secret, which is why the workflow writes `~/.ssh/id_rsa`). These commits are the `[Automated] Update Health Check Logs` entries that dominate `git log`.
3. **Static Next.js site** — `pages/` + `src/`, built with `output: 'export'` and deployed by `.github/workflows/pages.yml` to GitHub Pages. At runtime (in the browser) it re-fetches `urls.cfg` and the `*_report.log` files from `raw.githubusercontent.com/Spendbase/status-page/main/public/status/...` and renders from them. **The deployed site is read-only against GitHub; the workflow is the only writer.**

### Data flow in the UI

`pages/index.tsx` → `src/services/index.tsx` orchestrates two hooks that both parse the raw logs client-side:

- `src/services/hooks/useServices.tsx` — fetches each `<KEY>_report.log`, groups rows by day, and emits a 90-day `LogDaySummary[]` per service (status = `operational` if every row that day is success, `outage` if every row is failed, else `partial_outage`). Days with no data are backfilled as `unknown` via `fillData`.
- `src/services/hooks/useSystemStatus.tsx` — reads the last row of each log to compute the top-of-page banner (`All System Operational` / `Partial Outage` / `Outage` / `Unknown`).
- `src/incidents/hooks/useIncidents.tsx` — calls the GitHub REST API for issues on `Spendbase/status-page` labeled `incident`, groups them by month, renders under the services list. Incidents are managed as GitHub issues, not files.

Status strings live in `src/utils/constants.tsx` (`Status.OPERATIONAL | PARTIAL_OUTAGE | OUTAGE | UNKNOWN`).

### Repository-specific URLs you'll see hardcoded

The README's "fork and replace `mehatab/fettle`" instructions have already been applied — the repo slug `Spendbase/status-page` is baked into:

- `scripts/health-check.sh` (push target: `git@github.com:Spendbase/status-page.git`)
- `src/services/hooks/useServices.tsx` and `useSystemStatus.tsx` (log fetch URLs)
- `src/incidents/hooks/useIncidents.tsx` (GitHub issues API URL)

If this repo is ever forked or moved, all four hardcoded references must be updated together.

### Styling / framework

Next.js 15 (Pages Router, not App Router) + React 18 + Tailwind. `tsconfig.json` has `strict: true`. `CNAME` sets the custom domain for GitHub Pages.
