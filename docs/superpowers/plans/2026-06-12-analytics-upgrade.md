# Analytics Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Media & Geo Intelligence section and an Audience Funnel section to the AFP DDD dashboard, plus a visual refresh of existing sections, per `docs/superpowers/specs/2026-06-12-analytics-upgrade-design.md`.

**Architecture:** Single-file app (`dashboard-visibilite-ddd.html`). New analytics are pure render functions over `STATE.clientsAffiches` (already filtered), called from `renderAll()`. No new libraries; funnel is inline SVG, media/geo rankings are HTML ranked-bar rows. All strings via `t()` with keys in `LANG.fr` and `LANG.en`.

**Tech Stack:** Vanilla JS, Chart.js 4.4.0 (existing), CSS custom properties (existing token system).

**Verification harness (no unit tests exist):** serve repo via `python -m http.server 8765`, seed `sessionStorage.ddd_auth_ok="1"`, `ddd_role="ADMIN"`, and `localStorage.ddd_dashboard_data` with mock campaigns, then screenshot each section headlessly. Funnel totals are hand-computed from the mock and asserted visually.

**Forbidden (do not touch):** `checkSession`, `decryptData`, `normalizeClient`, `parseExcelRow`, `loadFromDataJson`, `sessionStorage.ddd_pwd` usage.

---

### Task 1: i18n keys

**Files:** Modify `dashboard-visibilite-ddd.html` — `LANG.fr` and `LANG.en` dictionaries (~lines 2778–3137).

- [ ] Add keys to BOTH `fr` and `en` blocks (suffix `_` names exactly as used in later tasks):
  - Nav/titles: `nav_media`, `nav_funnel`, `media_title`, `media_subtitle`, `funnel_title`, `funnel_subtitle`
  - Media stat strip: `media_kpi_outlets`, `media_kpi_countries`, `media_kpi_reach`, `media_kpi_covered`
  - Panels: `media_top_outlets`, `media_top_countries`, `media_campaigns_count`, `media_reach_label`, `media_empty_title`, `media_empty_hint`
  - Funnel stages: `funnel_stage_emails`, `funnel_stage_opens`, `funnel_stage_clicks`, `funnel_stage_views`, `funnel_stage_pickups`
  - Funnel rail: `funnel_rate_open`, `funnel_rate_click`, `funnel_rate_pickup1k`, `funnel_best_conv`, `funnel_included`, `funnel_caveat`, `funnel_empty_title`, `funnel_empty_hint`
  - Shared: `empty_no_data_title`, `empty_no_data_hint`
- [ ] Commit: `git commit -m "feat(i18n): keys for media intelligence and audience funnel sections"`

### Task 2: CSS

**Files:** Modify `dashboard-visibilite-ddd.html` — end of `<style>` block (~line 2076), plus print block.

- [ ] Add styles: `.intel-grid` (2-col, 1-col <1024px), `.intel-panel` (card), `.rankbar-row` (rank, label+badge, bar track/fill, value), `.country-badge`, `.stat-strip` (4-col auto-fit), `.funnel-layout` (svg + side rail), `.rate-card`, `.best-conv-card`, `.empty-state-rich` (icon + title + hint), `.funnel-caveat`.
- [ ] Print: include `#mediaSection, #funnelSection` in print visibility; `print-color-adjust: exact` on bars/funnel; `page-break-before: always` on new sections.
- [ ] Commit: `git commit -m "feat(css): styles for media intel, funnel, rich empty states"`

### Task 3: Section markup + nav

**Files:** Modify `dashboard-visibilite-ddd.html` — sidebar nav (~2107–2154), main sections (insert after `#comparative`, ~line 2363).

