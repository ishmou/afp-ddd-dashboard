# AFP DDD Dashboard — Analytics Upgrade & Visual Refresh

**Date:** 2026-06-12
**Status:** Approved by user
**Target file:** `dashboard-visibilite-ddd.html` (single-file app, ~5 290 lines)
**Audience priority:** Management / client presentations (screen-first, print must work)

## Goals

1. Surface the rich but under-used data (`topMedias`, `topPays`, email funnel metrics) as two new presentation-grade sections.
2. Deep visual refresh of existing sections, fixing all audit findings.
3. Zero regression on auth, decryption, data parsing, i18n, role gating.

## Non-goals

- No trends/momentum analytics (not selected).
- No Boost ROI section (not selected).
- No new libraries, no build step, no second file. Chart.js 4.4.0 + SheetJS stay as-is.

## Section A — Couverture média (`#mediaSection`)

New `<section id="mediaSection">` placed after `#comparative`, new sidebar nav item.
All aggregations run over `STATE.clientsAffiches` (respects global filters).

**A1. Stat strip** — 4 cards reusing KPI card visual language:
- Médias uniques: count of distinct outlet names across all `topMedias[]` (case-insensitive trim).
- Pays touchés: count of distinct keys across all `topPays{}`.
- Reach cumulé: sum of `topMedias[].reach` (null-safe).
- Campagnes couvertes: count of campaigns having `topMedias.length > 0` or non-empty `topPays`, shown as `n / total`.

**A2. Top Médias panel** — ranked list, top 12 outlets aggregated by name:
- Per row: rank, outlet name, country code badge (from `topMedias[].pays`), total reach (DM Mono), campaign count, proportional bar (share of #1).
- Custom HTML rows styled like the Executive ranking table (not Chart.js — rows need badges and two-part labels).

**A3. Top Pays panel** — `topPays` values summed per country across campaigns:
- Top 12 countries, proportional horizontal bars with share %, values in DM Mono.
- Bar colors cycle through the AFP region palette.

**A4. Empty state** — illustrated (SVG icon + i18n message) when no filtered campaign has media/geo data.

## Section B — Entonnoir d'audience (`#funnelSection`)

New `<section id="funnelSection">` after `#mediaSection`, new sidebar nav item.

**B1. Funnel** — custom inline SVG (no library), five stages computed over filtered campaigns *that have `emailsCibles`*:
1. Emails ciblés = Σ emailsCibles
2. Ouvertures = Σ (emailsCibles × tauxOuverture / 100), per-campaign, null-safe
3. Clics = Σ (emailsCibles × tauxClic / 100)
4. Vues uniques = Σ vuesUniques (labeled as multi-channel — journey, not strict funnel)
5. Pick-ups totaux = Σ totalPickups

Gradient trapezoids (primary palette), stage value in DM Mono, stage label via `t()`, conversion % between adjacent stages. Caveat note under the chart ("Les vues proviennent de l'ensemble des canaux" / EN equivalent).

**B2. Side rail** — key-rate cards:
- Taux d'ouverture moyen, Taux de clic moyen (weighted by emailsCibles, not naive mean).
- Pick-ups / 1 000 emails.
- "Meilleure conversion" callout: campaign with highest pickups-per-email among campaigns with ≥1 000 emails; clicking opens its detail panel.

**B3. Data honesty** — "n campagnes incluses" line showing how many filtered campaigns had email data. Empty state when zero.

## Section C — Visual refresh of existing sections

- **KPI cards:** accent gradient top border per card category; refined label/value/trend hierarchy; Revenue card gets a proper styled empty state instead of weak "n/d".
- **Chart.js pass:** gradient area fill on monthly chart (createLinearGradient), glass-style tooltips (bg `rgba(15,25,41,.95)`, border `--glass-border`, padding, Inter), rounded bars, softer gridlines; shared helper for tooltip config.
- **Comparative chart:** wider label column, full client + campaign name in tooltip (fixes truncation).
- **Heatmap:** color intensity normalized per column (per-KPI min/max) instead of global, so one dominant region no longer flattens others; text color switches for contrast on intense cells.
- **Radar (detail panel):** responsive container instead of fixed 220×220.
- **Empty states:** shared `.empty-state` upgrade — SVG icon + title + hint, applied to table, media, funnel sections.
- **Print:** new sections included in print styles; page-break-before on each major section; funnel SVG and ranked bars print correctly (background-color fallbacks via `print-color-adjust`).

## Constraints & wiring

- DO NOT modify: `checkSession`, `decryptData`, `normalizeClient`, `parseExcelRow`, `loadFromDataJson`, anything touching `sessionStorage.ddd_pwd`.
- Every new visible string: `data-i18n` attribute or `t("key")`, key added to **both** `LANG.fr` and `LANG.en`.
- New render functions `renderMediaIntel()` and `renderFunnel()` called from `renderAll()` only.
- Wiring per established pattern: sidebar nav item (`data-section`), section markup, `sectionIds` array, `sectionTitles` object.
- New sections visible to both ADMIN and INTERNAL (no financial data involved).
- All aggregations null-safe: fields may be `null`, arrays may be missing.

## Verification

1. Serve the repo over local HTTP (`python -m http.server`), open with headless browser.
2. Seed `sessionStorage.ddd_auth_ok` + role and `localStorage.ddd_dashboard_data` with mock campaigns (covering: full data, null emails, empty topMedias, single-region) before init.
3. Screenshot each section in FR and EN; verify funnel math against hand-computed mock totals; verify filters re-aggregate both new sections; verify print stylesheet via media emulation.
4. Verify login screen still gates access and existing sections render unchanged where not intentionally restyled.
