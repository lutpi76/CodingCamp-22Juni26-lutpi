# Implementation Plan: Expense & Budget Visualizer

## Overview

Implement a fully client-side, single-page web application using plain HTML, CSS, and Vanilla JavaScript. No build tools, no npm, no frameworks. All persistence goes through `localStorage`. The pie chart is rendered via Chart.js loaded from CDN. The entire application lives in exactly three files: `index.html`, `css/style.css`, and `js/app.js`.

---

## Tasks

- [x] 1. Scaffold project structure
  - Create `index.html` at the project root with boilerplate (doctype, charset, viewport meta, title)
  - Add `<link rel="stylesheet" href="css/style.css">` in `<head>`
  - Add Chart.js via CDN `<script>` tag before the closing `</body>`: `https://cdn.jsdelivr.net/npm/chart.js`
  - Add `<script src="js/app.js"></script>` after the Chart.js CDN tag
  - Create `css/style.css` as an empty file
  - Create `js/app.js` as an empty file
  - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5_

- [x] 2. Implement localStorage data layer in `js/app.js`
  - [x] 2.1 Implement `loadTransactions()` and `saveTransactions(list)`
    - `loadTransactions()`: reads `ebv_transactions` from `localStorage`; parses JSON; returns array; returns `[]` on missing key or parse error
    - `saveTransactions(list)`: serialises array to JSON and writes to `ebv_transactions`; wraps in `try/catch`; on error calls `showStorageWarning()`
    - `showStorageWarning()`: renders a visible warning banner in the UI stating data cannot be saved
    - _Requirements: 6.1, 6.2, 6.3_

  - [x] 2.2 Implement `addTransaction(name, amount, category)` and `deleteTransaction(id)`
    - `addTransaction`: creates object `{ id: Date.now(), name, amount: parseFloat(amount), category }`; pushes to in-memory list; calls `saveTransactions()`; returns the new transaction object
    - `deleteTransaction(id)`: filters the in-memory list to remove the matching id; calls `saveTransactions()`
    - Maintain a module-level `let transactions = []` array as the single source of truth
    - _Requirements: 1.2, 2.2, 6.1_

- [x] 3. Implement the HTML structure for all UI sections in `index.html`
  - Add a `<header>` section containing the Total Balance display element (e.g., `<span id="total-balance">`)
  - Add a `<section>` for the Input Form with: text input for Item Name (`id="item-name"`), number input for Amount (`id="item-amount"`), `<select>` for Category with options Food, Transport, Fun (`id="item-category"`), a submit `<button>`, and an error message `<p>` (`id="form-error"`)
  - Add a `<section>` for the Transaction List: a `<ul id="transaction-list">` and an empty-state `<p id="empty-state">` element
  - Add a `<section>` for the Chart: a `<canvas id="spending-chart">` element
  - _Requirements: 1.1, 3.1, 4.1, 5.1_

- [x] 4. Implement the Input Form with validation in `js/app.js`
  - [x] 4.1 Attach submit event listener to the form
    - Read values from `#item-name`, `#item-amount`, `#item-category`
    - Validate: all fields must be non-empty; amount must be a positive number (`parseFloat(amount) > 0`)
    - On invalid: set `#form-error` text to a descriptive message and `return` early without modifying data
    - On valid: clear `#form-error`; call `addTransaction()`; reset form fields to empty/default
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [x] 5. Implement Total Balance display in `js/app.js`
  - Write `renderBalance()`: sums `amount` of all transactions in the in-memory array; sets `#total-balance` text to the formatted sum (two decimal places)
  - When there are no transactions, display `0.00`
  - Call `renderBalance()` after every `addTransaction()` and `deleteTransaction()` call
  - _Requirements: 4.1, 4.2, 4.3_

- [x] 6. Implement Transaction List rendering in `js/app.js`
  - [x] 6.1 Write `renderList()`
    - Clears `#transaction-list` innerHTML
    - If `transactions` array is empty: show `#empty-state` message ("No transactions yet") and hide the list; otherwise hide `#empty-state` and show the list
    - For each transaction: creates an `<li>` showing name, amount (formatted to 2 decimal places), and category; appends a delete `<button data-id="...">` to each `<li>`
    - _Requirements: 2.3, 3.1, 3.2_

  - [x] 6.2 Attach delegated click listener on `#transaction-list` for delete buttons
    - On click of a delete button: read `data-id`; call `deleteTransaction(id)`; call `renderList()`, `renderBalance()`, and `renderChart()`
    - _Requirements: 2.1, 2.2_

- [x] 7. Implement Pie Chart using Chart.js in `js/app.js`
  - [x] 7.1 Write `renderChart()`
    - Compute per-category totals from the in-memory `transactions` array (only include categories with at least one transaction)
    - If no transactions exist: destroy any existing Chart instance and show a no-data placeholder message (e.g., set a `<p id="chart-placeholder">` visible); return early
    - Otherwise: hide the placeholder; build `labels` and `data` arrays from category totals; if a `Chart` instance already exists call `.destroy()` before creating a new one to avoid canvas reuse errors
    - Create `new Chart(document.getElementById('spending-chart'), { type: 'pie', data: { labels, datasets: [{ data, backgroundColor: [...] }] } })`
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

  - [x] 7.2 Add `<p id="chart-placeholder">` element to `index.html`
    - Place it adjacent to the `<canvas id="spending-chart">` inside the chart section
    - _Requirements: 5.3_

