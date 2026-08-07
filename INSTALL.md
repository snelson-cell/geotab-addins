# Installing the Mira Fleet Charts add-ins in MyGeotab

The add-ins are already built and hosted on GitHub Pages, so there's nothing to
host — you just register them in MyGeotab once.

**Live URLs**
- Fleet by Location — https://snelson-cell.github.io/geotab-addins/fleetFunnel.html
- Fleet Roster — https://snelson-cell.github.io/geotab-addins/fleet.html
- GO9 & Camera Rollout — https://snelson-cell.github.io/geotab-addins/rollout.html
- Speeding Rate by Vehicle — https://snelson-cell.github.io/geotab-addins/violin.html
- Top Speed by Driver — https://snelson-cell.github.io/geotab-addins/topSpeed.html

## Register in MyGeotab (one time)
1. In MyGeotab: **Administration → System… → Add-Ins**.
2. Click **New Add-In** → **Configuration** tab.
3. Paste the full contents of [`config/mira-fleet-charts.config.json`](config/mira-fleet-charts.config.json) → **OK**.
   - It's unsigned, so MyGeotab shows a trust prompt — accept it (fine for internal use).
4. Click **Save** (top of the Add-Ins page).
5. **Refresh** MyGeotab. Five items appear under the **Add-Ins** menu, in
   alphabetical order: **Fleet by Location**, **Fleet Roster**, **GO9 &
   Camera Rollout**, **Speeding Rate by Vehicle**, **Top Speed by Driver**.

To add more charts later, add another `items[]` entry pointing at the new
`…github.io/geotab-addins/<file>.html`, and re-paste the config.

## Access / permissions (how OM vs Exec see it)
- **Who sees the menu item:** driven by Security Clearance. Drivers
  (`NothingSecurity`) don't use the portal, so they never see it. To limit it to
  managers/execs, restrict the Add-In to `Supervisor` + `Everything` clearances
  in **Administration → Users → (clearance) → Add-Ins**.
- **What data each person sees:** automatic. The charts call the API in the
  viewer's own session, so Geotab scopes results to their group — a Chicago OM
  sees Chicago, an exec at the root sees the whole fleet. No per-role code.

## Change where it lives in the menu
In the config, `"path"` sets the menu section: `ActivityLink/`, `SafetyLink/`,
`EngineMaintenanceLink/`, `AdministrationLink/`. Change and re-save.

All five currently use `ActivityLink/`, but in the current MyGeotab UI they
render under their own **Add-Ins** section in the left nav, not under Activity —
so don't expect the path name to predict the heading you see. Menu items are
listed in the order the `items` array is written; ours are kept alphabetical.

## Updating a chart
Edit the file, `git commit`, `git push`. Pages rebuilds in ~30–60 s at the same
URL — just hard-refresh (Cmd+Shift+R) in MyGeotab. No config change needed.

Confirm it actually shipped by comparing hashes, not by looking at the page:

```bash
curl -s "https://snelson-cell.github.io/geotab-addins/violin.html" -o /tmp/live.html
shasum -a 256 violin.html /tmp/live.html   # digests must match
```

If a build is stuck or GitHub Actions is down, request one directly —
`gh api -X POST repos/snelson-cell/geotab-addins/pages/builds`. See the README's
"When Actions is down" section.

## Troubleshooting
- **Blank chart area:** the ApexCharts CDN was blocked. Inline the library:
  download `https://cdn.jsdelivr.net/npm/apexcharts@3.49.1/dist/apexcharts.min.js`
  and replace the `<script src="…apexcharts…">` line with `<script>…contents…</script>`.
- **"Add-in failed to load":** the URL is wrong or not https. Open it in a
  browser tab — it loads but renders nothing outside MyGeotab (expected; it only
  draws once Geotab hands it the `api`).
- **Empty / "0 vehicles":** the signed-in user's scope has no matching data, or
  their clearance can't read Devices. Test with an `Everything` user.

## How hosting is set up (reference)
Repo `snelson-cell/geotab-addins`, GitHub Pages serving from `main` / root.
Add-in HTML files sit at the repo **root** so their Pages URLs stay stable —
don't move them into subfolders or the installed add-in URLs break.
