# Mira Fleet Charts — MyGeotab Add-Ins

Custom MyGeotab dashboard add-ins for the Mira Home fleet. Each add-in is a
single self-contained HTML file, hosted on GitHub Pages, that queries Geotab
**live in the signed-in user's session** — so data is automatically scoped to
that user's role (an office OM sees their office; an exec sees the whole fleet).

**Live base URL:** https://snelson-cell.github.io/geotab-addins/

## Add-ins

| File | Menu name | What it shows |
|---|---|---|
| [`scorecard.html`](scorecard.html) | Driver Scorecard | Every driver ranked 0–100 against a frozen company benchmark — speeding 55%, braking 25%, cornering 10%, acceleration 10% — with office standings, a worked example of one driver's arithmetic, and 7/14/30-day windows |
| [`health.html`](health.html) | Market Standings | Per-location readiness: driver attribution, camera coverage, reporting rate, and a safety rank drawn from the *same* benchmark the scorecard uses. Gates each location as full / partial / not measurable |
| [`speeding.html`](speeding.html) | Speeding by Zone | Where speeding happens by posted limit — three policy bands (7 over under 30 mph, 10 over to 45, 12 over above) — with a how-far-over histogram and per-event links into Trips History |
| [`topSpeed.html`](topSpeed.html) | Top Speed by Driver | Peak speed per driver for a chosen day or range, against the written speeding policy, with speeding-event counts and a "speed demon" callout. Built for 1:1s and team meetings |
| [`fleet.html`](fleet.html) | Fleet Roster | Every vehicle in Geotab — ownership, location, driver, plate, camera and the compliance fields that used to live in the branch spreadsheets. KPI cards follow the active filters |
| [`rollout.html`](rollout.html) | Fleet & Camera Coverage | Per-location install progress: shipped → assigned → installed (VIN reporting) → camera fitted, so you can see which locations have done what |
| [`vehicles.html`](vehicles.html) | Maintenance | Per-vehicle worklist — oil life, tire pressure by corner, odometer freshness and triaged device faults — plus the roadside-assistance number and the Enterprise service-locator link |

Rows are in **menu order**, which is the order of the `items` array in the config.
It is not alphabetical and is not sorted — Geotab renders the array as given, so
the array *is* the running order.

### In the repo but not in the menu

Both still build and still work; they were retired from the menu, not deleted,
because their Pages URLs may still be bookmarked.

| File | Was | Retired | Why |
|---|---|---|---|
| [`violin.html`](violin.html) | Speeding Rate by Vehicle | 2026-08-27 | Superseded by **Speeding by Zone**, which answers the same question against the posted limit instead of a flat per-vehicle rate |
| [`fleetFunnel.html`](fleetFunnel.html) | Fleet by Location | 2026-08-27 | Its content was absorbed into **Market Standings**, which carries the per-location table alongside the safety rank |

Install config: [`config/mira-fleet-charts.config.json`](config/mira-fleet-charts.config.json) ·
Step-by-step: [`INSTALL.md`](INSTALL.md)

## Repo layout

```
geotab-addins/
├── scorecard.html        # add-in — Driver Scorecard     (root = stable Pages URL)
├── health.html           # add-in — Market Standings
├── speeding.html         # add-in — Speeding by Zone
├── topSpeed.html         # add-in — Top Speed by Driver
├── fleet.html            # add-in — Fleet Roster
├── rollout.html          # add-in — Fleet & Camera Coverage
├── vehicles.html         # add-in — Maintenance
├── violin.html           # retired from the menu, still served
├── fleetFunnel.html      # retired from the menu, still served
├── index.html            # landing page, served from Pages directly — NOT an
│                         # add-in, so relative asset paths are fine here
├── assets/
│   └── mira-home-logo.png
├── benchmark/
│   └── company-30d.json  # frozen company distribution behind the scorecard;
│                         # INLINED into scorecard.html and health.html, not fetched
├── config/
│   └── mira-fleet-charts.config.json   # paste into MyGeotab → Add-Ins
├── INSTALL.md
├── RULE-SETUP.md         # MyGeotab rule calibration checklist
└── README.md
```

`scorecard/` exists locally (driver-scorecard prototype) and is gitignored —
this repo is public and frontend-only.

