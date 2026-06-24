# Design Document

## Overview

The Expense & Budget Visualizer is a fully client-side, single-page web application built with plain HTML, CSS, and Vanilla JavaScript. It runs entirely in the browser with no backend, no external libraries, and no network requests after the initial page load. All data is stored and read from the browser's `localStorage` API.

The application provides:
- Full CRUD for expense entries
- Category management with protected defaults
- Per-category monthly budget tracking
- A dashboard with summary panels and interactive charts (pie/doughnut + bar) rendered entirely with the Canvas API
- Filtering and full-text search on the expense list
- CSV export of all expense records
- A responsive layout that spans 320 px – 2560 px

### Key Design Decisions

| Decision | Rationale |
|---|---|
| No external chart library | Requirement 7.5 forbids external network requests; the Canvas 2D API is used directly |
| Single HTML file or modular JS | Modular ES-module JS files with a single `index.html` entry point keeps code maintainable without a bundler |
| `localStorage` only | Requirement 9; no backend, no IndexedDB — keeps the architecture minimal |
| Optimistic UI with rollback | Requirement 2.2 requires atomic update with rollback on failure |
| CSS-only responsive layout | Requirement 11.4 forbids JS for layout switching |

---

## Architecture

The application follows a layered architecture with clear separation of concerns:

```
┌──────────────────────────────────────────────────────┐
│                    index.html                        │
│   (entry point – mounts app, links CSS & JS)        │
└────────────────────┬─────────────────────────────────┘
                     │ ES modules
        ┌────────────▼─────────────┐
        │       app.js (main)      │  ← initialisation, routing between views
        └──┬────────┬──────────────┘
           │        │
   ┌───────▼──┐  ┌──▼───────────────────────────┐
   │  store.js│  │         ui/                   │
   │(data layer)│ │  dashboard.js  expenses.js   │
   └───────────┘  │  categories.js budgets.js    │
                  │  charts.js     export.js      │
                  └──────────────────────────────┘
```

### Module Responsibilities

| Module | Responsibility |
|---|---|
| `store.js` | All `localStorage` reads/writes, serialisation, data validation at the persistence layer, and rollback support |
| `app.js` | Bootstrap, event delegation root, view initialisation, tab/section coordination |
| `ui/dashboard.js` | Renders Summary_Panel, wires month selector |
| `ui/expenses.js` | Renders Expense_List, Expense_Form, handles add/edit/delete |
| `ui/categories.js` | Category list UI, add/delete category flows |
| `ui/budgets.js` | Budget_Form and budget list UI |
| `ui/charts.js` | Canvas-based pie/doughnut and bar chart rendering |
| `ui/export.js` | CSV generation and file download |

### Data Flow

```
User Action
    │
    ▼
UI module (validate input)
    │ valid
    ▼
store.js (persist to localStorage)
    │ success / failure
    ▼
UI module (update DOM, charts, summary panel)
    │ failure only
    ▼
store.js (rollback snapshot)
```

---

## Components and Interfaces

### store.js — Public API

```js
// Expenses
store.getExpenses()           // → Expense[]
store.addExpense(expense)     // → { ok: boolean, error?: string }
store.updateExpense(expense)  // → { ok: boolean, error?: string, snapshot?: Expense }
store.deleteExpense(id)       // → { ok: boolean, error?: string }

// Categories
store.getCategories()                    // → Category[]
store.addCategory(name)                  // → { ok: boolean, error?: string }
store.deleteCategory(name)               // → { ok: boolean, error?: string }

// Budgets
store.getBudgets()                       // → Budget[]
store.setBudget(categoryName, month, amount) // → { ok: boolean, error?: string }
store.deleteBudget(categoryName, month)  // → { ok: boolean, error?: string }

// Utility
store.isAvailable()  // → boolean  (checks localStorage accessibility)
store.loadAll()      // → { expenses, categories, budgets }
```

### ui/charts.js — Public API

