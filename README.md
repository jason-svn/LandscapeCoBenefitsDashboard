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
canopy/softscape/native/lighting stats, an **embodied carbon & payback**
panel (the planting areas' one-off material GWP against the planting's
yearly CO₂e uptake, as the Dashboard app shows it) and a floor/softscape
breakdown table. In the header, a **growth-year slider** (for projects that
model the same planted layout at different tree ages — 0 yr as planted plus
5/10/15/20/25 or custom years from Growth Projections, or design options in
older models) picks the year everything shows. Near the bottom, **growth
curves** plot all five benefits on one chart in distinct colours (each as a %
of its own peak, so litres, kilograms and money share one axis, with the real
values in the tooltip and a table view), shading the years once every tree has
reached full size; click any translation card to chart that metric's yearly
figures as bars and bring its curve to the front.

No backend, no build step, no signup — open `index.html` in a browser (or
host it on GitHub Pages) and drop in a JSON file. Parsing happens entirely in
your browser; nothing you upload leaves it. Two features make small, public,
unauthenticated lookups using only data already in the export: the region map
(a reverse geocode, then a country's boundary shapes — see "Region map"
below) and the most-impactful-tree photo (a Wikipedia summary lookup by
species common name — see "Most impactful tree" below).

## Usage

1. In the **LandscapeDataManager** Dashboard app (Revit add-in), click
   **Export JSON**.
2. Open `index.html` here (double-click it, or visit the GitHub Pages URL).
3. Drag the exported `.json` file onto the page, or click **Choose JSON
   file**.
4. If the project models growth years, drag the **Growth year** slider at top
   (e.g. **0 / 5 / 10 / 15 / 20 / 25 yr**, or whatever years Growth
   Projections was run with) — it opens on the year the export marks primary
   (the Baseline / as-planted year when there is one). When the model also has
   real design options, pick one from the **Scenario** dropdown; the slider
   then steps through that option's years. Toggle **Annual / Lifetime**, and
   the species chart between **Chart / Pie / Table** view. Click any
   translation card (Avoided water run-off, CO₂, Air pollutants, Air
   generated, Cost saved), or a growth-curve legend entry, to make the growth
   bars plot that metric and bring its curve to the front; the curves follow
   the Annual / Lifetime toggle and switch between **Chart / Table**.

Don't have an export handy? Click **Load sample data** to preview the layout
with the bundled example (`sample-data/example-export.json`).

## What's in the export

The JSON schema this page expects (schema version 4; versions 3 and 2 still
load, version 2 just without the BNG panel) is produced by
`DashboardJsonExportService` in the
[LandscapeDataManager](https://github.com/jason-svn/LandscapeDataManager)
repo (`src/WWP.LandscapeDataManager.App.Dashboard`). All mass/volume figures
are always Metric (kg, m³) regardless of the unit system the export was made
under, so this page never has to guess a basis. Currency figures use
whatever currency the export was made in.

Top-level shape:

```jsonc
{
  "schemaVersion": 4,
  "project": { "title", "currency", "location": { "latitude", "longitude", "placeName" }, "generatedAt" },
  "scenarios": [
    {
      "label": "10 years",           // "Baseline · 10 years" when designOption is set
      "years": 10,                   // 0 (as planted) to 100; null when the name carries no year
      "isPrimary": true,
      "designOption": null,          // v4: the real design option this growth year belongs to
      "totals": { /* this design option's own annual + lifetime KPI numbers */ },
      "siteKpi": { /* canopy cover, native species ratio, lighting compliance, ... */ },
      "bng": { /* BNG baseline vs post-intervention units, net change %, 10% target, floor counts per role, byBroadHabitat */ },
      "species": [ /* per-species subtotals, scoped to this design option */ ],
      "floorTypes": [ /* per floor/softscape-type subtotals */ ]
    }
    // ...one entry per growth year (per design option, when there are real alternatives)
  ]
}
```

**Embodied carbon follows the Dashboard app.** `totals.floorTotalGwp` (and
each `floorTypes[].totalGwp`) is the planting areas' one-off GWP in kg CO₂e:
the product + transport stages of their materials from the landscape data
sheet, as Floor Calculator records it, and never negative — vegetation's
negative sheet "GWP" is its yearly sequestration, already counted as uptake.
The page shows payback = embodied carbon ÷ yearly CO₂e uptake (trees'
`co2EquivalentAnnualKg` + floors' `floorCarbonSequesteredAnnualKg` × 3.67)
for the selected growth year. Older exports that stored a negative GWP for
vegetation floors are summed per floor type with negatives left out, and the
panel asks for a recalculation.

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

**One scenario per growth year, never summed.** When a project models the
same planted layout at different tree ages, each growth year gets its own
entry in `scenarios` rather than being blended into one project-wide total —
a "25 years" figure added to a "5 years" figure for the same trees would be
meaningless. Growth years come from tree worksets named with a year count:
Growth Projections moves the original trees onto `LIM_Growth_0yr - Baseline`
(year 0, as planted) and copies them onto `LIM_Growth_<N>yr` for each year
asked for — 5/10/15/20/25 by default, any whole year up to 40. This page's
timeline (slider and growth charts) stops at 40 years too; later years in an
export (older versions allowed up to 100) are left out, and the
growth-curves subtitle says how many. A tree stops
growing once it reaches its type's mature size — the LDS max height/width
(`!_S_PLT_LDS_MaxHeight_Length` / `MaxWidth_Length`, downloaded from
Airtable's `Max_Height` / `Max_Width`) — or 25 years after planting without
one: the app caps each tree's modelled age there, so its i-Tree inputs and
results stop changing. Once every tree is capped, later years' figures are
identical, and the page shades that stretch of the growth curves. Older models used one design
option per year instead (e.g. "Growth Timeline : 5/10/15/20/25 Years"); those
exports still load as one timeline. When a model has growth-year worksets
*and* real design options, each scenario is one option at one year and
carries `designOption`, which the page offers as a Scenario dropdown. A
project with neither collapses to a single `"Primary model"` scenario, so
this still works for the common case. `years` is `null` when a name carries
no year — the page falls back to the full label then, and spaces the growth
curve evenly instead of to scale.

`location` is best-effort — Revit's Site Location defaults to an unset
placeholder on new projects, so the map panel gracefully shows a "location
not set" message instead when it's missing.

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
