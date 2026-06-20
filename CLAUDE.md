# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**ComptaPerso** is a personal budget tracker for French-speaking users. The entire application lives in a single file: `index.html`. There is no build step, no package manager, no framework — just plain HTML, CSS, and vanilla JavaScript served directly in a browser.

## Running the app

Open `index.html` directly in a browser (double-click or `file://` URL). No server required. For live-reload during development:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

There are no tests, no linter config, and no CI pipeline.

## Architecture

### Single-file structure

All CSS lives in a `<style>` block in the `<head>`. All JavaScript is in a single `<script>` block at the bottom of `<body>`. The HTML defines the static shell; the JS renders everything dynamically.

### External dependencies (CDN only)

- **Chart.js 4.4.1** — bar charts and doughnut chart
- **QRCodeJS 1.0.0** — QR code generation (currently disabled inside iframes)
- **Google Fonts** — JetBrains Mono (`--mono`) and Syne (`--sans`)

### State management

A single global object `S` holds the current month's data:

```js
S = { solde: 0, epargne: 0, prelevements: [], revenus: [], cb: [], banks: {} }
```

- `prelevements[]` — direct debits / standing orders; each item: `{name, amount, checked}`
- `revenus[]` — income sources; each item: `{name, received, expected}`
- `cb[]` — debit card transactions; each item: `{name, amount, date, cat, checked}`
- `banks` — `{hello: bool, bourso: bool}` — CB payment confirmation per bank
- `epargne` — savings/investment amount for the month

### localStorage schema

```
cp_{year}_{month}   →  JSON string of the S object above
cpn_{year}_{month}  →  HTML string (rich-text note for the month)
```

`load(y,m)` reads a month, `save(s)` writes the current month, `saveAndSync(s)` writes then debounces a GitHub Gist push (1500 ms delay).

### Rendering pipeline

Navigation (month/year change) triggers:

```js
S = load(year, month)
render()   // → renderKPI(), renderPre(), renderRev(), renderCB(), renderBanks()
loadNote() // loads rich-text note into contenteditable div
```

Charts are only rendered when the "Graphiques" tab is active (lazy, via `switchTab`). Each chart is destroyed and recreated on each call; references are kept in the `charts` map to call `.destroy()` before recreating.

### Key financial formulas

```
Solde bancaire   = Revenus − Prélèvements pointés − CB pointées
Reste à dépenser = Revenus − Total prélèvements − Épargne − Total CB (unpointed logic not applied here)
```

"Pointé" means reconciled/cleared. Checking off a prélèvement or CB transaction subtracts it from the live balance.

### Cloud sync (GitHub Gist)

Optional. The user provides a GitHub personal access token (scope: `gist`) stored in `localStorage` under `gh_token`. On first sync a private Gist is created and its ID saved under `gist_id`. Subsequent saves PATCH the same Gist. `loadFromCloud()` is called at init and on month navigation when local data is empty.

`saveAndSync(s)` is the only function that should be called when mutating `S` — it writes to localStorage and schedules the Gist push.

## UI conventions

- CSS custom properties are defined in `:root` and used everywhere — never use raw color hex values in new CSS.
- Two font families: `var(--mono)` (JetBrains Mono) for numbers/labels, `var(--sans)` (Syne) for prose.
- Modals use the `.overlay` + `.modal` pattern; open by adding class `open`, close via `closeAll()`.
- Feedback is always shown via `toast(msg)` (2.2 s auto-dismiss), never via `alert()`.
- The app is entirely in French — all UI strings, labels, and `toast()` messages must remain in French.

## Key patterns to follow

- Any data mutation must go through `saveAndSync(S)` then call the appropriate `render*()` functions — never mutate `S` without saving.
- Notes are stored separately from the main state and must be saved via `saveNoteAndSync()` or `saveNote()`.
- When modifying the render functions, prefer regenerating the full `innerHTML` of list containers rather than patching individual DOM nodes (existing pattern).
- Chart instances must be destroyed before recreation via `destroyChart(id)` to avoid canvas conflicts.