> **Add-in HTML files live at the repo root on purpose** — their Pages URLs
> are referenced in the MyGeotab config. Moving them changes the URL and breaks
> the installed add-in. Add new add-ins at the root too.
>
> That is also why **several files keep filenames they outgrew**. `fleetFunnel.html`
> has not drawn a funnel since it was rebuilt as location × ownership;
> `violin.html` stopped being a violin plot; `health.html` is "Market Standings"
> and `vehicles.html` is "Maintenance". In every case the filename is baked into
> the installed config, so renaming it would break the add-in for every user —
> and for the two retired pages it would also break any bookmark. **The file name
> is not a description; the table above is.**

## Update workflow

Edit a file, then:

```bash
git add <file> && git commit -m "…" && git push
```

GitHub Pages rebuilds in ~30–60 s at the same URL. In MyGeotab, hard-refresh
(Cmd+Shift+R) the add-in — no config change needed.

### Verifying a deploy

**Compare the hash of the served file to the local one. Nothing else counts.**

```bash
curl -s "https://snelson-cell.github.io/geotab-addins/topSpeed.html" -o /tmp/live.html
shasum -a 256 topSpeed.html /tmp/live.html   # the two digests must match
```

That is immune to CDN caching, build lag, and quirks in the status APIs, and it
answers the only question worth asking: is what is being served the same as what
was written.

Two ways of checking that both produced **wrong answers, in opposite
directions**, on 2026-08-06:

- **Grepping the fetched file for a string you just added.** A cached response
  serves the old file with a 200, so the string is absent and the deploy looks
  broken; worse, a string that already existed in the previous build makes a
  stale response look current. This wrongly reported a good deploy as live
  before it had built.
- **`gh api repos/<o>/<r>/pages/builds` as a *status* check.** That endpoint
  reports **legacy builds**. A deploy driven by the `pages-build-deployment`
  Actions workflow did not show up there, so it reported a phantom gap and a
  perfectly healthy deploy looked like it had never happened. For status, use
  `gh api repos/<o>/<r>/deployments` or the repo's Deployments page.
  **Do not read this as "avoid that API"** — the same endpoint's *write* side is
  the outage escape hatch; see the next section. Read ≠ write here.

Do not tell anyone a change is live on the strength of a string match.

### When Actions is down: build Pages directly

The warning above is about the legacy **read** endpoint. The legacy **write**
endpoint is a different thing and is the escape hatch — this repo's Pages source
is `build_type=legacy` (deploy from a branch), so a build can be requested
without Actions at all:

```bash
gh api -X POST repos/snelson-cell/geotab-addins/pages/builds
gh api repos/snelson-cell/geotab-addins/pages/builds/latest -q '.status'
```

On 2026-08-06 a GitHub incident (Actions + Pages, "major outage", runners not
being acquired) failed three `pages-build-deployment` runs in a row and left a
fourth permanently wedged — GitHub's own state was self-contradictory for it,
with `cancel` reporting "already completed" and `rerun` reporting "already
running". Four commits sat undeployed for nine hours. The POST above built them
in **20 seconds**. Reach for it when a run is queued or failing for
infrastructure reasons; verify with the hash check as always.

## Design conventions

- **Chart library:** ApexCharts (CDN) *where a page actually plots data* —
  `topSpeed`, `rollout`, `violin`, `fleetFunnel`. The four newest pages
  (`scorecard`, `health`, `speeding`, `vehicles`) load **no charting library at
  all**: they are tables with inline bars and CSS-drawn distributions, and a CDN
  dependency plus a render cycle would buy nothing. Each says so in a comment at
  the top of the file. Don't "standardise" one of them onto Apex.
- **The scorecard's benchmark is frozen, inlined, and shared.** `benchmark/company-30d.json`
  is a company-wide distribution generated offline so that every login scores
  identically regardless of what that user can see. It is **inlined into both
  `scorecard.html` and `health.html`**, never fetched — these files are injected
  into MyGeotab's page, so a relative URL resolves against `my.geotab.com` and
  quietly fails. Two consequences: (1) **regenerate both copies together**, or
  Market Standings will rank offices against a stale snapshot while the scorecard
  uses a fresh one — the generated date is printed on screen so a drift is
  visible rather than silent; (2) the inlined copy is a **branch-level aggregate
  only, no individuals**, so a branch-scoped login can see the whole standings
  table without seeing anyone else's people.
