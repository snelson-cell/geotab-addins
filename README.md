# Mira Fleet Charts — MyGeotab Add-Ins

Custom MyGeotab dashboard add-ins for the Mira Home fleet. Each add-in is a
single self-contained HTML file, hosted on GitHub Pages, that queries Geotab
**live in the signed-in user's session** — so data is automatically scoped to
that user's role (an office OM sees their office; an exec sees the whole fleet).

**Live base URL:** https://snelson-cell.github.io/geotab-addins/

## Add-ins

| File | Menu name | What it shows |
|---|---|---|
| [`fleetFunnel.html`](fleetFunnel.html) | Fleet by Type | Funnel of the fleet by vehicle model (VIN-decoded) |
| [`topSpeed.html`](topSpeed.html) | Top Speed by Driver | Peak speed per driver for a chosen day or range, against the written speeding policy, with speeding-event counts and a "speed demon" callout. Built for 1:1s and team meetings |
| [`violin.html`](violin.html) | Speeding Distribution | KPI summary + per-vehicle speeding rate (events/1,000 mi) distributed by office, over Safe/Watch/High-risk zones, with a per-vehicle table |

Install config: [`config/mira-fleet-charts.config.json`](config/mira-fleet-charts.config.json) ·
Step-by-step: [`INSTALL.md`](INSTALL.md)

## Repo layout

```
geotab-addins/
├── fleetFunnel.html      # add-in (root = stable Pages URL — don't move)
├── violin.html           # add-in (root = stable Pages URL — don't move)
├── index.html            # landing / general info page (scaffold — expand later)
├── assets/
│   └── mira-home-logo.png
├── config/
│   └── mira-fleet-charts.config.json   # paste into MyGeotab → Add-Ins
├── INSTALL.md
└── README.md
```

> **Add-in HTML files live at the repo root on purpose** — their Pages URLs
> are referenced in the MyGeotab config. Moving them changes the URL and breaks
> the installed add-in. Add new add-ins at the root too.

## Update workflow

Edit a file, then:

```bash
git add <file> && git commit -m "…" && git push
```

GitHub Pages rebuilds in ~30–60 s at the same URL. In MyGeotab, hard-refresh
(Cmd+Shift+R) the add-in — no config change needed.

## Design conventions

- **Chart library:** ApexCharts (CDN). Keep it consistent across add-ins.
- **Speeding-risk tiers (fleet-wide, fixed):** Safe `<10`, Watch `10–39`,
  High risk `≥40` events/1,000 mi (fleet median / 80th percentile, derived
  2026-07-30). Status colors: high `#d03b3b`, watch `#fab219`, safe `#0ca30c`.
- **Prefer written policy to derived statistics when it exists.** `topSpeed.html`
  bands come from Megan Warner's escalations in the `#*-accidents-incidents`
  Slack channels, not from percentiles: **80+** is a final warning (termination
  if already on one), **85.1** is where her automated alert fires, **91+** is
  immediate termination. These superseded an earlier "85+ is termination"
  wording, so re-confirm with her before trusting them long-term. Percentile
  bands (as in `violin.html`) are the fallback for when no policy exists.
- **Thresholds are policy; statistics are description.** The tiers are *fixed*
  so "High risk" means the same thing to every viewer regardless of data scope,
  and stays comparable month over month. The fleet-median line and per-office
  median markers are computed live and move with the data. Don't blur the two.
- **This database is young, so the tiers will go stale — but not silently.**
  `checkDrift()` recomputes the empirical median/p80 on every render and raises
  a visible banner with re-derived values once ≥85% of vehicles land in one
  tier. When it fires, update `LO`/`HI` in `violin.html` **and** this section.
- **Tiers are calibrated to a specific rule** — currently `RulePostedSpeedingId`
  ("Speeding"). They are *not* transferable: the retired `RuleGpsSpeedingWindowId`
  fired ~4× as often, which is why the old tiers were `<80 / 80–249 / ≥250`.
- **Tier is encoded by background zone, never by dot color.** Tier is a pure
  function of x-position, so coloring the dots by tier spends the only free
  visual channel restating the axis. It also can't be made accessible: red↔green
  measures ΔE 2.8 under protanopia (validated, target ≥8) — no hex choice fixes
  that, because red and green *are* the confusion axis. Large labelled regions
  don't depend on hue discrimination. Dot fill carries driver-assigned instead.
- **Axis maxima come from a round step, not from rounding the data max.**
  Dividing a rounded max by a tick count is what produced ticks like
  `0/23/45/68/90`. See `axisScale()`.
- **Nothing is hidden without saying so.** Offices under the size cutoff,
  vehicles under the mileage floor, and outliers pinned at the axis edge are all
  named in the footer. A hidden office reads as an office with no problem.
