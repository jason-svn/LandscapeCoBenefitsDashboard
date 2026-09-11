# Landscape Co-Benefits Dashboard

A single self-contained HTML page that turns a **LandscapeDataManager** JSON
export into a PowerBI-style dashboard: project location on a map, carbon /
storm water / air-pollution KPIs with everyday "equivalent to" comparisons, a
cost-savings donut, pollutant and species bar charts, site & biodiversity
stats, and a floor/softscape breakdown table.

No backend, no build step, no signup — open `index.html` in a browser (or
host it on GitHub Pages) and drop in a JSON file. Parsing happens entirely in
your browser; nothing is uploaded anywhere.

## Usage

1. In the **LandscapeDataManager** Dashboard app (Revit add-in), click
   **Export JSON**.
2. Open `index.html` here (double-click it, or visit the GitHub Pages URL).
3. Drag the exported `.json` file onto the page, or click **Choose JSON
   file**.
4. Toggle **Annual / Lifetime**, and the species chart between **Chart /
   Table** view.

Don't have an export handy? Click **Load sample data** to preview the layout
with the bundled example (`sample-data/example-export.json`).

## What's in the export

The JSON schema this page expects (schema version 1) is produced by
`DashboardJsonExportService` in the
[LandscapeDataManager](https://github.com/jason-svn/LandscapeDataManager)
repo (`src/WWP.LandscapeDataManager.App.Dashboard`). All mass/volume figures
are always Metric (kg, m³) regardless of the unit system the export was made
under, so this page never has to guess a basis. Currency figures use
whatever currency the export was made in.

Top-level shape:

```jsonc
{
  "schemaVersion": 1,
  "project": { "title", "currency", "location": { "latitude", "longitude", "placeName" }, "generatedAt" },
  "totals": { /* project-wide annual + lifetime KPI numbers */ },
  "siteKpi": { /* canopy cover, native species ratio, lighting compliance, ... */ },
  "species": [ /* per-species subtotals */ ],
  "floorTypes": [ /* per floor/softscape-type subtotals */ ]
}
```

`location` is best-effort — Revit's Site Location defaults to an unset
placeholder on new projects, so the map panel gracefully shows a "location
not set" message instead when it's missing.

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