- **Weights and the benchmark are coupled, and the page hard-stops if they drift.**
  Editing `METRICS` weights in `scorecard.html` without regenerating leaves every
  rank built from a different scoring function than the score beside it.
  `weightMismatch()` names each mismatched metric and refuses to render;
  `floorMismatch()` does the same if the benchmark predates the clean-sheet rule.
- **A clean sheet costs nothing.** Percentile scoring breaks when most of the
  fleet is tied at zero — 85 of 101 acceleration breakpoints were exactly `0.00`,
  so a zero landed mid-tie and cost points for a record that could not be
  improved. A driver at a metric's floor takes the **bottom** of the tie block,
  for both score and cell colour. A clean metric therefore reads "clean — nothing
  off", never "better than 100% of the company": they are *tied* with everyone at
  zero, not ahead of them.
- **Rank is noise-dominated; coach from the band, not the number.** Scoring two
  independent 30-day windows with identical weights moves the median driver **33
  places**. About 48% of drivers cannot be told apart from typical. Changing the
  weights is a *smaller* perturbation than one fortnight of luck. When someone
  asks why a driver moved 30 places, the answer is usually "they did not move,
  the sample did."
- **Geotab CAN reprocess history — check before clamping a window.** The 11 Aug
  2026 rule change created a genuine discontinuity, and the scorecard carried a
  fetch clamp for a fortnight because of it. On 27 Aug an overnight reprocess
  re-evaluated every event from 1 Jul against the current rule definitions and
  the discontinuity disappeared from live data. Verify with a **control week**
  that never changed: if it reproduces, history has been recalculated and the
  clamp is now doing harm. The scorecard's `rulesChangedAt` is `2026-07-01` with
  a full 30-day clean window and an empty `metricChangedAt`.
- **30 days is a choice, not a limit.** ~56 days are available. In early July the
  reporting fleet was 47% Chicago/Detroit/Indianapolis — the highest-speeding
  markets — against 21% now, with miles up ~5x as devices came online. A longer
  window imports a fleet composition that no longer exists.
- **Speeding policy branches on the posted limit, because the rule engine cannot.**
  `speeding.html` applies the cut client-side: **7 over** in a 20–30 zone,
  **10 over** in 31–45, **12 over** at 46+. Geotab's rule engine can't express
  "when posted is under 30" — `SpeedLimit` is only usable paired with `Speed`.
  The thresholds differ on purpose: 10 over is 40% over in a 25 but 15% over in a
  65, so a flat number gets steadily softer as speeds rise. The bands are
  **contiguous** — posted 31–34 used to fall in a gap and be silently discarded.
  Posted limits **below 20 mph are dropped** as parking lots, yards and map
  estimates; left in, a 5 mph "zone" yields a 4.6x ratio and tops the list forever.
- **Speed comes from the event's own distance ÷ duration, not from LogRecord.**
  Geotab admits ~300 calls/minute and every multiCall sub-call counts, so
  resolving each event individually costs ~1,800 calls for one office-week.
  `GetRoadMaxSpeeds` takes a date *range*, so one call per device covers the whole
  window. The sustained average is also the *better* number: unlike a peak, it
  cannot be manufactured by a single bad GPS fix.
- **Faults are triaged by ID, never counted.** Geotab returns ~3,939 "Active"
  faults across 297 vehicles, but `severity` is null on every one and the J1939
  lamp fields are never populated on light-duty OBD — so a raw count flags 80% of
  the fleet and means nothing. `vehicles.html` splits them into `DEV_ACTION`
  (unplugged, low voltage, poor GPS, accelerometer, CAN short) and `DEV_NOISE`
  (power cycles, modem/radio restarts, watchdog resets). "All power removed"
  alone fires on 208 of 387 vehicles, which is what turned the attention list into
  the whole fleet.
- **Tire pressure self-calibrates against the vehicle's own four corners.**
  FMVSS 138 wants a lamp at 25% below the *placard* pressure, and the placard is
  not in Geotab. Each vehicle is compared with the median of its own four corners
  instead, so Mavericks, Colorados and rentals all judge correctly with no lookup
  table. A 6 psi spread best-to-worst reads as a probable leak.
