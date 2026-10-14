# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file vanilla HTML/CSS/JS Kanban board ([index.html](index.html)) for a fictitious bank's internal IT PMO (demo/training tool). There is no build, lint, or test tooling; run it by opening `index.html` in a browser (no server needed).

## Hard constraints (from the original spec)

- Vanilla only: no frameworks, bundlers, npm, CDN scripts, web fonts, or image files. System font stack and inline SVG/Unicode for icons.
- Everything stays in `index.html` (markup, `<style>`, `<script>`).
- No persistence: no localStorage/sessionStorage/IndexedDB/cookies. A refresh must reset to seeded demo data (the UI says so).
- Use a neutral "IT PMO" text wordmark; no real UOB logo, trademarks, or imitation of official systems.
- CSS: custom properties for palette/spacing, no `!important`.
- Only network call is the FormSubmit AJAX endpoint; the notification email address must appear only in `FORMSUBMIT_ENDPOINT`.

## Architecture

- `state` object is the single source of truth: `tasks`, `filters`, plus UI bits (`theme`, `confirmDeleteId`, `dragId`, `nextId`, `sending`, `lastSubmitAt`).
- Rendering is full re-render: `renderBoard()` rebuilds all columns/cards via `renderCard()` and refreshes the KPI tiles (`renderKpis()`) and SVG charts (`renderCharts()`). Do not mutate card DOM directly; change `state` then call `renderBoard()`.
- Mutations go through `addTask()`, `moveTask()`, `deleteTask()`. `applyFilters()` is applied at render time only, so KPIs and charts reflect all tasks while column badges reflect filtered tasks.
- Events are delegated on `#board` (click for delete confirm, `change` for the "Move ▸" select, HTML5 drag/drop). Because the board is re-rendered by innerHTML, don't attach listeners to individual cards.
- Themes (professional/fun/futuristic) are CSS custom-property sets on `html[data-theme]`; `setTheme()` whitelists the value and keeps it in memory only. Status colours are `--st-*` tokens, redefined per theme.
- Hardening: a CSP meta tag pins the inline script and style to SHA-256 hashes. After ANY edit to the inline script or style block, recompute both hashes and update the meta tag or the page breaks. No inline `style=` attributes or inline event handlers (CSP blocks them); chart geometry uses SVG attributes. User text goes through `cleanText()`; ids are checked against `ID_PATTERN`; task count is capped (`MAX_TASKS`).
- All user-supplied strings must pass through `escapeHtml()` before going into HTML strings.
- Add-task flow (`handleSubmit`): validate → optimistic `addTask()` → close dialog/reset → toast → `notifyNewTask()` in try/catch; a failure only shows a warning toast and must never break the board.
- Task IDs are `UOB-ITPM-####` from `newId()`; seeds consume 0001–0008. Seed due dates are relative to today so some are always overdue.
