# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Kanban board for a fictitious bank's internal IT PMO (demo/training tool). The whole app is `index.html` — markup, `<style>` and `<script>` together. There is no build, package manager, linter or test suite.

- **Run:** `open index.html` (double-click works; no server needed).
- **Syntax-check the JS:** extract the `<script>` body and run `node --check` on it.
- **Test:** none checked in. Past verification was done by injecting a throwaway test `<script>` into a *copy* of the file (kept outside the repo) and running `Chrome --headless=new --virtual-time-budget=… --dump-dom` on it, reading results from a DOM node. Do this on a copy; don't commit test code into `index.html`.

## Hard constraints (from the original brief — don't break them)

- Vanilla HTML/CSS/JS only: no frameworks, bundler, npm, CDN, web fonts or image files. System font stack; inline SVG/Unicode for icons.
- **No persistence of any kind** — no localStorage/sessionStorage/IndexedDB/cookies. Board state is in memory and resets to seed data on refresh (the header note says so). Verify with: `grep -nE "localStorage|sessionStorage|indexedDB|document.cookie|https?://|!important" index.html` — only the FormSubmit URL should match.
- Only backend is FormSubmit's AJAX JSON endpoint (`FORMSUBMIT_ENDPOINT` in the CONFIG block at the top of the script). The email address lives there and nowhere else. A FormSubmit failure must never break the board (try/catch + warning toast).
- Don't use UOB's real logo/branding; the wordmark is a neutral "IT PMO". The `UOB-ITPM-####` ID format and `[UOB IT PMO]` email subject are deliberately kept as specified.
- CSS: tokens in `:root`, no `!important`. Visible `:focus-visible` rings; priority is never conveyed by colour alone (pills carry text).

## Architecture

State → render, one direction:

- `state = { tasks, filters, confirmingDeleteId, nextId }` is the single source of truth. Transient UI state (e.g. which card shows "Delete? Yes / No") lives in `state` too, not in the DOM.
- Mutators (`addTask`, `moveTask`, `deleteTask`) change `state` then call `renderBoard()`. `renderBoard()` runs `applyFilters()`, rebuilds every column's card list via `renderCard()`, updates the count badges, and calls `renderSummary()` (summary strip counts all tasks, not just filtered ones; column badges show filtered counts). Don't mutate card DOM directly elsewhere.
- All user-supplied strings must go through `escapeHtml()` before entering an `innerHTML` string (cards are built as HTML strings). Toasts use `textContent`.
- Events are delegated on `#board` (click for delete confirm, `change` on the per-card "Move ▸" `<select>` as the keyboard fallback, and the HTML5 `dragstart/dragover/dragleave/drop/dragend` handlers). Because cards are re-rendered wholesale, don't attach listeners to individual cards.
- Add-task flow (`handleSubmit`): `validateTask` → inline errors (no `alert()`) → optimistic `addTask` + toast → `await notifyNewTask(task)` (15 s abort timeout) → on failure a warn toast, `finally` re-enable button, reset form, close modal. The modal stays open showing "Sending…" while the request is in flight.
- Dates are compared as local `YYYY-MM-DD` strings (`todayStr()`, `offsetDate()`), not via `toISOString()`, to avoid UTC off-by-one. Seed due dates are relative to today so the Overdue badge always has examples.
- Task IDs come from `nextTaskId()` (incrementing counter, zero-padded); seeding consumes IDs 0001–0008.
