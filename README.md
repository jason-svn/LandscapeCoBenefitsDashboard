# Landscape Co-Benefits Dashboard

**Live dashboard:** <https://jason-svn.github.io/LandscapeCoBenefitsDashboard/> — open it, then
drop in a JSON file from the LandscapeDataManager Dashboard's **Export JSON** (or click
**Load sample data**). Exports come from
[LandscapeDataManager](https://github.com/jason-svn/LandscapeDataManager), whose
[quick-start guide](https://jason-svn.github.io/LandscapeDataManager/) covers installing the Revit add-in.

A single self-contained HTML page that turns a **LandscapeDataManager** JSON
export into a PowerBI-style dashboard, styled to match the LandscapeDataManager
Dashboard app itself (mostly green, quiet shadcn-style neutral cards, a gold
accent for money): carbon / storm water / air-pollution metrics translated
into everyday equivalents (paired cards — raw value on the left, its everyday
equivalent on the right — joined by a wavy connector), an abstract country map
with the project's specific region highlighted, a "most impactful tree"
highlight card (top species by carbon sequestered, with a real reference
photo), and a pastel categorical palette (distinct from the page's brand
green) for anything that identifies categories rather than magnitude: the
cost-savings donut, the species-composition donut, per-species colors shared
across the species bar/pie/table views, and per-pollutant colors shared
across the air-pollutants bar/pie/table views — plus ratio meters for
canopy/softscape/native/lighting stats and a floor/softscape breakdown table.
Every data table uses the desktop Dashboard's table language: columns that
belong together sit in a pale swim lane named in capitals above them, each
header is its name over its unit, and the cells hold plain right-aligned numbers.
Near the bottom, a
**growth-year selector** (for projects that model the same planted layout at
different tree ages via Revit design options, e.g. 5/10/15/20/25 years) sits
beside the translation cards — click any card to drive the growth bar/curve
charts by that metric. With design scenarios, the growth curve draws every
scenario on one chart (the existing site as a grey dashed reference), with each
scenario's value at the year being viewed underneath; click a point or a
scenario to view it there.

No backend, no build step, no signup — open `index.html` in a browser (or
host it on GitHub Pages) and drop in a JSON file. Parsing happens entirely in
your browser; nothing you upload leaves it. Two features make small, public,
unauthenticated lookups using only data already in the export: the region map
(a reverse geocode, then a country's boundary shapes — see "Region map"
below) and the most-impactful-tree photo (a Wikipedia summary lookup by
species common name — see "Most impactful tree" below). A schema 9 export
with a 3D scene also downloads the [three.js](https://threejs.org) library
(0.170.0) from jsDelivr to draw it — a plain library download; no project data
is sent anywhere (see "3D site" below).

## Usage

1. In the **LandscapeDataManager** Dashboard app (Revit add-in), click
   **Export JSON**.
2. Open `index.html` here (double-click it, or visit the GitHub Pages URL).
3. Drag the exported `.json` file onto the page, or click **Choose JSON
   file**.
4. If the project uses growth-year design options, pick a stage (e.g. **5 yr
   / 10 yr / 15 yr / 20 yr / 25 yr**) from the tabs at top — it defaults to
   whichever design option Revit has flagged Primary. Toggle **Annual /
   Lifetime**, and the species chart between **Chart / Table** view. Click
   any translation card (Avoided water run-off, CO₂, Air pollutants, Air
   generated, Cost saved) to make the growth charts below plot that metric.
5. With a schema 9 export, the **3D site** panel shows the scenario's trees
   and planting areas. Drag to pan, right-drag to rotate, scroll to zoom, click
   a tree or planting area for its details. Click rows of the result board
   beside it to show only those species or landscape types (Shift+click
   selects a range); **Show all** clears the selection. With a schema 10
   export, **Colour by** colours the 3D as one of the model's Revit views does
   with its filters (e.g. O₂ produced per tree in bands), with a legend.

Don't have an export handy? Click **Load sample data** to preview the page with
the bundled example (`sample-data/example-export.json`): a schema 10 export of
the LIM sandbox model (LAN_LIM Arboris_EWR Sandbox) with its 3D scene and the
colour schemes of its Revit views — three schemes (A, B and C, each with the
whole existing site, retained or demolished) and the existing site — made on
2026-10-25 by the export code of LandscapeDataManager v1.8.0. It is 1.6 MB, so
the page fetches it when the button is clicked rather than carrying it inline;
that works on GitHub Pages or any web server, but a page opened as a local file
can't fetch it, so drop the file onto the page instead.

## What's in the export

The JSON schema this page expects (schema version 10; versions 2 to 9 still
load, without the panels their fields feed) is produced by
`DashboardJsonExportService` in the
[LandscapeDataManager](https://github.com/jason-svn/LandscapeDataManager)
repo (`src/WWP.LandscapeDataManager.App.Dashboard`). All mass/volume figures
are always Metric (kg, m³) regardless of the unit system the export was made
under, so this page never has to guess a basis. Currency figures use
whatever currency the export was made in.

Top-level shape:

```jsonc
{
  "schemaVersion": 10,
  "project": { "title", "currency", "location": { "latitude", "longitude", "placeName" }, "generatedAt" },
  "scene": { /* schema 9, optional: the trees and planting areas in 3D — see "3D site" below */ },
  "scenarios": [
    {
      "label": "Growth Timeline : 10 Years (Primary)",
      "years": 10,
      "isPrimary": true,
      "totals": { /* this design option's own annual + lifetime KPI numbers */ },
      "siteKpi": { /* canopy cover, native species ratio, lighting compliance, ... */ },
      "bng": { /* BNG baseline vs post-intervention units, net change %, 10% target, floor counts per role, byBroadHabitat */ },
      "species": [ /* per-species subtotals, scoped to this design option */ ],
      "floorTypes": [ /* per floor/softscape-type subtotals */ ],
      "sceneTrees": [ /* schema 9: indices into scene.trees */ ],
      "sceneFloors": [ /* schema 9: indices into scene.floors */ ]
    }
    // ...one entry per design option
  ]
}
```

**BNG follows the metric's on-site headline.** Each floor's Revit phases decide
its row of the Statutory Biodiversity Metric: floors created in the project's
first phase (e.g. Existing) are the baseline (sheet A-1) — retained, lost if
demolished, or enhanced (sheet A-3) when `!_S_PLT_BNGInput_Enhanced_YesNo` is
ticked — and floors created later (e.g. New Construction) are new habitat
(sheet A-2). `bng.baselineUnits` is the A-1 total; `bng.postInterventionUnits`
is retained + enhanced + created units; `bng.netChangePercent` is their change
(null without a baseline), checked against the statutory 10% in
`bng.meetsStatutoryNetGain`. Only floors whose stored result still matches
their inputs, area and phase count; stale or unassessed ones are counted in
`staleFloorCount` / `notAssessedFloorCount` and make `isComplete` false.
Hedgerows, watercourses, off-site units and trading rules aren't modelled.

**One scenario per design option, never summed.** If a project uses design
options to model the same planted layout at different tree ages (this WWP
project's "Growth Timeline : 5/10/15/20/25 Years" set), each option gets its
own entry in `scenarios` rather than being blended into one project-wide
total — a "25 years" figure added to a "5 years" figure for the same trees
would be meaningless. A project with no such design options in play
collapses to a single `"Primary model"` scenario, so this still works for
the common case. `years` is parsed from the design option's name (`5`, `10`,
`15`, `20`, or `25`) and is `null` when it can't be — the page falls back to
showing the option's full label in that case.

**Maintenance and data notices (schema 6).** The maintenance panel leads the
page: yearly maintenance for trees (per tree) and planting areas (per m²) from
the WWP sheet's rates, against the cost saved, and the net value. It shows
per year in both periods, since a maintenance rate has no lifetime figure, and
stays hidden for exports without maintenance figures. Above it, notices count
trees whose i-Tree results are out of date (`totals.staleTreeCount`: their
inputs changed after calculation; they stay in the totals with their old
results), trees with no result (`totals.treesExcludedFromTotals`) and trees
using WWP-sheet estimates (`totals.estimatedTreeCount`). With growth-year
worksets and i-Tree forecasts, `benefitsFromForecast` is true and each growth
year's benefits are every planted tree's forecast for that year.

**Connected habitat.** When the Connectivity tool has scored a scenario,
`siteKpi.connectedHabitatPercent` is its equivalent connected area (EC) as a
share of the habitat area (100% when every tree and planting area reaches
every other), and `siteKpi.equivalentConnectedAreaSquareMeters` the EC itself,
averaged over the tool's dispersal distances. It's the scheme as drawn (new
trees at the design year), the same in every growth year. The page shows it
in place of the hand-entered `habitatConnectivityScore`, which stays as the
fallback. Both fields are absent for scenarios not scored yet.

`location` is best-effort — Revit's Site Location defaults to an unset
placeholder on new projects, so the map panel gracefully shows a "location
not set" message instead when it's missing.

## 3D site

A schema 9 export can carry a `scene`: every tree and every planting area of
the model, which the **3D site** panel draws with
[three.js](https://threejs.org). The library (0.170.0) is fetched from
cdn.jsdelivr.net only when the file has a scene. It is a plain library
download: the export itself never leaves the browser. Without a connection the
panel says so and its tables still work; without WebGL likewise.

```jsonc
"scene": {
  "units": "m",
  "origin": [x, y, z],        // Revit internal coordinates; positions below are relative to it (X east, Y north, Z up)
  "growth": "forecast",       // "forecast" | "worksets" | "drawn"
  "maxYear": 25,
  "profiles": [ [ [trunkHeight, crownHeight, crownRadius, trunkRadius], ... ] ],   // one row per year 0..maxYear
  "trees": [ { "id", "speciesCode", "commonName", "type", "xyz", "drawn": [trunkHeight, crownHeight, crownRadius, trunkRadius],
               "profile", "phaseRole", "designOption" } ],
  "floors": [ { "id", "ldsType", "type", "surfaceClass", "areaM2", "holes", "phaseRole", "designOption",
                "positions": [x, y, z, ...], "indices": [a, b, c, ...] } ],
  "colourSchemes": [          // schema 10, optional: the model's views that colour trees or floors with filters
    { "name": "O2", "parameter": "!_S_PLT_Result_OxygenProducedAnnual_Mass", "unit": "kg/yr",
      "rules": [ { "label": "4–8", "filter": "WWP_Tree O2 04-08 kg per yr", "colour": "#aad669", "min": 4, "max": 8 } ],
      "trees": [ 1, 3, -1, ... ],        // per scene tree: its entry in rules, -1 for none; null when no rule is for trees
      "floors": null,
      "treeValues": [ 7.67, ... ] }      // with one number parameter behind every rule: each tree's value in `unit`
  ]
}
```

- **What's shown follows the numbers.** Each scenario lists the trees and
  floors its figures come from (`sceneTrees`, `sceneFloors`, indices into the
  scene), so the 3D always shows what the KPIs count: the scheme's design
  options, demolished elements left out, the existing site as it stood.
- **Tree sizes.** A tree is a trunk cylinder with an ellipsoid crown on top,
  as the Revit planting family draws it (trunk = H/3, crown = 2H/3). With
  `growth` `"forecast"` each scenario year takes the tree's `profile` row for
  that year, the family's formulas applied to i-Tree's forecast, so moving the
  growth-year slider grows the trees; otherwise the tree is drawn at `drawn`.
- **Planting areas** are each floor's triangulated top face, holes already cut
  out, colored by landscape type (water blue, hard surfaces grey, vegetation
  greens).
- **Result board.** Beside the 3D, two tables in the desktop Dashboard's
  swim-lane style list the scenario's trees by species and its planting areas
  by landscape type, following the Annual / Lifetime toggle (planting areas
  are annual only). Click rows to show only those species or types in the 3D
  (each table filters its own kind; nothing selected shows everything;
  Shift+click selects every row from the last one clicked); click an element in
  the 3D to see its details and its row. Species keep the colors they have in
  the charts above, the same in every scenario and year.
- **Colour by** (schema 10). The model's Revit views that colour trees or
  floors with view filters — every view, not only those on sheets; views that
  colour the same way are listed once, under the shortest name on a sheet —
  become choices in the panel's **Colour by** list. Revit itself decided which
  filter colours each element when the file was exported (the first of the
  view's filters that takes it, as Revit applies them), so the 3D matches the
  view: the colours are the filters' surface (or line) colours, and an element
  no filter takes is grey. A kind the view doesn't colour keeps its usual
  colours. The legend lists each band or value with how many of the scenario's
  elements take it; click entries (Shift+click for a range) to show only
  those. The values are the parameters as they are in Revit, so they don't
  change with the growth-year slider. The info card adds the element's value
  and band. On a phone the legend starts folded to its title.
- **Highlight.** With the panel's **Highlight** switch on (the default,
  remembered in the browser), the tree or planting area under the pointer
  lights up, and the one you click stays highlighted — an orange crown and a
  ring on the ground for a tree, an orange tint and outline for a planting area
  — while you change scenario or year, its card following its size. Click empty
  ground, the card's ×, or press Esc to let it go.

## Region map

Not a real basemap — no map tiles, no pan/zoom. It's an abstract illustration
of the country the site is in (grey), with the one admin-1 region (state /
province) containing the site filled in and a pin at the exact point, in the
spirit of a choropleth like the ones i-Tree-style reports use. Two lookups
build it, both cached (in-memory for the session, and in `sessionStorage` for
the boundary shapes) so a period/scenario toggle never re-fetches:

1. **[BigDataCloud's client-side reverse geocoder](https://www.bigdatacloud.net/geocoding-apis)**
   (free, no key, CORS-enabled by design) turns the lat/lng into a country
   code. Nominatim was tried first and rejected here — its demo server's CDN
   intermittently drops the CORS header on a cache hit, which silently breaks
   browser `fetch()` about half the time.
2. **[geoBoundaries](https://www.geoboundaries.org)** supplies that country's
   simplified ADM1 boundary shapes (a public academic project, CC BY). Its
   download links point at Git-LFS-tracked files on GitHub; this page resolves
   them straight to `media.githubusercontent.com` itself rather than letting
   the browser follow `github.com/.../raw/...`, whose redirect response
   carries a header that breaks `fetch()` before the redirect is ever taken.

Point-in-polygon (plain ray casting) picks the matching region client-side.
If either lookup fails, or geoBoundaries has no ADM1 data for that country,
the panel falls back to showing the location's plain coordinates instead of
erroring.

## Most impactful tree

Whichever species ranks #1 by carbon sequestered (the same ranking the "Top
species" chart uses) gets its own highlight card with a real reference photo,
not a decorative icon — looked up by common name via
[Wikipedia's page-summary API](https://en.wikipedia.org/api/rest_v1/) (free,
no key, reliable CORS). Common tree names resolve well in practice, and a
genus-only entry like `"Oak, spp. (Quercus)"` gets a second attempt against
the parenthesized genus name. A name is never guaranteed to match, though, so
a failed or ambiguous lookup always falls back to a plain placeholder rather
than risk showing the wrong tree — it never blocks the rest of the page, and
results are cached per species name for the session.

## Methodology / equivalency figures

The "≈ 705 baths" style callouts are computed client-side from constants at
the top of `index.html`'s `<script>` block (`EQUIV`). Where a defensible,
citable public figure exists it's used and cited in the page footer (EPA's
Greenhouse Gas Equivalencies Calculator for driving miles; O₂-from-CO₂ via
the 32:44 stoichiometric mass ratio). The cigarette and cups-of-coffee
equivalencies are illustrative round numbers for public communication, not
regulatory claims — edit `EQUIV` to match your organization's approved
source before using this for anything published externally.

## Hosting

This is a static site — any static host works (GitHub Pages, Netlify, a
plain file share). To publish via GitHub Pages: Settings → Pages → Deploy
from branch → `main` / `/ (root)`.