```js
charts.renderPieChart(canvasEl, data)
// data: Array<{ label: string, value: number, color: string }>

charts.renderBarChart(canvasEl, data)
// data: Array<{ day: number, total: number }>

charts.clearChart(canvasEl)
```

### ui/expenses.js — Public API

```js
expenses.init(container)          // mount component
expenses.refresh(filters?)        // re-render list with optional filter state
expenses.openEditForm(expenseId)  // populate form with existing expense
```

### Event Bus (custom events on `document`)

Components communicate via `CustomEvent` to avoid tight coupling:

| Event Name | Detail Payload | Fired By |
|---|---|---|
| `expense:saved` | `{ expense }` | expenses.js |
| `expense:deleted` | `{ id }` | expenses.js |
| `expense:updated` | `{ expense }` | expenses.js |
| `category:added` | `{ category }` | categories.js |
| `category:deleted` | `{ name }` | categories.js |
| `budget:set` | `{ budget }` | budgets.js |
| `budget:deleted` | `{ categoryName, month }` | budgets.js |
| `month:changed` | `{ month }` | dashboard.js |

---

## Data Models

All objects are serialised as JSON and stored under discrete `localStorage` keys.

### localStorage Keys

| Key | Type | Description |
|---|---|---|
| `ebv_expenses` | `Expense[]` | All expense records |
| `ebv_categories` | `Category[]` | All category records (includes defaults) |
| `ebv_budgets` | `Budget[]` | All budget records |

### Expense

```ts
interface Expense {
  id: string;           // UUID v4, generated at creation time
  amount: number;       // positive float, 0.01 – 999_999_999.99
  categoryName: string; // must match an existing Category.name
  date: string;         // ISO 8601 date string "YYYY-MM-DD"
  description: string;  // optional, max 200 chars, defaults to ""
  createdAt: string;    // ISO 8601 datetime, set once at creation
  updatedAt: string;    // ISO 8601 datetime, updated on each edit
}
```

### Category

```ts
interface Category {
  name: string;        // unique, 1–50 chars (trimmed), case-insensitive uniqueness
  isDefault: boolean;  // true for the 6 protected defaults
  createdAt: string;   // ISO 8601 datetime
}
```

Default categories on first load (if `ebv_categories` key is absent):
`Food`, `Transport`, `Entertainment`, `Health`, `Shopping`, `Others`

### Budget

```ts
interface Budget {
  id: string;          // UUID v4
  categoryName: string;
  month: string;       // "YYYY-MM" format
  amount: number;      // positive float, 0.01 – 999_999_999.99
  updatedAt: string;   // ISO 8601 datetime
}
```

Uniqueness constraint: `(categoryName, month)` pair must be unique. An upsert replaces the existing record.

### Validation Rules (shared between UI and store)