- **Never hardcode a rule id as the only lookup.** A rule deleted in MyGeotab
  takes its historical exception events with it, and the Get call then returns
  `[]` rather than an error — which renders a plausible-looking chart of all
  zeros. Resolve rules by id-list *and* name fallback, and fail loudly on an
  empty result (see `resolveRule` / the zero-event guard in `violin.html`).
- **Data unit:** the vehicle (solid); driver labels are the current Geotab
  assignment (approximate). The violin tooltip names the vehicle and its
  current driver — the office is already the row label, so repeating it there
  wasted the hover. Driver comes from `DeviceStatusInfo.driver` (a `{id}` object,
  or the string `UnknownDriverId` when nobody is assigned) joined to
  `Get User {isDriver:true}`. Roughly a third of vehicles have no assignment and
  show "no driver assigned".
- **Don't present the driver as attribution.** It is who Geotab has assigned to
  the vehicle *now*, not who was driving during the 30-day window the events
  come from — hence the "current driver:" prefix rather than a bare name.
- **Live fetch only** — never bake in a data snapshot, or role-scoping breaks.
- **Ship CSS from JavaScript, not from `<head>`.** MyGeotab injects the add-in
  HTML into the host page and the `<head>` goes with it, so a `<style>` block
  there is silently dropped — the chart still renders (ApexCharts injects its
  own CSS from JS) while everything around it appears unstyled. `injectStyles()`
  appends the stylesheet at `initialize()`. The test harness must not add a
  `<style>` of its own, or it masks this.
- **Drill through to today's trips, not the vehicle edit page.** The question a
  dot raises is "what did it actually do?", which the trips view answers with
  route, stops and events. Parameter names follow the documented `tripsHistory`
  URL contract, **plus `devices`**, which that guide omits:
  `dateRange:(interval:Today), entityType:Device, devices:!(<id>),
  selectedEntities:!(<id>)`. With `selectedEntities` alone the page loads empty —
  world-wide `mapBounds` and `routes:()`. The way to settle this is to select a
  vehicle in Trips History by hand and diff the URL MyGeotab writes against the
  one you build: everything else in it (`routes`, `mapBounds`, `expandedCardIds`)
  is state the page derives after loading and must not be supplied.
  `dateRange:(interval:Today)` is right as-is — MyGeotab expands it against the
  database's timezone, so don't compute a UTC offset yourself.
  See [Using MyGeotab URLs](https://developers.geotab.com/myGeotab/guides/myGeotabUrls/);
  the add-in guide lists valid page names but no parameters at all, and the URL
  guide is incomplete, so verify against a real URL.
- **Never put `overflow` on the chart's container.** `overflow-x:auto` makes
  `overflow-y` compute to `auto` too (CSS forbids `visible` on one axis beside a
  non-visible other), so the card silently becomes a scroll container. ApexCharts
  renders its tooltip *inside* that box, so a tooltip near an edge extends the
  scroll area, toggles a scrollbar, shifts the layout under the cursor and
  dismisses itself — the tooltip flashes and vanishes, and hovering feels jerky.
  The chart is sized to its container, so it fits without any overflow rule.
- **Don't rebuild the chart while the pointer is on it.** A re-render destroys
  the element the tooltip is anchored to. The resize handler defers until
  `mouseleave` (see `pointerInChart`).
- **The vehicle table flows with the page — no nested scrolling.** It sits below
  the chart and you never need both at once. A `max-height` scroll box showed
  ~8 rows of 100+ behind a second scrollbar. Leaving overflow visible also keeps
  the sticky header anchored to the page scroll.
- **Size the chart explicitly and re-render on container resize.** ApexCharts'
  default `width: '100%'` re-used the width it first measured, so the plot never
  grew with the pane; the width is now passed from the measured container. It
  also only reflows on *window* resize, which misses a MyGeotab sidebar collapse
  or a split-screen drag — a `ResizeObserver` plus a 400ms poll covers it.
  Two traps worth remembering: an unreferenced `ResizeObserver` can be garbage
  collected and stop firing, and `ResizeObserver` delivery rides the
  animation-frame loop, so it goes quiet in a hidden tab. Key the "needs
  re-render" check on the width last *rendered*, never the width last *seen* —
  optimistically marking a width handled before the render runs leaves the chart
  frozen if that render is superseded.

## Where the data tooling lives (NOT here)

The Python API tooling that pulls/analyzes Geotab data, plus `NOTES.md`, lives
locally in `~/Documents/Archive/GeoTab/` and is **intentionally not in this
repo** — it contains credentials and must stay off GitHub. This repo is
frontend/deployed assets only.