- **Odometer must be fetched in small backward steps.** It logs heavily: a single
  24h fleet-wide `Get` hits the row cap and then returns *fewer* devices than a
  12h one, because truncation eats them. `LOOKBACK_H` walks 12 / 36 / 84 / 180
  hours and keeps the newest per device.
- **Speeding-risk bands (fleet-wide, fixed) — `violin.html`, now retired from the
  menu:** Within target `<10`, Monitor
  `10–39`, Coaching review `≥40` events/1,000 mi (fleet median / 80th
  percentile, derived 2026-07-30). Colors: high `#d03b3b`, watch `#fab219`,
  safe `#0ca30c` (the internal keys stay `high`/`mod`/`low`).
  **Reviewed 2026-08-07 and deliberately left as-is.** They were derived from
  *truncated* data — see the 25,000-row cap below — so the statistics behind
  them were wrong: the true 30-day fleet median is `5.6` and p80 `27.8`, not the
  `8.1`/`35.8` they came from, which would imply roughly `LO=5, HI=30`. Both
  bands therefore sit more leniently than intended. Samson was shown these
  numbers and chose to keep 10/40: moving `HI` changes who gets flagged for
  coaching, which is a decision about people, not a correctness fix. The bands
  still discriminate (66% Within target / 21% Monitor / 14% Coaching review) and
  `checkDrift()` does not fire. **Don't re-propose re-deriving them** unless
  `checkDrift()` actually fires or the thresholds come up on their own.
- **Band names describe the measurement and the next step, never the driver.**
  Renamed from Safe / Watch / High risk on 2026-08-06. "High risk" characterises
  a *person* in a record that is discoverable; "Coaching review" states what
  happens next, which is a fact about our workflow. "Safe" was the same problem
  inverted — a vehicle at 9 events/1,000 mi is still speeding regularly, it is
  just under the fleet's own line, and we should never appear to certify that.
  Numbered levels (Level 1/2/3) were considered and rejected: numbers carry no
  inherent direction (a Level 1 trauma centre is the best one, a Category 1
  hurricane the weakest), so they force every viewer to learn a mapping.
  **This is not legal protection** — discovery gets the definitions too. What
  protects the company is a flag here being followed by *documented* coaching.
  All six surfaces read from the single `TIERS` object; before the rename there
  were four names for one band (zone "HIGH RISK", KPI "High risk", table "High",
  tooltip "High") against a banner that already said "coaching review".
- **A zone label is dropped when its band is too narrow to hold it.** The
  within-target band is only `LO/visMax` of the plot — a tenth of it on a 100
  axis — so on a narrow pane even the old three-letter "SAFE" overprinted
  "WATCH" and read as "SAFEWATCH". `zone()` measures and omits; the legend
  always carries the full names, so nothing is lost.
- **A rate needs exposure, not just a sample.** Miles are the *denominator*, so
  a vehicle with few miles swings wildly on one event: at 250 mi one extra event
  moves it 4 points, at 100 mi it moves it 10. `MIN_MILES` (50) is a hard floor
  for rating at all; `THIN_MILES` (250) is a softer one that names the affected
  vehicles in the footer. This matters most on the 7-day window — 72 of 189
  vehicles on 2026-08-06 — which is exactly the window someone is most tempted
  to act on. Roughly 20% of vehicles land in a different tier at 7d vs 30d.
- **Windows are 7 / 30 / 60 / 90 days**, but the database only began collecting
  in June 2026, so 60d and 90d currently return the same ~57 days of data, and
  the prior-window trend is usually unavailable. When it is, the card says *why*
  ("only 109 of these 191 vehicles were driving in the prior 7 days") rather
  than a bare dash — on a fleet still being installed that absence is itself
  information.
