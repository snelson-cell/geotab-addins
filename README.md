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
| [`violin.html`](violin.html) | Speeding Distribution | Per-vehicle speeding rate (events/1,000 mi) distributed by office, dots colored High/Moderate/Low |

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
- **Speeding-risk tiers (fleet-wide, fixed):** Low `<10`, Moderate `10–39`,
  High `≥40` events/1,000 mi (fleet median / 80th percentile). Colors:
  High `#c0392b`, Moderate `#f2994a`, Low `#27ae60`.
- **Tiers are calibrated to a specific rule** — currently `RulePostedSpeedingId`
  ("Speeding"). They are *not* transferable: the retired `RuleGpsSpeedingWindowId`
  fired ~4× as often, which is why the old tiers were `<80 / 80–249 / ≥250`.
  If the speeding rule changes in MyGeotab, re-derive the median/p80 from live
  data and update `LO`/`HI` in `violin.html` — otherwise every vehicle collapses
  into one tier.
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

## Where the data tooling lives (NOT here)

The Python API tooling that pulls/analyzes Geotab data, plus `NOTES.md`, lives
locally in `~/Documents/Archive/GeoTab/` and is **intentionally not in this
repo** — it contains credentials and must stay off GitHub. This repo is
frontend/deployed assets only.
