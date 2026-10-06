# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A demo/training IT PMO Kanban board for a fictitious bank. The whole app is the single file `index.html`: markup, one `<style>` block and one `<script>` block.

## Hard constraints (from the original brief — keep them)

- Vanilla HTML/CSS/JS only. No frameworks, libraries, build step, bundler or npm.
- The app must stay a single `index.html` and run by double-clicking it (`file://`).
- No external resources: no CDNs, web fonts or image files. Use the system font stack and inline SVG or Unicode icons.
- **No persistence.** Don't use localStorage, sessionStorage, IndexedDB or cookies. A refresh resets the board to the seed data on purpose, and the UI says so.
- The only network call is FormSubmit's AJAX endpoint. A failure there must never break the board.
- Don't use UOB's logo or branding, or imitate an official UOB system. The visible wordmark is plain "IT PMO" text. The `UOB-ITPM-####` task IDs and the `[UOB IT PMO]` email subject are the only uses of the name, and the brief asked for both.
- Don't use `alert()`, `confirm()` or `!important`.
- Every user-supplied string goes through `escapeHtml()` before it reaches `innerHTML`.

## Running / checking

- Run: `open index.html`. No server is needed. To test real FormSubmit delivery, serve the folder over http (`python3 -m http.server`), because FormSubmit may reject requests from `file://` pages.
- There is no test suite, linter or Node on this machine. To syntax-check the script, extract the `<script>` body and parse it with JavaScriptCore:
  `osascript -l JavaScript -e 'ObjC.import("Foundation"); new Function($.NSString.stringWithContentsOfFileEncodingError("app.js",4,null).js); "OK"'`
- To test pure logic outside the page, `eval` the script with a stub `document` (`{getElementById(){return {}}}`), remove the trailing `init();`, and put the test code in the same `eval` string, since the `const`s stay scoped to that `eval`.

## Deployment

`.github/workflows/pages.yml` publishes `index.html` (only that file) to GitHub Pages on every push to `main`. A `check` job runs first on every push and pull request, and fails if `index.html` references external resources (other than FormSubmit) or uses browser storage, `alert(`, `confirm(` or `!important`. The live URL is https://arulnavneet-rgb.github.io/ClaudeDemo/ and runs are listed at https://github.com/arulnavneet-rgb/ClaudeDemo/actions. The repo's Pages source must be set to "GitHub Actions". The Pages URL is served over https, so FormSubmit works there; it may not when the file is opened from `file://`.

## Architecture (inside the `<script>`)

- **Config:** `FORMSUBMIT_ENDPOINT` is the first constant. It's the only place the notification email address is set. While it still contains `YOUR_EMAIL`, `notifyNewTask()` deliberately throws without sending anything, which shows the "email notification failed" warning toast.
- **A single `state` object is the source of truth:**
  - `tasks`: the task records
  - `filters`: the filter bar values
  - `nextId`: the ID counter
  - `ui`: transient view state, namely `moveMenuId`, `confirmDeleteId`, `draggingId`, `pendingFocus` and `sending`
- **Render from state only.** `renderBoard()` rebuilds every column's card list from `applyFilters(state.tasks, state.filters)` and updates the column counts, the header summary and the filter status. Cards are HTML strings from `renderCard()`. The inline "Move ▸" menu and the "Delete? Yes / No" confirm are drawn from `state.ui`, never toggled directly in the DOM. To change what a card shows, change state and call `renderBoard()`.
- **Focus after re-render:** cards are rebuilt on every render, so to keep keyboard focus, set `state.ui.pendingFocus` to a selector (use `cardSelector(id, inner)` or `columnHeadingSelector(status)`) before rendering. `restoreFocus()` applies it at the end of `renderBoard()`.
- **Actions:** `addTask()`, `moveTask()` and `deleteTask()` change `state.tasks`, then re-render. Board events (clicks, drag and drop) are handled by delegated listeners on `#board`, routed by each button's `data-action` attribute.
- **Add Task flow (optimistic):** `handleFormSubmit` runs `validateForm()`, which returns `{field: message}` for the inline errors. It then calls `addTask()` so the card appears at once, resets the form, and runs `notifyNewTask()` with `setSending(true/false)` around the request. Failures show the warning toast and the card stays.
- **Fixed lists:** `STATUSES`, `PRIORITIES`, `PROJECTS` and `CATEGORIES` fill both the form and filter dropdowns (`renderSelectOptions()`) and act as the validation whitelists. Column `data-status` values must match `STATUSES` exactly.
- **Seed data:** `SEED_TASKS` uses `dueInDays`, a due date relative to today, so some tasks are always overdue. Dates are local `YYYY-MM-DD` strings compared lexically (`todayISO()`), never UTC.

## Styling

All colours and spacing are CSS custom properties on `:root`. Priority is shown three ways: the card's left border colour, a text pill, and a ▲/▼ glyph. Colour must never be the only signal. The columns stack below 768px, and the Add Task sidebar stacks above the board below 1100px.