- [ ] Nav items with `data-section="mediaSection"` / `data-section="funnelSection"` and `data-i18n="nav_media"` / `data-i18n="nav_funnel"`, matching existing nav-item markup (svg icon + span).
- [ ] `<section id="mediaSection">`: `.section-head` (h2 `data-i18n="media_title"`, sub `media_subtitle`), `.stat-strip` (4 cards with value ids `mediaKpiOutlets`, `mediaKpiCountries`, `mediaKpiReach`, `mediaKpiCovered`), `.intel-grid` (two `.intel-panel`s with bodies `#topMediasList`, `#topPaysList`).
- [ ] `<section id="funnelSection">`: head + `.funnel-layout` with `#funnelViz` (svg container), side rail `#funnelRates`, caveat `<p class="funnel-caveat" data-i18n="funnel_caveat">`, included-count `#funnelIncluded`.
- [ ] Commit: `git commit -m "feat(html): media intelligence and funnel section markup + nav"`

### Task 4: renderMediaIntel()

**Files:** Modify `dashboard-visibilite-ddd.html` — new function near `renderHeatmap` (~line 5185).

- [ ] Aggregation (null-safe):

```js
function aggregateMedia(list){
  const outlets = new Map(); // key: lowercased trimmed name
  const countries = new Map(); // key: country name from topPays
  let covered = 0;
  list.forEach(c => {
    const hasMedias = Array.isArray(c.topMedias) && c.topMedias.length;
    const hasPays = c.topPays && Object.keys(c.topPays).length;
    if (hasMedias || hasPays) covered++;
    if (hasMedias) c.topMedias.forEach(m => {
      if (!m || !m.nom) return;
      const k = m.nom.trim().toLowerCase();
      const e = outlets.get(k) || { nom: m.nom.trim(), pays: m.pays || "", reach: 0, count: 0 };
      e.reach += Number(m.reach) || 0; e.count++; outlets.set(k, e);
    });
    if (hasPays) Object.entries(c.topPays).forEach(([p, v]) => {
      countries.set(p, (countries.get(p) || 0) + (Number(v) || 0));
    });
  });
  return { outlets, countries, covered };
}
```

- [ ] `renderMediaIntel()`: run over `STATE.clientsAffiches`; fill stat strip via `animateCounter`; render top-12 outlets (sorted by reach desc, bar width = reach/max) and top-12 countries (value desc, share % of total); rich empty state if `covered === 0`. Region palette colors cycled for country bars.
- [ ] Commit: `git commit -m "feat: media & geo intelligence section (top outlets, top countries, stat strip)"`

### Task 5: renderFunnel()

**Files:** Modify `dashboard-visibilite-ddd.html` — new function after `renderMediaIntel`.

- [ ] Aggregation:

```js
function aggregateFunnel(list){
  const withEmails = list.filter(c => c.emailsCibles != null && c.emailsCibles > 0);
  let emails=0, opens=0, clicks=0, views=0, pickups=0;
  withEmails.forEach(c => {
    emails += c.emailsCibles;
    if (c.tauxOuverture != null) opens += c.emailsCibles * c.tauxOuverture / 100;
    if (c.tauxClic != null) clicks += c.emailsCibles * c.tauxClic / 100;
    if (c.vuesUniques != null) views += c.vuesUniques;
    if (c.totalPickups != null) pickups += c.totalPickups;
  });
  return { n: withEmails.length, stages: [emails, opens, clicks, views, pickups], withEmails };
}
```

- [ ] SVG funnel: five horizontal gradient bands, width proportional to `sqrt(value/max)` (sqrt so small stages stay visible), centered; stage label left, value (fmtNum) right in DM Mono; conversion % between adjacent bands (skip the clicks→views transition arrow label, views is multi-channel — show "·" instead of %).
- [ ] Side rail: weighted open rate `Σ(emails×taux)/Σemails`, weighted click rate, `pickups/emails*1000` per-1k card; best-conversion callout = max `totalPickups/emailsCibles` among campaigns with `emailsCibles ≥ 1000`, click → `openDetail(id)`.
- [ ] `#funnelIncluded` = `t("funnel_included")` with n / total. Empty state if `n === 0`.
- [ ] Commit: `git commit -m "feat: audience funnel section with key-rate rail"`

