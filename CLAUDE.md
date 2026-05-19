# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment

Push to `main` → GitHub Actions (`.github/workflows/static.yml`) → GitHub Pages.  
Live URL: **https://andresdiazm.github.io/hospitalizados_hoy/**

There is no build step. The entire app is a single `index.html` file with zero local dependencies. To publish a change:

```powershell
git add index.html
git commit -m "descripción"
git push
```

## Architecture

Everything lives in `index.html` (~1100 lines). Structure within the file:

| Section | Lines approx. | Content |
|---|---|---|
| CSS | `<style>` block | Design tokens, component styles, responsive breakpoints |
| HTML | `<body>` | Three states: `#upload-state`, `#report-state`, `#btn-export` |
| JS | `<script>` | All logic — no modules, no bundler |

**External CDNs (no local install):**
- `SheetJS` — reads `.xls`/`.xlsx` files in the browser
- `Chart.js 4.4` — renders the stacked bar chart
- Google Fonts — Bricolage Grotesque, Public Sans, JetBrains Mono

## Data Flow

```
Excel file (drag/drop or input)
  → SheetJS XLSX.read()
  → processWorkbook()          # parses rows, classifies services, returns APP.data
  → APP state updated
  → showReport()               # triggers all render functions
```

**Expected Excel columns:** `Servicio_Perteneciente`, `Diagnostico`, `Estada_Hospital`, `Estada_Servicio`

## State (`APP` object)

```js
APP = {
  data: null,          // result of processWorkbook(): { date, global, services, allRows }
  activeFilter: 'all', // bed type: 'all' | 'critica' | 'materno' | 'general'
  activeService: null, // specific service name, or null for all
  chart: null,         // Chart.js instance (destroyed/recreated on filter change)
}
```

Both filters combine: `filteredRows()` applies `activeFilter` then `activeService`.  
`filteredServices()` does the same for the chart and service table.  
When `activeFilter` changes, `activeService` resets to `null` and `renderServiceFilter()` rebuilds the service chips.

## Key Functions

| Function | Purpose |
|---|---|
| `processWorkbook(wb, filename)` | Parses SheetJS workbook → `{ date, global, services[], allRows[] }`. Each service object includes its raw `rows[]` for bucket computation. |
| `classifyService(name)` | Maps service name → `'critica' \| 'materno' \| 'general'` based on keywords (UCI, UTI, UPC, CORONARIA, GINECOLOG, etc.) |
| `computeMetrics(rows)` | Returns `{ n, medianHosp, medianServ, pct30, pct10 }` for a row set |
| `computeStayBuckets(rows)` | Returns 6-element array of % per range (0–3, 4–7, 8–10, 11–20, 21–30, 31+ days) using `estadaHospital` |
| `computeDiagnoses(rows)` | Groups by `diagnostico`, returns top 10 + "Otros" |
| `renderServiceFilter()` | Rebuilds the horizontal-scroll service chip bar based on current `activeFilter` |
| `renderChart(services)` | Destroys and recreates the Chart.js stacked 100% bar chart |
| `buildExportHTML(data)` | Returns a self-contained HTML string (inline CSS + data + Chart.js CDN script) for download |

## Design Tokens

Brand colors are `--chsj-blue: #0061BE` and `--chsj-red: #E51D2F`.  
The logo is embedded as `LOGO_B64` (base64 PNG) in the JS block — do not replace without re-encoding.

Stay bucket colors run green → red: `#1F8A5B → #6AAF3D → #B8770E → #E07B1A → #E51D2F → #7B0D14`.

## Adding or Changing Visualizations

- The chart is recreated on every filter change via `updateServicioView()` → `renderChart(services)`.
- To add a new metric to the KPI grid, add it to `computeMetrics()` and update `renderKPIs()`.
- `buildExportHTML()` must be kept in sync with any chart or table changes — it is a standalone HTML generator that duplicates chart logic as inline JSON.
