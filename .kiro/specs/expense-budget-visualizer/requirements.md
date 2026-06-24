# Requirements Document

## Introduction

The Expense & Budget Visualizer is a client-side web application that allows users to track personal expenses and visualize spending by category. Users can add and delete transactions, view a running total balance, and see a pie chart of spending distribution. All data is stored locally in the browser using localStorage. The app is built with plain HTML, CSS, and Vanilla JavaScript — no frameworks, no backend.

## Glossary

- **App**: The Expense & Budget Visualizer web application
- **Transaction**: A single recorded expense entry with a name, amount, and category
- **Category**: One of three fixed labels used to classify transactions: Food, Transport, or Fun
- **Transaction_Form**: The UI component used to add a new transaction
- **Transaction_List**: The UI component that displays all recorded transactions in a scrollable list
- **Total_Balance**: The running sum of all transaction amounts, displayed at the top of the page
- **Chart**: A pie chart rendered via Chart.js showing spending distribution by category
- **Local_Storage**: The browser's built-in localStorage API used for persistent client-side data

---

## Requirements

### Requirement 1: Add Transaction

**User Story:** As a user, I want to add a new transaction, so that I can record my spending.

#### Acceptance Criteria

1. THE Transaction_Form SHALL include an Item Name field (text), an Amount field (numeric), and a Category selector with exactly three options: Food, Transport, Fun.
2. WHEN a user submits the Transaction_Form with all fields filled and a valid positive numeric amount, THE App SHALL save the transaction to Local_Storage and display it in the Transaction_List.
3. IF the user submits the Transaction_Form with any field empty, THEN THE App SHALL display a validation error message and reject the submission without modifying Local_Storage.
4. IF the user submits the Transaction_Form with an amount that is not a positive number, THEN THE App SHALL display a validation error message and reject the submission.
5. WHEN a transaction is successfully saved, THE App SHALL clear the Transaction_Form fields and update the Total_Balance and Chart automatically without requiring a page reload.

---

### Requirement 2: Delete Transaction

**User Story:** As a user, I want to delete a transaction, so that I can remove incorrect entries.

#### Acceptance Criteria

1. THE Transaction_List SHALL display a delete button for each transaction entry.
2. WHEN a user clicks the delete button on a transaction, THE App SHALL remove that transaction from Local_Storage and update the Transaction_List, Total_Balance, and Chart automatically.
3. WHEN the last transaction is deleted, THE Transaction_List SHALL display an empty-state message.

---

### Requirement 3: Transaction List

**User Story:** As a user, I want to view all my transactions in a list, so that I can review my spending history.

#### Acceptance Criteria

1. THE Transaction_List SHALL display each transaction's Item Name, Amount, and Category.
2. THE Transaction_List SHALL be scrollable when the number of entries exceeds the visible area.
3. WHEN the App is loaded or reloaded, THE App SHALL read all transactions from Local_Storage and render them in the Transaction_List.

---

### Requirement 4: Total Balance

**User Story:** As a user, I want to see the total amount I have spent, so that I know my overall expense balance.

#### Acceptance Criteria

1. THE App SHALL display the Total_Balance at the top of the page.
2. WHEN a transaction is added or deleted, THE App SHALL recalculate and update the Total_Balance automatically.
3. WHEN there are no transactions, THE Total_Balance SHALL display a value of zero.

---

### Requirement 5: Spending Chart

**User Story:** As a user, I want to see a pie chart of my spending by category, so that I can understand where my money goes.

#### Acceptance Criteria

1. THE App SHALL render a pie chart using Chart.js showing the spending distribution across Food, Transport, and Fun categories.
2. WHEN a transaction is added or deleted, THE Chart SHALL update automatically to reflect the current spending distribution.
3. WHEN there are no transactions, THE Chart SHALL display a no-data state (empty chart or placeholder message).
4. THE Chart SHALL only include segments for categories that have at least one transaction.

---

### Requirement 6: Data Persistence

**User Story:** As a user, I want my transactions to persist between browser sessions, so that I do not lose my records when I close the browser.

#### Acceptance Criteria

1. THE App SHALL save all transactions to Local_Storage on every add or delete operation.
2. WHEN the App is opened or reloaded, THE App SHALL restore all transactions from Local_Storage and render them in the Transaction_List.
3. IF Local_Storage is unavailable, THEN THE App SHALL display a warning that data cannot be saved.

---

### Requirement 7: Sort Transactions (Optional)

**User Story:** As a user, I want to sort my transactions, so that I can view them in a meaningful order.

#### Acceptance Criteria

1. WHERE sorting is enabled, THE App SHALL allow users to sort the Transaction_List by Amount (ascending or descending) or by Category (alphabetical).
2. WHERE sorting is enabled, WHEN the user selects a sort option, THE Transaction_List SHALL reorder immediately without affecting stored data in Local_Storage.

---

### Requirement 8: Spending Limit Highlight (Optional)

**User Story:** As a user, I want to set a spending limit and be visually alerted when I exceed it, so that I can manage my budget.

#### Acceptance Criteria

1. WHERE spending limits are enabled, THE App SHALL allow the user to set a numeric spending limit.
2. WHERE spending limits are enabled, WHILE the Total_Balance exceeds the set limit, THE App SHALL apply visually distinct styling to indicate the limit has been exceeded.

---

### Requirement 9: Dark/Light Mode (Optional)

**User Story:** As a user, I want to toggle between dark and light mode, so that I can use the app comfortably in different lighting conditions.

#### Acceptance Criteria

1. WHERE dark/light mode is enabled, THE App SHALL provide a toggle control to switch between dark and light themes.
2. WHERE dark/light mode is enabled, WHEN the user toggles the theme, THE App SHALL apply the selected theme to all UI elements immediately.

---

### Requirement 10: Mobile-Friendly Layout

**User Story:** As a user, I want the app to work well on my phone, so that I can track expenses on the go.

#### Acceptance Criteria

1. THE App SHALL render a usable layout on viewport widths from 320px upward, with no horizontal scrollbar and all interactive elements accessible.
2. WHEN the viewport width is below 768px, THE App SHALL stack all sections into a single-column layout.
3. THE App SHALL use CSS media queries to control layout changes without requiring JavaScript for layout switching.

---

### Requirement 11: Project Structure and Tech Constraints

**User Story:** As a developer, I want a clean, constrained project structure, so that the codebase stays simple and maintainable.

#### Acceptance Criteria

1. THE App SHALL be implemented using HTML, CSS, and Vanilla JavaScript only — no frameworks or libraries except Chart.js for the chart.
2. THE App SHALL have exactly one CSS file located inside a `css/` folder.
3. THE App SHALL have exactly one JavaScript file located inside a `js/` folder.
4. THE App SHALL have an `index.html` file at the root level.
5. THE App SHALL function without a backend server.
6. THE App SHALL work correctly on Chrome, Firefox, Edge, and Safari.
