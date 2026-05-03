# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal health-tracking PWA for a male user (30, 1.70m, 93kg → 78kg goal) with an L5-S1 disc herniation (no gym, no running — physiotherapy + walking only) who travels weekly between **Tetouan** (4 days, home-cooked meals) and **Casablanca** (3 days, fast-food survival). UI is in **French**.

## Architecture

The entire application lives in a **single `index.html` file** at the repo root. There is no build step, no package manager, no bundler, no test suite. Open the file in a browser (or `python3 -m http.server` and visit `localhost:8000`) and it runs.

External dependencies are loaded via CDN inside the `<head>`:
- **Chart.js** — all charts (line, bar, doughnut)
- **Lucide** — icons (rendered via `lucide.createIcons()` after each tab render)

PWA wiring: the `manifest.json` is embedded inline as a `data:` URL on the `<link rel="manifest">` tag — keep it that way to preserve the single-file constraint.

### Code layout inside `index.html`

1. **`<style>` block** — CSS variables drive theming. Two color systems live side-by-side:
   - Light/dark mode: toggled by setting `data-theme="dark"` on `<html>`.
   - Semantic palette: `--green` (Tetouan / good), `--orange` (Casablanca / warning), `--red` (pain / hunger emergency), `--blue` (hydration). Do not hardcode hex values in components — always reference the variables so dark mode keeps working.

2. **`<body>` markup** — five `<section class="tab">` panels (one per tab: Today, Meal Plan, Progress, Physio, Nutrition), a fixed bottom nav, a floating hunger-emergency button, and modal overlays (recipe detail, hunger emergency, settings). Only one tab section is visible at a time (`.tab.active`).

3. **`<script>` block** — organized as:
   - **State & persistence** — a single `state` object mirrored to `localStorage` under one root key. All reads go through `loadState()` / writes through `saveState()`; never touch `localStorage` directly from feature code or you will desync the in-memory copy.
   - **Static data** — `MEALS_TETOUAN` and `MEALS_CASABLANCA` (7-day arrays, each entry has `breakfast`, `snack`, `lunch`, `dinner` with `name`, `ingredients`, `protein`, `calories`, `carbs`, `fat`).
   - **Mode resolution** — `getTodayMode()` reads `state.settings.casablancaDays` (array of weekday indices, 0=Sunday) and returns `'tetouan' | 'casablanca'`. The travel-mode toggle at the top of the screen overrides this for the current day only.
   - **Per-tab `render*()` functions** — each tab has its own render function that rebuilds its DOM from `state`. After any state mutation, call the affected tab's render function (and `lucide.createIcons()` if new icons were inserted).
   - **Chart instances** — Chart.js instances are stored on a `charts` object and **destroyed before re-render** (`charts.weight?.destroy()`); skipping this leaks canvases and breaks tooltips.
   - **Notifications** — `Notification.requestPermission()` is called on first interaction (not on load — Chrome blocks it). Reminders at 08:00, 13:00, 21:00 are scheduled with `setTimeout` chains computed from `Date.now()`.

### Key invariants

- **Targets are constants**: 1800 kcal, 150g protein, 170g carbs, 65g fat, 10 glasses water, 30 min walking. Goal weight 78kg, start weight 93kg (15kg goal). If these change, search for the literal numbers — they are referenced both in the targets object and in chart axis configs.
- **Streaks** (plan streak in Progress, physio streak in Physio) are computed from a date-keyed log, not from a counter. Resetting requires clearing the log entries, not zeroing a number.
- **Nutrition tab macros are derived**, not stored — they sum the macros of meals checked off in Today for the current date. Don't add a separate "logged macros" store; compute from the meal-completion log.
- **Photos** are stored as base64 data URLs in localStorage. Cap at one per ISO week (`YYYY-Www` key) and warn if total localStorage usage gets close to the ~5MB browser quota.

## Git workflow

Active development branch: `claude/health-tracking-pwa-MJRmw`. All commits and pushes go there unless the user explicitly says otherwise.
