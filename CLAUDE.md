# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

ReviewRadar is a single-file, zero-dependency web dashboard for visualizing performance review data. The entire application lives in `index.html` — no build step, no npm, no server required. Open the file in a browser and drag in an `.xlsx` file.

## Development

No build or install step. Open `index.html` directly in a browser to run. There are no tests, linters, or CI pipelines.

## Architecture

`index.html` is structured as one self-contained file with three logical sections:

**CSS** (`<style>`) — CSS custom properties for the dark theme (`--card`, `--border`, `--accent`, `--muted`, `--dim`, `--text`, `--raised`, `--surf`). Component classes follow a prefix convention: `.sm-` (scoremap), `.sg-` (scoremap group), `.ind-` (individual), `.sheat-` (skill heat cells in individual view), `.avg-` (heatmap averages), `.person-` (person row/selector).

**HTML** — Two top-level screens: `#upload-screen` (drag-drop zone + format hint table) and `#dashboard` (tab navigation + four view containers: `#view-radar`, `#view-heatmap`, `#view-individual`, `#view-scoremap`).

**JavaScript** — All logic is inline. Key flow:

1. **`parseXlsx(arrayBuffer)`** — reads the file using SheetJS (`XLSX`), builds two passes:
   - Pass 1: discovers schema — `groupShortMap`, `skillShortMap`, `skillFullMap`, `templateRole`, `roleSkillOrder` (skill order per platform)
   - Pass 2: builds `people` array and `meta` object
   - Returns `{people, meta}`

2. **`initDashboard({people, meta})`** — sets globals `PEOPLE` and `META`, wires up tab switching, calls the four view builders.

3. **View builders** — `buildRadarView()`, `buildHeatmap()`, `buildIndividual()`, `buildScoremap()` — each generates HTML strings and inserts via `innerHTML`, then attaches event listeners.

**Global state**: `PEOPLE` (array of person objects), `META` (schema/colors/maps), `RC`/`IC` (Chart.js radar instances keyed by canvas id — must be destroyed before re-creating).

**Persistence**: The raw `.xlsx` ArrayBuffer is saved to `localStorage` as base64 (`reviewradar_file`) after a successful parse and auto-restored on page load. Cleared only when the user clicks "Загрузить другой файл".

## Data model

Each person object in `PEOPLE`:
```
{ id, n (name), tmpl, team, res (result/grade), rev (reviewer name),
  rt (reviewer total), st (self total), role_type (platform string),
  sk: [{g, s, f, self, rv}],   // skill entries: group-short, skill-short, skill-full, values
  groups: {groupShort: avgScore} }
```

`META` contains: `areas`, `groupColors`, `roles`, `roleColors`, `teams`, `teamColors`, `resultColors`, `roleSkillOrder` (Map: platform → [{s,g,f}]), `skillFullMap` (Map: short→full).

## Color and scoring helpers

- `sColor(v)` — background colour for a numeric score (transparent → dark blue → teal → gold)
- `sText(v)` — text colour: `#fff` for v≤64, `#f0d060` above, `C.dim` for zero
- `gapCol(g)` — gap colour (red = overrated, green = underrated)
- `pColor(role)` / `tColor(team)` / `gColor(group)` — lookup from META colour maps
- `GROUP_PALETTE`, `TEAM_PALETTE`, `COLOR_Q` — ordered palettes assigned by first-appearance index

## Optional columns

`template_name`, `answer_weight` (int multiplier on `answer_value`, default 1), `comment`, `team` — all optional. `answer_weight` is applied at parse time; the multiplied value is what flows through everywhere.
