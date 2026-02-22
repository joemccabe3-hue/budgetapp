# CLAUDE.md — Budget App

## Project Overview

This is **My Cloud Budget** (internally versioned as Budget V54), a mobile-first personal finance web app. The entire application is a **single HTML file** (`index.html`) with no build process, no package manager, and no server-side code. All logic, styles, and markup live in one ~706-line file.

## Architecture

### Single-File Application

The only source file is `index.html`. It is structured in three sections:

| Lines | Section |
|-------|---------|
| 1–78 | `<head>`: metadata, PWA manifest (base64-embedded), CDN imports, all CSS |
| 80–269 | `<body>` HTML: all UI screens and modal elements |
| 270–703 | `<script>`: all application JavaScript |

There are no separate `.js`, `.css`, or template files. Do not create new files unless strictly required.

### External Dependencies (CDN only)

All dependencies are loaded via CDN at runtime. There is no `package.json` or local install step.

- **Chart.js** (latest v5 from jsDelivr) — doughnut and bar charts
- **Firebase App Compat** v9.22.0 — app initialization
- **Firebase Firestore Compat** v9.22.0 — cloud database with real-time sync
- **Firebase Auth Compat** v9.22.0 — Google Sign-In

### Backend: Firebase

- **Project**: `monthly-budget-8f827`
- **Auth**: Google Sign-In via popup (`firebase.auth.GoogleAuthProvider`)
- **Database**: Cloud Firestore (compat mode)
- **No server, no functions, no hosting config in this repo** — deployed externally

## Firestore Data Model

```
users/
  {uid}                          ← User config document
    startDate: string            ← "YYYY-MM-DD" (budget cycle start)
    budgetRules: Array<{
      effectiveDate: string,     ← "YYYY-MM-DD"
      amount: number             ← Weekly discretionary budget in USD
    }>
    fixedItems: Array<{
      name: string,
      amount: number,
      freq: "monthly" | "semimonthly",
      type: "income" | "expense"
    }>
    netWorth: {
      current: {
        assets: [{name, amount}],
        liabilities: [{name, amount}]
      },
      history: [{
        date: string,
        total: number,
        assets: [{name, amount}],
        liabilities: [{name, amount}]
      }]
    }

  {uid}/transactions/            ← Subcollection
    {transactionId}: {
      id: number,                ← Date.now() + Math.random() (float)
      date: string,              ← "YYYY-MM-DD"
      desc: string,
      amount: number,            ← Always positive; sign is determined by emoji category
      emoji: string              ← Category icon (e.g. "🛒", "🍔", "💰")
    }
```

**Key rules:**
- `amount` is always stored as a positive number for expenses. Income is flagged by the category (`isIncome: true` on the category object), not by a negative amount.
- Transaction `id` is a float (`Date.now() + Math.random()`). It is converted to string for Firestore document IDs.
- Firestore writes use `set(..., { merge: true })` for upserts.
- Batch writes are chunked at 450 operations per batch (Firestore limit is 500).

## Application State

All runtime state lives in two `let` variables at the top of the script block:

```javascript
let currentUser = null;
let appData = {
  transactions: [],            // Loaded from Firestore subcollection
  startDate: "2026-01-01",
  budgetRules: [...],
  fixedItems: [],
  netWorth: { current: {...}, history: [] }
};
```

Additional UI state:
- `viewDate` — the date driving the week shown in the Tracker tab
- `chartDate` — the month shown in the spending doughnut chart
- `annualChartDate` — the year shown in the annual bar chart
- `chartInst`, `annualInst`, `nwInst` — Chart.js instances (destroyed and recreated on each render)
- `pendingAction` — stores action type/payload for the confirm modal
- `selectedSnapshotIdx` — index into `netWorth.history` for snapshot modal

## Views / Tabs

The app has five views, each a `<div class="view-container">`. Only one has `class="active"` at a time.

| Tab | DOM ID | Description |
|-----|--------|-------------|
| Tracker | `view-tracker` | Weekly transaction log, surplus display, add form, charts |
| Fixed | `view-fixed` | Recurring income/expense items, weekly discretionary calc |
| Wealth | `view-networth` | Net worth snapshot tracking, assets/liabilities |
| History | `view-history` | Searchable/filterable full transaction history |
| Setup | `view-settings` | Budget start date, budget rules, CSV import, logout |

Tab switching is handled by `switchTab(v, btn)`.

## Key Functions

### Data Flow
- `loadData()` — sets up two Firestore `onSnapshot` listeners (user doc + transactions subcollection); triggers `refreshUI()` on changes
- `saveConfig()` — writes `startDate`, `budgetRules`, `fixedItems`, `netWorth` back to the user document via `set(..., { merge: true })`
- `refreshUI(animate)` — master render function; calls all tab-specific renderers

### Rendering
- `renderSpendingChart()` — monthly doughnut chart (Chart.js)
- `renderAnnualChart()` — annual category bar chart (Chart.js)
- `renderFixedTab()` — income/expense lists, weekly discretionary calculation
- `renderNetWorthTab()` — assets, liabilities, history list, sparkline
- `renderNWChart()` — net worth over time line chart (Chart.js)
- `renderFullHistory()` — filtered, grouped transaction list
- `renderSettings()` — budget rules list, current rule display
- `renderSparkline()` — tiny SVG trend line in net worth header