```js
const AMOUNT_MIN = 0.01;
const AMOUNT_MAX = 999_999_999.99;
const DATE_MIN   = "1900-01-01";
const DATE_MAX   = "2100-12-31";
const DESC_MAX   = 200;
const CAT_MAX    = 50;
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: Expense Add Round-Trip

*For any* valid expense record (amount in [0.01, 999,999,999.99], an existing category name, a date in [1900-01-01, 2100-12-31], and description ≤ 200 chars), adding it via `store.addExpense()` and then calling `store.getExpenses()` SHALL return a list containing a record with identical field values.

**Validates: Requirements 1.1, 2.1**

---

### Property 2: Invalid Expense Input Rejection

*For any* expense submission where at least one required field is invalid — amount is zero, negative, non-numeric, or greater than 999,999,999.99; date is absent or outside [1900-01-01, 2100-12-31]; or description exceeds 200 characters — the store SHALL remain unchanged (same number of records, same content) after the rejection.

**Validates: Requirements 1.2, 1.3, 1.4, 1.5, 2.3**

---

### Property 3: Expense Form Clears After Successful Save

*For any* valid expense that is successfully saved, all input fields of the Expense_Form SHALL be empty/reset to default values after the save operation completes.

**Validates: Requirements 1.6**

---

### Property 4: Expense Delete Removes Record

*For any* expense that exists in the store, confirming its deletion SHALL result in `store.getExpenses()` returning a list that does NOT contain a record with that expense's `id`.

**Validates: Requirements 3.2**

---

### Property 5: Category Add Round-Trip

*For any* valid category name (non-empty, non-whitespace-only, 1–50 characters trimmed, not matching an existing category name case-insensitively), adding it via `store.addCategory()` and then calling `store.getCategories()` SHALL return a list containing a category with that name.

**Validates: Requirements 4.2**

---

### Property 6: Invalid Category Name Rejection

*For any* category name submission that is empty, composed entirely of whitespace characters, or matches an existing category name (case-insensitive comparison), `store.addCategory()` SHALL return `{ ok: false }` and `store.getCategories()` SHALL return the same list as before the call (no new entry added, no existing entry modified).

**Validates: Requirements 4.3**

---

### Property 7: Category Deletion When No Linked Data

*For any* non-default category that has zero associated expenses and zero associated budgets, calling `store.deleteCategory()` SHALL succeed and `store.getCategories()` SHALL subsequently NOT include that category name.

**Validates: Requirements 4.4**

---

### Property 8: Category With Linked Data Cannot Be Deleted

*For any* category that has at least one associated expense or budget record, calling `store.deleteCategory()` SHALL return `{ ok: false }` and `store.getCategories()` SHALL still include that category.

**Validates: Requirements 4.5**

---

### Property 9: Default Categories Cannot Be Deleted

*For any* of the six default category names (Food, Transport, Entertainment, Health, Shopping, Others), calling `store.deleteCategory()` SHALL return `{ ok: false }`, and all six defaults SHALL remain in `store.getCategories()` after the call, regardless of whether any expenses or budgets are linked.

**Validates: Requirements 4.1, 4.6**

---

### Property 10: Budget Add/Update Round-Trip

*For any* valid budget (amount in [0.01, 999,999,999.99], existing category name, valid month string "YYYY-MM"), calling `store.setBudget()` and then `store.getBudgets()` SHALL return a list containing a budget record matching the provided categoryName, month, and amount.

**Validates: Requirements 5.1, 5.4**

---

### Property 11: Invalid Budget Amount Rejection

*For any* budget submission where the amount is zero, negative, non-numeric, or greater than 999,999,999.99, `store.setBudget()` SHALL return `{ ok: false }` and `store.getBudgets()` SHALL remain unchanged.

**Validates: Requirements 5.2**

---

### Property 12: Budget Uniqueness Per Category-Month

*For any* category name and month, regardless of how many times `store.setBudget()` is called with that same (categoryName, month) pair, `store.getBudgets()` SHALL contain exactly one budget record for that pair, and its amount SHALL equal the most recently submitted value.

**Validates: Requirements 5.3**

---

### Property 13: Budget Delete Removes Record

*For any* budget that exists in the store, calling `store.deleteBudget(categoryName, month)` SHALL result in `store.getBudgets()` returning a list that does NOT contain a record with that (categoryName, month) pair.

**Validates: Requirements 5.5**

---

### Property 14: Monthly Summary Computation

*For any* set of expense records belonging to a given calendar month, the total displayed for that month SHALL equal the arithmetic sum of all `expense.amount` values in that set. *For any* category with a budget set for that month, the remaining budget value SHALL equal `budget.amount − sum(expenses for that category in that month)`, which may be negative when overspent, and the displayed expense count for the month SHALL equal the number of expense records in that month.

**Validates: Requirements 6.1, 6.2, 6.4**

---

### Property 15: Over-Budget Visual Indicator

*For any* category in the Summary_Panel where the total spent exceeds the budget limit, the rendered DOM element for that category SHALL have the over-budget CSS class applied. *For any* category where total spent does not exceed the budget limit (or no budget is set), that CSS class SHALL NOT be present.

**Validates: Requirements 6.3**

---

### Property 16: Pie/Doughnut Chart Data Matches Category Totals

*For any* set of expenses in a selected month, the data passed to `charts.renderPieChart()` SHALL contain exactly one segment per category that has at least one expense in that month, and each segment's value SHALL equal the sum of expenses for that category in that month.

**Validates: Requirements 7.1**

---

### Property 17: Bar Chart Data Matches Daily Totals

*For any* set of expenses in a selected month, the data passed to `charts.renderBarChart()` SHALL contain exactly one entry per calendar day of that month, and each entry's total SHALL equal the sum of all `expense.amount` values whose `date` matches that day (zero for days with no expenses).

**Validates: Requirements 7.2**

---

### Property 18: Category Filter Returns Exact Subset

*For any* active category filter value and any set of expenses, the Expense_List SHALL display exactly those expenses whose `categoryName` matches the filter value — no more and no fewer.

**Validates: Requirements 8.1**

---

### Property 19: Date Range Filter Returns Exact Subset

*For any* valid date range [startDate, endDate] where startDate ≤ endDate and any set of expenses, the Expense_List SHALL display exactly those expenses whose `date` falls within the inclusive range [startDate, endDate].

**Validates: Requirements 8.2**

---

### Property 20: Search Filter Returns Exact Subset

*For any* search term of at least 2 characters and any set of expenses, the Expense_List SHALL display exactly those expenses whose `description` contains the search term (case-insensitive substring match) — all matching expenses included, all non-matching expenses excluded.

**Validates: Requirements 8.3**

---

### Property 21: Clear Filters Restores Date-Descending Order

*For any* set of expenses and any combination of previously active filters, clearing all filters SHALL result in the Expense_List displaying all expenses sorted by `date` descending (most recent first), with no expenses omitted.

**Validates: Requirements 8.4**

---

### Property 22: Conjunctive Multi-Filter

*For any* set of simultaneously active filters (category, date range, search term) and any expense list, the Expense_List SHALL display exactly those expenses that satisfy ALL active filter conditions simultaneously — equivalent to the intersection of each individual filter's result set.

**Validates: Requirements 8.5**

---

### Property 23: JSON Serialisation Round-Trip

*For any* valid `Expense`, `Category`, or `Budget` object, serialising it to a JSON string via `JSON.stringify` and then deserialising it via `JSON.parse` SHALL produce an object with all field values equal to those of the original object.

**Validates: Requirements 9.4**

---

### Property 24: CSV Export Faithfulness with RFC 4180 Escaping

*For any* set of expense records (including records whose `description` field contains commas, double-quote characters, or newline characters), the generated CSV SHALL contain exactly one header row followed by exactly one data row per expense record, and parsing the CSV using an RFC 4180-compliant parser SHALL recover the original field values (date, category, amount, description) exactly, with double-quotes properly escaped.

**Validates: Requirements 10.1, 10.4**

---

## Error Handling

### localStorage Failure Strategy

All `store.js` write operations are wrapped in `try/catch`. On failure:

1. The store returns `{ ok: false, error: "storage_write_failed" }`
2. For edit operations, the pre-edit snapshot (captured before the write) is re-applied: `localStorage.setItem(key, snapshot)`
3. The UI module receives the failure result and renders an inline error message in the relevant form
4. For add operations, the form data is NOT cleared, preserving user input (Requirement 1.7)

### Malformed localStorage Data

On app load, each key is read and parsed inside a `try/catch`:
- If JSON parsing fails, the key is treated as empty, a `console.warn` is emitted, and loading continues (Requirement 9.4)
- If `isAvailable()` returns `false` (localStorage access denied or unavailable), a persistent, non-dismissible error banner is rendered immediately and all write operations are suppressed for the session (Requirement 9.3)

### Validation Error Display

All validation errors are displayed as inline messages adjacent to the relevant form field, not as modal alerts. Each error message is cleared when the user modifies the associated field. Error messages are rendered via `aria-live="polite"` regions for accessibility.

### Deletion Rollback

If `localStorage.removeItem` throws during a delete operation, the expense entry remains in the UI (DOM is not mutated) and an error message is displayed (Requirement 3.4).

### Error Recovery Hierarchy

```
localStorage unavailable
    → Show persistent banner, disable all write actions

