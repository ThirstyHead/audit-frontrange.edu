# audit-frontrange.edu

Audit https://frontrange.edu (Front Range Community College) for WCAG 2.1 AA compliance.

## Architecture

This repo holds the **audit tooling only**. Its git history contains features and bug fixes — not generated data.

- `audit/axe-audit.mjs` — Playwright + axe-core audit runner; also renders a standalone per-page HTML audit report via [axe-html-reporter](https://www.npmjs.com/package/axe-html-reporter)
- `audit/pages.json` — the list of audited pages (`{ baseUrl, pages[] }`). Add a page as a one-line entry here.
- `audit/reports/` — local run artifacts: one `axe-<timestamp>.json` per run (not committed; see `.gitignore`) + `pages/<slug>.html` per-page reports from the latest run
- `site/build-site.mjs` — renders the public GitHub Pages site from report JSON; the home page lists every audited page with a link to its individual report
- `.github/workflows/weekly-audit.yml` — weekly scheduled run

**Publishing pipeline** (runs every Monday 06:00 UTC, or on demand via the Actions tab):

1. The workflow installs deps + Chromium, runs the audit (`npm test`).
2. It pushes the new report JSON (plus the run's per-page HTML reports) to **[`ThirstyHead/frcc-audit`](https://github.com/ThirstyHead/frcc-audit)** (the results repo).
3. It renders the site into that repo's `docs/` folder — served by GitHub Pages at **<https://thirstyhead.com/frcc-audit/>** (the account's `github.io` URL 301-redirects there).

Enabling Pages in the results repo (one-time): in `frcc-audit` → Settings → Pages → Deploy from a branch → `main` / `docs`.

## Usage

```bash
npm test                        # audit all pages in audit/pages.json
node audit/axe-audit.mjs <url>  # audit specific URL(s) instead
npm run site:build -- --latest --history-dir <reports-dir>   # build site locally
```
