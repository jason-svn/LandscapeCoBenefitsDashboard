# Landscape Co-Benefits Dashboard

A single self-contained HTML page that turns a **LandscapeDataManager** JSON
export into a PowerBI-style dashboard, styled to match the LandscapeDataManager
Dashboard app itself (mostly green, quiet shadcn-style neutral cards, a gold
accent for money): carbon / storm water / air-pollution metrics translated
into everyday equivalents (paired cards — raw value on the left, its everyday
equivalent on the right — joined by a wavy connector), an abstract country map
with the project's specific region highlighted, a cost-savings donut, a
species-composition donut plus ratio meters for canopy/softscape/native/
lighting stats, pollutant and species bar charts, and a floor/softscape
breakdown table. Near the bottom, a **growth-year selector** (for projects
that model the same planted layout at different tree ages via Revit design
options, e.g. 5/10/15/20/25 years) sits beside the translation cards — click
any card to drive the growth bar/curve charts by that metric.

No backend, no build step, no signup — open `index.html` in a browser (or
host it on GitHub Pages) and drop in a JSON file. Parsing happens entirely in
your browser; nothing you upload leaves it. The region map does make two
small, public, unauthenticated lookups (a reverse geocode, then a country's
boundary shapes — see "Region map" below), sending only the project's own
already-public latitude/longitude.

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

Don't have an export handy? Click **Load sample data** to preview the layout
with the bundled example (`sample-data/example-export.json`).

## What's in the export

The JSON schema this page expects (schema version 2) is produced by
`DashboardJsonExportService` in the
[LandscapeDataManager](https://github.com/jason-svn/LandscapeDataManager)
repo (`src/WWP.LandscapeDataManager.App.Dashboard`). All mass/volume figures
are always Metric (kg, m³) regardless of the unit system the export was made
under, so this page never has to guess a basis. Currency figures use
whatever currency the export was made in.

Top-level shape:

```jsonc
{
  "schemaVersion": 2,
  "project": { "title", "currency", "location": { "latitude", "longitude", "placeName" }, "generatedAt" },
  "scenarios": [
    {
      "label": "Growth Timeline : 10 Years (Primary)",
      "years": 10,
      "isPrimary": true,
      "totals": { /* this design option's own annual + lifetime KPI numbers */ },
      "siteKpi": { /* canopy cover, native species ratio, lighting compliance, ... */ },
      "species": [ /* per-species subtotals, scoped to this design option */ ],
      "floorTypes": [ /* per floor/softscape-type subtotals */ ]
    }
    // ...one entry per design option
  ]
}
```

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