localStorage write fails
    → Roll back data, show inline error, preserve form input

localStorage read malformed
    → Treat key as empty, log console warning, continue loading

Validation error
    → Show inline error adjacent to field, block form submission
```

---

## Testing Strategy

### Overview

This feature uses a **dual testing approach**:
- **Unit / example-based tests**: Verify specific scenarios, edge cases, and error conditions
- **Property-based tests**: Verify universal invariants across all inputs (backed by a PBT library)

Property-based testing is appropriate here because the core logic — data validation, store reads/writes, aggregation computations, filter predicates, and CSV serialisation — consists of pure or near-pure functions with clear input/output contracts whose correctness should hold across arbitrary inputs.

### Property-Based Testing Library

**Target language**: JavaScript (Vanilla)
**PBT library**: [`fast-check`](https://github.com/dubzzz/fast-check) (version-pinned, e.g. `"fast-check": "3.15.0"`)

Fast-check runs in any JS environment (browser or Node), requires no framework, and supports arbitrary generators for strings, numbers, arrays, and custom objects.

**Minimum 100 iterations** per property test (fast-check default `numRuns` is 100; set explicitly via `fc.assert(fc.property(...), { numRuns: 100 })`).

Each property test MUST include a comment tag in this format:
```js
// Feature: expense-budget-visualizer, Property N: <property_text>
```

### Test Files

| File | Coverage |
|---|---|
| `tests/store.test.js` | Properties 1–15, 23 — all store-layer invariants |
| `tests/filters.test.js` | Properties 18–22 — filter and search predicates |
| `tests/charts.test.js` | Properties 16–17 — chart data computation |
| `tests/export.test.js` | Property 24 — CSV generation and RFC 4180 compliance |
| `tests/ui.test.js` | Example-based tests: edit cancel, delete cancel, empty-state messages, disabled export button, month default, error banner |

### Unit / Example-Based Tests

Unit tests cover:
- Delete confirmation prompt renders (Requirement 3.1)
- Edit cancel discards changes (Requirement 2.4)
- No-data chart message when month has no expenses (Requirement 7.4)
- App defaults to current month on load (Requirement 7.6)
- Export button disabled when store is empty (Requirement 10.3)
- CSV download filename matches `expenses-YYYY-MM-DD.csv` (Requirement 10.2)
- localStorage unavailable → persistent error banner shown (Requirement 9.3)
- Summary_Panel loads within 500 ms (Requirement 6.5)

### Property Test Configuration

```js
import fc from "fast-check";

// Reusable arbitraries
const validAmount = fc.float({ min: 0.01, max: 999_999_999.99, noNaN: true });
const validDate   = fc.date({ min: new Date("1900-01-01"), max: new Date("2100-12-31") })
                      .map(d => d.toISOString().slice(0, 10));
const validDesc   = fc.string({ maxLength: 200 });
const categoryName = fc.constantFrom("Food", "Transport", "Entertainment", "Health", "Shopping", "Others");

const validExpense = fc.record({
  amount: validAmount,
  categoryName,
  date: validDate,
  description: validDesc,
});
```

### Responsive Layout Testing

Responsive layout (Requirement 11) is verified by browser-based snapshot tests at four breakpoints: 320 px, 768 px, 1440 px, and 2560 px. These are smoke tests and are outside the scope of the JS unit test suite.

### Accessibility

- All form fields include `<label>` elements and `aria-describedby` for error messages
- Error messages are rendered in `aria-live="polite"` regions
- Chart canvases include `role="img"` and `aria-label` with a textual summary of the data
- Interactive elements meet WCAG 2.1 AA colour contrast requirements