- **The office filter is a re-render, never a refetch.** `load()` fetches the
  whole fleet once and stores it in `DATA`; `render()` applies `officeFilter`.
  Switching office costs zero API calls. Two things stay fleet-wide while
  filtered, on purpose: the dashed **fleet median** line (otherwise it collapses
  onto the office's own median and stops being a comparison), and `checkDrift()`
  (one small office sitting in one band says nothing about the policy bands).
  The "Worst office" card becomes **"Rank among offices"**, which is the
  question a branch manager actually has.
- **A speed is only reportable if the rule actually flagged it.** `topSpeed.html`
  matches each speeding `ExceptionEvent` to the trip containing it (same device,
  `activeFrom` inside `start`–`stop`) and ranks a driver on their fastest
  *flagged* trip. Without this the headline was a driver at 80 mph with zero
  speeding events — motorway driving, not an offence. Drivers who drove but were
  never flagged are excluded from the ranking and **named in the footer**: the
  posted-speed rule cannot fire where Geotab has no limit data for the road, so
  a fast unflagged run still deserves a look.
- **Prefer written policy to derived statistics when it exists.** `topSpeed.html`
  bands come from Megan Warner's escalations in the `#*-accidents-incidents`
  Slack channels, not from percentiles: **80+** is a final warning (termination
  if already on one), **85.1** is where her automated alert fires, **91+** is
  immediate termination. These superseded an earlier "85+ is termination"
  wording, so re-confirm with her before trusting them long-term. Percentile
  bands (as in `violin.html`) are the fallback for when no policy exists.
- **Thresholds are policy; statistics are description.** The bands are *fixed*
  so "Coaching review" means the same thing to every viewer regardless of data
  scope, and stays comparable month over month. The fleet-median line and
  per-office median markers are computed live and move with the data. Don't
  blur the two.
- **This database is young, so the bands will go stale — but not silently.**
  `checkDrift()` recomputes the empirical median/p80 on every render and raises
  a visible banner with re-derived values once ≥85% of vehicles land in one
  band. When it fires, update `LO`/`HI` in `violin.html` **and** this section.
- **Bands are calibrated to a specific rule** — currently `RulePostedSpeedingId`
  ("Speeding"). They are *not* transferable: the retired `RuleGpsSpeedingWindowId`
  fired ~4× as often, which is why the old bands were `<80 / 80–249 / ≥250`.
- **Band is encoded by background zone, never by dot color.** Band is a pure
  function of x-position, so coloring the dots by band spends the only free
  visual channel restating the axis. It also can't be made accessible: red↔green
  measures ΔE 2.8 under protanopia (validated, target ≥8) — no hex choice fixes
  that, because red and green *are* the confusion axis. Large labelled regions
  don't depend on hue discrimination. Dot fill carries driver-assigned instead.
- **The branch spreadsheets move into Geotab custom properties, not a file.**
  `PropertySet`/`Property` are supported but ship empty; `setup_fleet_properties.py`
  creates the *Mira Compliance* set. Geotab allows only **Boolean and String**
  property types — no Date, List or Number — so the checkboxes are Booleans and
  everything else is text. Properties are matched **by name at runtime, never by
  hardcoded id**: ids differ per database and a wrong one fails silently as a
  permanently blank column. `Remove` needs the whole entity; passing `{id}` alone
  returns a null-reference error.
- **Enterprise data enters Geotab, never this repo.** This repo is public, so a
  VIN/plate list must not be committed here — `.gitignore` blocks `*.json`
  outside `config/`, which is what caught it. Instead `backfill_ownership.py`
  writes ownership onto each vehicle as a Geotab custom property, so the add-in
  reads it through the authenticated session like everything else. The cost is
  that vehicles not yet in Geotab cannot appear in `fleet.html`;
  `rollout.html` covers that gap per location using counts only.
- **Ownership is Owned / Rental / Unconfirmed — there is no "Leased".**
  Enterprise splits out 29 "Leased Vehicle" rows, but they are TRAC leases:
  Mira carries the residual, so operationally they are owned and the split is
  noise on a register. `backfill_ownership.py` collapses it.
- **Writing a custom property means sending ONLY the ones that have values.**
  Geotab materialises every defined property on every device with no `value`
  key at all. Send those valueless entries back in a `Set` and the whole write
  blanks — including the property you were setting. The first backfill only
  worked because the array was still empty; the second silently nulled 11
  vehicles. Filter `c.get("value") not in (None, "")` before appending.
- **Ownership comes from the Enterprise export, but absence from it proves
  nothing.** The export is a monthly snapshot and new deliveries lag it, so a VIN
  it doesn't list is only a rental if the device is *also* named "Rental" —
  otherwise it is `Unconfirmed`. Guessing cost 17 owned Mavericks and Colorados
  their identity in the first pass. The two signals agree on the rest: 48 of the
  unmatched vehicles say "Rental" in their name and no owned or leased one does.
- **Enterprise records a garage city, Geotab records an office group.** Left
  unmapped the location filter lists both — "Brecksville" *and* "Cleveland" as
  separate places. `CITY_TO_OFFICE` normalises onto the Geotab name. Offices are
  an **allow-list**: Geotab keeps adding system groups, and a block-list lets each
  new one leak in as a fake location.
- **A GO9 with no VIN is not installed.** `rollout.html` treats a decoded VIN
  as the proof an install actually happened — a device can be registered,
  named and grouped and still be sitting in a box. Of 506 GO9s, 506 are
  registered but only 150 report a VIN. 346 are still *named after their own
  serial number*, which is the cleanest "never commissioned" signal there is.
  Cameras are their own entity (`Get Camera`, all `GO Focus Plus`) and join to
  the GO9 by **`deviceSerialNumber`, not by device id**.
- **Shipment counts are the one thing Geotab does not know.** The `ORDERED`
  table at the top of `rollout.html` is hand-maintained from ops and must be
  updated after each shipment. The add-in cross-checks its total against the
  live GO9 count and raises a banner when they disagree, so it complains
  rather than quietly showing a wrong "remaining" column.
- **On a horizontal bar chart the category axis is still `xaxis.categories`.**
  Omitting it doesn't just lose the labels — Apex falls back to a numeric axis
  and auto-ranges it, which collapses the plot into a fraction of its width.
- **Zero can mean "no access", not "nothing done".** Role scoping means an
  office user sees only their own devices, so `rollout.html` only claims a
  fleet-wide view when it can see ≥90% of the fleet; otherwise it hides
  locations it cannot see instead of showing them as zero.
- **Axis maxima come from a round step, not from rounding the data max.**
  Dividing a rounded max by a tick count is what produced ticks like
  `0/23/45/68/90`. See `axisScale()`.
- **Nothing is hidden without saying so.** Offices under the size cutoff,
  vehicles under the mileage floor, and outliers pinned at the axis edge are all
  named in the footer. A hidden office reads as an office with no problem.
- **Geotab caps a `Get` at 25,000 rows and tells you nothing.** `resultsLimit`
  is an upper bound you request, not the cap you get: ask for 50,000 or 100,000
  and a busy range still returns **exactly 25,000**, with no error, no flag and
  no continuation token. Any guard written as `rows.length >= resultsLimit` can
  therefore never fire. Fetch date ranges through a windowed helper that splits
  a window whenever it comes back at 25,000 (`fetchWindowed()` in both
  `violin.html` and `topSpeed.html`), and bound the recursion.
  This is not a cosmetic loss. In `violin.html` the rate is
  `events ÷ (miles ÷ 1000)`, so missing **trips** means missing **denominator**:
  measured on 2026-08-06 the 30-day view lost 3,660 trips, which hid 53 vehicles
  under the 50-mile floor (all but five of N Indiana, and all 23 of ATL North —
  an office with a *perfect* record showed as "hidden, 1 vehicle"), overstated
  106 rates by up to 147%, and put 10 vehicles in the wrong tier. The 60- and
  90-day views were worse. The tell that something is wrong: a **shorter**
  window rating **more** vehicles than a longer one.
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
- **The office map is hardcoded, and one label does not match Geotab.** Every
  add-in carries its own `LEAF` map of group id → office name (`fleetFunnel`,
  `violin`, `topSpeed`, plus `pull_risk_data.py` and friends in the tooling
  repo). Two consequences, both verified 2026-08-07 against the live Groups:
  - **13 of the 14 names match Geotab exactly. `b27B1` does not** — Geotab calls
    it **`Northern Indy`**, the dashboards display **`N Indiana`**. Nothing is
    miscounted (the id is what's matched), but a manager who filters by office
    in MyGeotab sees a different name than the chart shows. The tooling scripts
    are split too: most say "Northern Indy", `pull_risk_data.py` says
    "N Indiana". Pick one and align all of them.
  - **A new office group would be silently invisible.** Devices whose group is
    not in `LEAF` are dropped from the by-office views (`fleetFunnel` counts
    them as ungrouped and says so; the others just omit them). The parent groups
    — Mira Home, and the per-state groups — are correctly excluded, but a real
    new branch would look like it had no vehicles until someone edits every map.

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
locally in `~/Documents/geotab-scripts/` and is **intentionally not in this
repo** — it contains credentials and must stay off GitHub. This repo is
frontend/deployed assets only.