- [x] 8. Wire everything together on page load in `js/app.js`
  - At the bottom of `js/app.js`, add a `DOMContentLoaded` listener (or inline initialisation after DOM is ready) that:
    - Calls `loadTransactions()` and assigns result to the in-memory `transactions` array
    - Calls `renderBalance()`, `renderList()`, and `renderChart()` to restore the full UI from `localStorage`
  - Verify `addTransaction()` flow calls all three render functions so every mutation keeps balance, list, and chart in sync
  - Verify `deleteTransaction()` flow calls all three render functions
  - _Requirements: 3.3, 6.2, 1.5, 2.2_

- [x] 9. Checkpoint — Verify core flows in browser
  - Open `index.html` directly in a browser (no server needed)
  - Confirm: adding a valid transaction updates balance, list, and chart; form clears after submit
  - Confirm: deleting a transaction updates balance, list, and chart; empty-state message appears when last transaction is deleted
  - Confirm: reloading the page restores all transactions from localStorage
  - Confirm: chart no-data placeholder appears when transaction list is empty
  - Ask the user if any questions arise.

- [x] 10. Implement responsive CSS in `css/style.css`
  - [x] 10.1 Write base (mobile-first) styles
    - Set `box-sizing: border-box` globally; `font-family` and base `font-size`
    - Stack header, form section, transaction list section, and chart section in a single column with appropriate spacing
    - Ensure `min-width: 320px` renders without horizontal scrollbar; all inputs and buttons sized for touch targets
    - _Requirements: 10.1, 10.2_

  - [x] 10.2 Write desktop layout via `@media (min-width: 768px)`
    - Apply a two-column (or multi-column) layout using CSS Grid or Flexbox — form and list on one side, chart on the other (or any reasonable wider layout)
    - No JavaScript involved in the layout switch
    - _Requirements: 10.2, 10.3_

- [x] 11. Implement optional: Sort Transactions in `js/app.js` and `index.html`
  - [x] 11.1 Add sort controls to `index.html`
    - Add a `<select id="sort-select">` with options: Default, Amount Ascending, Amount Descending, Category A–Z
    - _Requirements: 7.1_

  - [x] 11.2 Implement sort logic in `renderList()`
    - Read `#sort-select` value; derive a sorted copy of `transactions` before rendering (do NOT mutate the in-memory array or localStorage)
    - Attach `change` listener on `#sort-select` to call `renderList()`
    - _Requirements: 7.1, 7.2_

- [x] 12. Implement optional: Spending Limit Highlight in `js/app.js` and `index.html`
  - [x] 12.1 Add spending limit input to `index.html`
    - Add a number input `<input id="spending-limit" type="number" placeholder="Set spending limit">` near the balance display
    - _Requirements: 8.1_

  - [x] 12.2 Implement over-limit highlight logic
    - In `renderBalance()`: read `#spending-limit` value; if it is a positive number and total balance exceeds it, add an `over-limit` CSS class to the balance element; otherwise remove it
    - Add `.over-limit` styles in `css/style.css` (e.g., red text or red background)
    - Attach `input` listener on `#spending-limit` to call `renderBalance()`
    - _Requirements: 8.1, 8.2_

- [x] 13. Implement optional: Dark/Light Mode Toggle in `js/app.js` and `index.html`
  - [x] 13.1 Add theme toggle button to `index.html`
    - Add `<button id="theme-toggle">Toggle Dark/Light Mode</button>` in the header
    - _Requirements: 9.1_

  - [x] 13.2 Implement theme toggle logic in `js/app.js`
    - On click: toggle a `dark-mode` class on `<body>`; persist preference to `localStorage` under key `ebv_theme`
    - On page load (before first render): read `ebv_theme` from `localStorage` and apply `dark-mode` class if saved preference is `"dark"`
    - Add `.dark-mode` CSS rules to `css/style.css` covering background, text, input, and button colours
    - _Requirements: 9.1, 9.2_

- [x] 14. Final checkpoint — Full integration verification
  - Open `index.html` in Chrome, Firefox, Edge, and Safari; verify all core flows work correctly
  - Confirm responsive layout: single-column below 768px, multi-column at or above 768px; no horizontal scroll at 320px
  - Confirm localStorage persistence across page reloads
  - Confirm Chart.js pie chart renders correctly and updates on every mutation
  - Ask the user if any questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- There is exactly one HTML file (`index.html`), one CSS file (`css/style.css`), and one JavaScript file (`js/app.js`) — do NOT split `js/app.js` into modules
- Chart.js is loaded via CDN `<script>` tag only — do NOT use npm or the Canvas 2D API directly
- No build tools, no npm, no test framework — the app opens directly as a local HTML file in a browser
- Each task references specific requirements for traceability

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "2.2"] },
    { "id": 1, "tasks": ["4.1", "6.1", "7.2"] },
    { "id": 2, "tasks": ["6.2", "7.1"] },
    { "id": 3, "tasks": ["10.1", "10.2"] },
    { "id": 4, "tasks": ["11.1", "12.1", "13.1"] },
    { "id": 5, "tasks": ["11.2", "12.2", "13.2"] }
  ]
}
```