### CRUD
- `addTransaction()` — writes new transaction doc to Firestore subcollection
- `saveEdit()` — updates existing transaction doc
- `saveFixedItem()` — upserts into `appData.fixedItems`, then calls `saveConfig()`
- `saveNwItem()` — upserts asset or liability, then calls `saveConfig()`
- `saveNetWorthSnapshot()` — appends snapshot to `netWorth.history`, then calls `saveConfig()`
- `processCSV()` — async batch importer; parses Capital One or generic CSV format

### CSV Importer Details
The CSV importer (`processCSV`) supports two formats:
1. **Capital One format**: detected by `transaction date` + `debit` headers; skips rows with "pymt" or "payment" in description
2. **Generic format**: `date, description, amount[, category]`

Auto-categorization logic:
1. Check if `rawCat` column matches a known category name or emoji
2. Check `processedCats` — a lookup of description → emoji built from existing transactions
3. Fall back to keyword matching on 40+ hardcoded merchant names/keywords

Fixed items are auto-skipped during import by fuzzy-matching description against `fixedItems[].name`.

## Categories

Defined as a constant array at the top of the script block. Each entry:
```javascript
{ name: string, icon: string (emoji), color: string (hex), isIncome?: true }
```

Income categories have `isIncome: true`. The `Transfer` (🔄) category is excluded from spending charts. The emoji is used as the primary key — category identity is stored as `emoji` in Firestore, not `name`.

## CSS / Theming

Uses CSS custom properties defined on `:root`:
- `--primary`: `#1B263B` (dark navy)
- `--accent`: `#21F5FF` (cyan)
- `--green`: `#10b981`, `--red`: `#ef4444`
- `--bg`, `--surface`, `--text`, `--text-secondary`, `--border`, `--shadow-card`

No dark mode toggle — the theme is fixed. PWA theme color is `#1B263B`.

Mobile-specific: uses `env(safe-area-inset-bottom)` for bottom nav padding (notched devices), `viewport-fit=cover`, and `-webkit-overflow-scrolling: touch`.

## Development Workflow

### Making Changes

1. **Edit `index.html` directly** — there is no build step
2. Open the file in a browser (or serve with any static server) to test
3. Firebase credentials are hardcoded in the file — changes take effect immediately in the live Firebase project

### Testing

There is no automated test suite. Manual testing is the only option. To test:
```bash
# Any static file server works; e.g. Python
python3 -m http.server 8080
# Then open http://localhost:8080
```

### Serving Locally

The app requires Firebase (Auth and Firestore) to function. Firebase Auth with `signInWithPopup` requires a proper HTTP(S) origin — file:// will not work for Google Sign-In. Use a local server.

### Git Workflow

- Main branch: `master`
- Feature branches: `claude/<description>-<session-id>`
- Commit directly to the feature branch; do not merge to master without review

## Important Constraints

1. **Single file** — all code stays in `index.html`. Do not split into separate JS/CSS files unless the user explicitly requests a refactor.
2. **No build tools** — do not introduce webpack, Vite, TypeScript, npm, or any build pipeline unless explicitly requested.
3. **CDN dependencies only** — do not add `<script>` tags for libraries not already present without confirming with the user.
4. **Firebase credentials in source** — the API key and project config are intentionally in the client-side code (standard for Firebase browser apps). Do not move them to `.env` or suggest removing them — Firebase security is enforced via Firestore security rules in the Firebase console, not by hiding the config.
5. **Amount sign convention** — `amount` is always stored as a positive number. Sign is inferred from the emoji category (`isIncome`). Do not negate amounts when storing.
6. **ID format** — transaction IDs are floats (`Date.now() + Math.random()`). Firestore document IDs are the float `.toString()`. Be careful with type coercion when looking up by ID.

## Common Pitfalls

- **Date timezone issues**: The app appends `"T12:00:00"` when constructing `Date` objects from `YYYY-MM-DD` strings to avoid UTC midnight rollover issues. Follow this pattern for any new date handling.
- **Chart.js instances**: Always call `chartInst.destroy()` (or `annualInst`, `nwInst`) before creating a new chart on the same canvas, or Chart.js will throw.
- **Firestore batch limit**: Batch commits are capped at 450 ops. The CSV importer already handles chunking — preserve this logic if modifying the importer.
- **`sortList` and index-based operations**: `openFixedModal` passes the array index of a `fixedItems` entry. If items are reordered (e.g. by `sortList`), the index stored in `fixedEditIndex` must match the current `appData.fixedItems` array order.
- **`categoryColors` lookup**: The `renderSpendingChart` function references `categoryColors[emoji]` — this is a `window`-scope object that is implicitly expected but not explicitly defined in the visible code. If adding chart logic, build this lookup from the `categories` array using `categories.reduce((m, c) => ({...m, [c.icon]: c.color}), {})`.