### Task 6: Wiring

**Files:** Modify `dashboard-visibilite-ddd.html` — `renderAll()` (~4158), `sectionIds` (~4319), `sectionTitles` (~4320).

- [ ] Add `renderMediaIntel(); renderFunnel();` to `renderAll()`.
- [ ] Add `"mediaSection","funnelSection"` to `sectionIds`; add `sectionTitles` entries using i18n keys consistent with existing pattern.
- [ ] Verify language toggle re-renders both (applyLang → renderAll covers it).
- [ ] Commit: `git commit -m "feat: wire media and funnel sections into render + nav"`

### Task 7: KPI refresh + revenue empty state

**Files:** Modify KPI card CSS (~lines 470–600 region) and `renderKPIs` (~3458–3514).

- [ ] CSS: accent gradient top border (`::before` 2px gradient bar) keyed by a `data-accent` attribute on `.kpi-card`; refined value/label hierarchy.
- [ ] `renderKPIs`: when revenue is null, render styled placeholder (muted em-dash + `t("kpi_no_data")` caption) instead of bare "n/d"; add `kpi_no_data` key FR/EN.
- [ ] Commit: `git commit -m "refactor: KPI card visual refresh + proper revenue empty state"`

### Task 8: Chart.js styling pass

**Files:** Modify `renderMonthlyChart` (~3831–3907), `renderCompChart` (~3909–3985).

- [ ] Shared helper `glassTooltip()` returning the tooltip config (bg `rgba(15,25,41,.96)`, borderColor glass, padding 10, Inter, cornerRadius 8); use in both charts and radar.
- [ ] Monthly: `createLinearGradient` vertical fill (primary 0.18 → 0); line width 2; point radius 0 / hover 4.
- [ ] Comparative: `borderRadius: 6`; tooltip title callback returns full `nom — titreCampagne` (fixes truncation); axis tick callback keeps short label.
- [ ] Commit: `git commit -m "refactor: chart styling pass (gradients, glass tooltips, full-name tooltips)"`

### Task 9: Heatmap normalization + responsive radar

**Files:** Modify `renderHeatmap` (~5185–5244), radar creation in `openDetail` (~3773–3815) + its CSS.

- [ ] Heatmap: compute min/max **per column** (per KPI); opacity = (v−min)/(max−min) clamped [0.06, 0.85]; cell text switches to `--text-primary` when opacity > 0.5.
- [ ] Radar: container CSS `width:100%; max-width:280px; aspect-ratio:1; margin:auto`, Chart.js `responsive:true, maintainAspectRatio:true`.
- [ ] Rich empty states: swap table/section empty divs to `.empty-state-rich` markup with `t("empty_no_data_title")` / hint.
- [ ] Commit: `git commit -m "fix: per-column heatmap scale, responsive radar, rich empty states"`

### Task 10: Verification

- [ ] `python -m http.server 8765` in repo dir (background).
- [ ] Headless browser: open `http://localhost:8765/dashboard-visibilite-ddd.html`, run init script seeding `sessionStorage.ddd_auth_ok`, `ddd_role`, and `localStorage.ddd_dashboard_data` with 5 mock campaigns: (a) full data incl. topMedias/topPays, (b) emails but no taux, (c) no emails, (d) empty media, (e) second region. Reload.
- [ ] Assert: media stat strip counts match hand-computed mock values; funnel stage values match hand-computed sums; FR and EN toggle; filter by region re-aggregates; screenshots of every section; existing sections still render.
- [ ] Clean up mock storage; verify login screen still appears on fresh profile.
- [ ] Final commit + push to `main` (GitHub Pages serves it).

### Out of scope / follow-up

- `dashboard-local.html` regeneration: user runs `SYNC_DASHBOARD.BAT` after pulling (script re-injects data into the updated HTML).
