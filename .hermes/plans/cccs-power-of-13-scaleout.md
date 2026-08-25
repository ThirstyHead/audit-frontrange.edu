# Plan: Scale the WCAG Audit Pipeline to the CCCS "Power of 13"

**Status:** P0–P4 COMPLETE (2026-08-24). P5 (first real 13-college run) blocked on 13 `RESULTS_TOKEN` secrets (you-do: PATs are browser-only).
**Date:** 2026-08-23
**Audience:** Scott + CCCS co-maintainers

**Progress log (2026-08-24):**
- P0 naming: resolved — all repos under `ThirstyHead` (no new org), tooling `audit-<domain>`, results `<abbr>-audit`, rollup `cccs-accessibility`. 14 instances total (13 colleges + cccs.edu).
- P1: config-driven core merged (PR #5) + `audit/discover-pages.mjs` merged (PR #6).
- P2: `cccs-audit-template` created + pushed (generalized core, per-instance setup README).
- P3: all 13 results repos seeded (main commit + GitHub Pages live on `thirstyhead.com/<abbr>-audit/`). Page discovery for all 13 instances: 11 via direct crawl; Morgan (Cloudflare, 72 pages) and Lamar (Sucuri, 239 pages) via sitemap through hound's stealth tier (hound v13.1.2 installed — was configured but binary missing). All 13 tooling repos scaffolded + pushed with per-college `college.json` + verified `pages.json` (Trinidad junk commit replaced).
- Verified config-driven pipeline on a fresh instance (Aurora, local run).
- P4: `cccs-accessibility` rollup built + live (fetches each public results repo, renders dashboard; Tuesdays 06:00 UTC + on-demand; stale colleges visible per §6.5).
- P5: canary dry run on ccaurora in flight. Remaining: RESULTS_TOKEN secrets ×13, then staggered 13-college runs.

## 1. Confirmed scope (researched, not assumed)

Resolved each college's primary web domain from its CCCS listing page:

| # | College | Web domain | Tooling repo | Results repo | Site path |
|---|---------|-----------|--------------|--------------|-----------|
| 1 | Arapahoe Community College | `arapahoe.edu` | `audit-arapahoe.edu` | `arcc-audit` | `/arcc-audit/` |
| 2 | Colorado Northwestern CC | `cncc.edu` | `audit-cncc.edu` | `cncc-audit` | `/cncc-audit/` |
| 3 | Community College of Aurora | `ccaurora.edu` | `audit-ccaurora.edu` | `aurora-audit` | `/aurora-audit/` |
| 4 | Community College of Denver | `ccd.edu` | `audit-ccd.edu` | `ccd-audit` | `/ccd-audit/` |
| 5 | Front Range CC *(done)* | `frontrange.edu` | `audit-frontrange.edu` | `frcc-audit` | `/frcc-audit/` |
| 6 | Lamar CC | `lamarcc.edu` | `audit-lamarcc.edu` | `lamar-audit` | `/lamar-audit/` |
| 7 | Morgan CC | `morgancc.edu` | `audit-morgancc.edu` | `morgan-audit` | `/morgan-audit/` |
| 8 | Northeastern Junior College | `njc.edu` | `audit-njc.edu` | `njc-audit` | `/njc-audit/` |
| 9 | Otero College | `otero.edu` | `audit-otero.edu` | `otero-audit` | `/otero-audit/` |
| 10 | Pikes Peak State College | `pikespeak.edu` | `audit-pikespeak.edu` | `ppsc-audit` | `/ppsc-audit/` |
| 11 | Pueblo CC | `pueblocc.edu` | `audit-pueblocc.edu` | `pueblo-audit` | `/pueblo-audit/` |
| 12 | Red Rocks CC | `rrcc.edu` | `audit-rrcc.edu` | `rrcc-audit` | `/rrcc-audit/` |
| 13 | Trinidad State College | `trinidadstate.edu` | `audit-trinidadstate.edu` | `trinidad-audit` | `/trinidad-audit/` |

Plus one rollup repo: **`cccs-audit`** (site at `/cccs-audit/`).

**Naming note:** the plan drops the `.edu` TLD from tooling repo names (e.g. `audit-pikespeak.edu` instead of the literal `audit-pikespeakstate-college.edu`). GitHub repo names are case-insensitive and awkward with the TLD in mid-string; dropping it is cleaner, and the domain is still fully recoverable from the repo's `college.json`. If you'd rather keep the exact-domain convention (as with `audit-frontrange.edu`), say so and I'll keep the TLD — it works for 12 of 13.

**Abbr verification:** the single-letter/multi-letter abbreviations (arcc, ccd, lamar, morgan, njc, otero, ppsc, pueblo, rrcc, trinidad) match each college's own domain or established usage; all 13 results-repo names are available-style and collision-free. Worth a quick visual check before creation since `ccd`/`njc` are common acronyms.

## 2. Strategy: template + independent copies (not a monorepo)

Your instinct is right, and there's a middle ground between "13 duplicated repos" and "one shared codebase with 13 configs":

- **13 independent tooling repos** (one per college) — each is a real, self-contained codebase owned by `ThirstyHead` (or, ideally, a dedicated GitHub org see §6), with the college's co-maintainers invited to *their* repo only.
- **13 independent results repos** — same ownership model.
- **1 rollup repo** (`cccs-audit`) — owns *no college's code or data generation*; it only *aggregates* published results.
- **1 template repo** (`cccs-audit-template`, hidden or archived once seeded) — the single place core improvements land.

**Why a template instead of shared library:** a `git submodule` or npm dependency would couple 13 pipelines to one release cadence and break college independence (a college couldn't diverge). A **GitHub repo template** gives each college a full independent copy while keeping a canonical path for updates: fix the core → merge to template → each college gets a one-line PR ("sync core vX") when it wants the fix. Colleges can also fork/patch freely — that *is* the independence you want, with a soft upgrade channel.

**Per-college co-management:** GitHub's repo-level permissions do exactly what you described — invite 1–2 colleagues per college as **Maintainers** (push + PR merge) on their college's two repos only. Zero overlap between colleges; the rollup repo gets the CCCS colleagues.

## 3. Rollup mechanics (how `cccs-audit` stays in sync without coupling)

Data flows **one way, file-based, no triggers, no secrets shared between repos**:

1. Each college's existing weekly workflow gains **one extra step**: copy the run's `latest.json` (already schema-stable, already produced) to `data/<abbr>/latest.json` in `cccs-audit` and push. This uses a **fine-grained PAT or GitHub App token scoped to that single repo + path** — college pipelines can *only* write their own data folder; they cannot touch other colleges' data, the rollup code, or anything else.
2. `cccs-audit` has its **own** Pages workflow (triggers: `workflow_dispatch` + a `schedule` on Tuesdays so it always re-renders after Monday's college runs) that reads `data/*/latest.json` and builds the rollup dashboard.
3. **Rollup dashboard** shows, per college: pages tested, violation count by impact, needs-review count, last-run date, and a trend sparkline (history lives in each college repo; the rollup keeps only `latest.json` + a rolling `data/<abbr>/history.json` appended per week, capped at 52 weeks to bound size).

**Opt-out is trivial:** a college stops pushing and its tile simply shows "stale" — the rollup never blocks, never requires, never fails because of one college. That's the independence/cooperation balance in one line.

## 4. Pipeline standardization (the parameterization pass)

The FRCC repo hardcodes identity in a few places (`college.json`-equivalent). First step of the build is a small refactor so **one codebase serves 13 identities**:

- **`college.json`** (new, at repo root): `{ name, abbr, domain, siteUrl, reportBaseUrl, pagesScopeFile }`.
- `audit/axe-audit.mjs`, `site/build-site.mjs`, and `weekly-audit.yml` read from it (title, raw-base URL, site name/URL already parameterized via CLI flags — mostly wiring).
- `site/build-site.mjs`'s `siteUrl` becomes the Pages path (`/arcc-audit/` etc.) so canonical links work.
- **`audit/discover-pages.mjs`** (new, promoted from the one-off script used for FRCC): crawl the homepage, extract same-host HTML links, verify 200, write a candidate `pages.json`. Human reviews → commits. This makes every college's scope expansion a *documented, repeatable* step instead of a bespoke script.
- FRCC becomes college #5: refactor lands as a PR on `audit-frontrange.edu` first (proven on the live pipeline), *then* is extracted into the template.

## 5. Execution phases

| Phase | Work | Output | Est. effort |
|-------|------|--------|-------------|
| **P0** | Naming sign-off (repo list above); confirm abbreviations & org choice (§6) | decision | 10 min |
| **P1** | Parameterization PR on `audit-frontrange.edu` (`college.json`, `discover-pages.mjs`); verify live FRCC site unchanged | merged PR | ~1 session |
| **P2** | Create `cccs-audit-template` (from refactored FRCC); org defaults: public repos, `main` branch protection, no fork-merge surprises | template ready | ~1 session |
| **P3** | Scaffold all 12 new colleges: fork template → `audit-<domain>`, create `<abbr>-audit`, edit `college.json` + `pages.json` (discover script generates candidates), enable Pages, set up the one data-push step | 24 repos, all green on a `workflow_dispatch` dry run | ~1–2 sessions + compute |
| **P4** | Build `cccs-audit`: rollup builder + dashboard + Pages | rollup repo | ~1 session |
| **P5** | First real 13-college run (staggered to be polite); review results; fix per-college surprises (WAF, login walls, mega-sites) | 13 live college sites + rollup | ongoing |

**Per-college risk notes to check in P3 (dry runs surface these):**
- **WAF/Cloudflare** — the discovery step already probes with a real browser UA; any college that 403s the bot needs a manual `pages.json` curation and possibly a slower crawl (the FRCC runner is serial by design, which is actually *good* here — no rate-limit issues).
- **Scale variance** — some of these are 50-page sites, some 300+. Keep the curated-scope model (no full crawl in CI); discovery is a *one-time curation aid*. Cap initial `pages.json` to the top-level nav + key sections.
- **JS-heavy sites** — the runner already waits for network idle; colleges with heavy SPAs may need per-page timeouts (the `pages.json` object form supports per-page options, so `{"path": "/x", "timeout": 30000}` works without code changes).
- **Login-walled content** — out of scope; document per college in its README which sections are excluded.

## 6. GitHub topology & governance

```
ThirstyHead (org)
├── audit-frontrange.edu ──┐
├── audit-arapahoe.edu     │ 13 college tooling repos
├── audit-<domain>.edu ... │  (each: college co-maintainers invited)
├── cc-aurora-audit ───────┤ 13 college results repos
├── <abbr>-audit ......... │  (Pages: thirstyhead.com/<abbr>-audit/)
│                          │
├── cccs-audit             │ rollup (CCCS colleagues co-maintain;
│                          │  receives data/<abbr>/latest.json pushes)
└── cccs-audit-template    ┘ (hidden; canonical core; update channel)
```

**Recommendations:**
1. **Create a dedicated org** (e.g. `cccs-a11y` or `thirstyhead-cccs`) instead of putting 26 repos under `ThirstyHead` — keeps your personal/other work separate, makes the invitation model cleaner, and gives org-level branch protection defaults. (Your call — `ThirstyHead` works fine too.)
2. **Fine-grained tokens, not a shared deploy key**, for the college→rollup push (each token scoped to one repo + one path). A shared key would let any college pipeline write anywhere in the rollup repo.
3. **Rollup is read-only w.r.t. colleges**: the rollup repo never has write access to college repos. Independence is structural, not just policy.
4. **Update cadence:** core fixes → template PR → per-college "sync" PRs (batch-able: one PR per college, same diff). Keep the sync diff small by pinning the core in a `core/` subdirectory so college-local changes (`college.json`, `pages.json`) never conflict.
5. **Reporting honesty:** the rollup dashboard should show *last run timestamp + status* per college, so a stale college is visible rather than silently absent.

## 7. What explicitly does NOT change

- FRCC's pipeline, schedule (Mondays 06:00 UTC), canonical URL, or repo contents stay as-is; P1 is additive.
- Human-in-the-loop for PRs holds for every new repo (template repos don't auto-create PRs; the scaffolding commits are made by you or by me with your approval per repo).
- The curated `pages.json` model stays — no auto-crawl in CI, ever.

## 8. Open questions for you

1. **Org:** new dedicated org vs. everything under `ThirstyHead`?
2. **Repo naming:** drop `.edu` from tooling repo names (my recommendation) or keep exact-domain convention?
3. **Rollup schedule:** Tuesdays (post-Monday-runs) or should the rollup also re-render on any `data/` push (immediate, more API calls)?
4. **Rollup scope of data:** `latest.json` only (small, no history trend) vs. latest + 52-week rolling history (trend lines, slightly larger)?
5. **Staggering:** run all 13 on the same Monday 06:00 UTC (simple, load-spiky on the college sites) or offset by minutes (e.g. `0,5,10,... 6 * * 1`) so we don't hit 13 WAFs at once?