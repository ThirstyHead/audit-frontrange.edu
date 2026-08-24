# audit-frontrange.edu

Audit https://frontrange.edu (Front Range Community College) for WCAG 2.1 AA compliance.

## Architecture

This repo holds the **audit tooling only**. Its git history contains features and bug fixes — not generated data.

- `college.json` — **instance config** (see below). This repo audits the site named here; every other script reads its defaults from this file.
- `audit/config.mjs` — loads `college.json` (with FRCC fallbacks) for the runner, site builder, and workflow.
- `audit/axe-audit.mjs` — Playwright + axe-core audit runner; also renders a standalone per-page HTML audit report via [axe-html-reporter](https://www.npmjs.com/package/axe-html-reporter)
- `audit/pages.json` — the list of audited pages (`{ baseUrl, pages[] }`). Add a page as a one-line entry here.
- `audit/reports/` — local run artifacts: one `axe-<timestamp>.json` per run (not committed; see `.gitignore`) + `pages/<slug>.html` per-page reports from the latest run
- `site/build-site.mjs` — renders the public GitHub Pages site from report JSON; the home page lists every audited page with a link to its individual report
- `.github/workflows/weekly-audit.yml` — weekly scheduled run

**Publishing pipeline** (runs every Monday 06:00 UTC, or on demand via the Actions tab):

1. The workflow installs deps + Chromium, runs the audit (`npm test`).
2. It pushes the new report JSON (plus the run's per-page HTML reports) to **[`ThirstyHead/frcc-audit`](https://github.com/ThirstyHead/frcc-audit)** (the results repo), using the `RESULTS_TOKEN` repository secret.
3. It renders the site into that repo's `docs/` folder — served by GitHub Pages at **<https://thirstyhead.com/frcc-audit/>** (the account's `github.io` URL 301-redirects there).

> **Note on `RESULTS_TOKEN`:** the default `GITHUB_TOKEN` is scoped to this repo only and cannot push to a sibling results repo (it 403s — this was observed in the 2026-08-24 scheduled run). The publish step therefore requires a `RESULTS_TOKEN` repository secret: any token that can write to `frcc-audit` (e.g. a fine-grained PAT with Contents: read/write).

Enabling Pages in the results repo (one-time): in `frcc-audit` → Settings → Pages → Deploy from a branch → `main` / `docs`.

## Config-driven instances (CCCS "Power of 13")

This tooling is shared across the 14 CCCS audit instances (13 colleges + the CCCS system site). Each instance is an independent repo pair that differs **only by `college.json` (+ `audit/pages.json`)** — the code is identical. Fields in `college.json`:

| Field | Meaning |
|---|---|
| `name` / `slug` | Display name / short id |
| `domain` / `baseUrl` | Audited domain and base URL for `pages.json` paths |
| `toolingRepo` / `resultsRepo` | `ThirstyHead/<…>` repo names (results repo = the Pages site) |
| `siteName` / `canonicalSiteUrl` | Report-site branding and published URL |

Core changes (like this one) land here first, then propagate to each college repo as a "sync core" update when a college wants it. A weekly **rollup** of all instances is published from the `cccs-accessibility` repo at <https://thirstyhead.com/cccs-accessibility/>.

## Usage

```bash
npm test                        # audit all pages in audit/pages.json
node audit/axe-audit.mjs <url>  # audit specific URL(s) instead
npm run site:build -- --latest --history-dir <reports-dir>   # build site locally
```
