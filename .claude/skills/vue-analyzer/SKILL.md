---
name: vue-analyzer
description: Analyze Vue 3 component structure and suggest optimizations for performance and code reuse. Use when asked to review, optimize, or analyze Vue components.
---

# Vue Component Analyzer

Analyze Vue 3 Composition API components for performance issues, code reuse opportunities, and structural improvements. This project uses `export default { setup() }` style (not `<script setup>`).

## How to Run the Analysis

### Step 1: Gather Component Data

Read each `.vue` file in `client/src/views/` and `client/src/components/`. For each component, extract:

- **Line count** per section (template, script, style)
- **Imports** (Vue APIs, composables, api.js methods, child components)
- **Reactive state** (ref, reactive, computed, watch)
- **API calls** (which `api.*` methods are used)
- **Props and emits**
- **Composable usage** (useFilters, useI18n, useAuth)

Also read `client/src/composables/*.js` and `client/src/api.js` for cross-reference.

### Step 2: Performance Analysis

Check each component for these issues, ordered by impact:

#### High Impact

| Issue                             | How to Detect                                                                         | Fix                                                      |
| --------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Heavy logic in template           | Inline calculations, chained method calls, or complex ternaries in `{{ }}` or `:bind` | Move to computed property                                |
| Missing computed for derived data | `methods` or inline expressions that recalculate on every render                      | Convert to `computed()`                                  |
| Unbounded watchers                | `watch()` without `{ immediate }` that triggers API calls without debounce            | Add debounce (300ms for user input, 500ms for API calls) |
| Large monolithic component        | Template >150 lines or script >200 lines                                              | Extract child components or composables                  |

#### Medium Impact

| Issue                            | How to Detect                                                                 | Fix                                                          |
| -------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Duplicated data loading pattern  | Multiple components with identical `loading/error/data` + `try/catch/finally` | Extract to a `useAsyncData(fetchFn)` composable              |
| Repeated filter watching         | Multiple components with `watch([selectedPeriod, selectedLocation, ...])`     | Already handled by useFilters — verify all components use it |
| Inline event handlers with logic | `@click="item.count++; save()"` in template                                   | Move to a named method                                       |
| v-if vs v-show misuse            | `v-if` on elements that toggle frequently (tabs, dropdowns)                   | Use `v-show` for frequent toggles                            |

#### Low Impact

| Issue                          | How to Detect                                                | Fix                                          |
| ------------------------------ | ------------------------------------------------------------ | -------------------------------------------- |
| Unused imports                 | Imported but not returned from `setup()` or used in template | Remove                                       |
| v-for without proper key       | `:key="index"` instead of a unique identifier                | Use `item.id`, `item.sku`, etc.              |
| Redundant `.value` in template | `{{ count.value }}` instead of `{{ count }}`                 | Remove `.value` (auto-unwrapped in template) |

### Step 3: Code Reuse Analysis

Look for these extraction opportunities:

#### Composable Candidates

Scan for logic patterns repeated across 2+ components:

1. **Currency formatting** — If multiple components compute `currencySymbol` from `currentCurrency`, extract to useI18n or a `useCurrency()` composable
2. **Date formatting** — If multiple components define `formatDate()` with locale logic, extract to a `useFormatDate()` composable
3. **Async data loading** — The `loading/error/data` + `try/catch/finally` pattern appears in most views. A `useAsyncData(fetchFn, options)` composable would eliminate boilerplate
4. **Status class mapping** — If multiple components map status strings to CSS classes (e.g. `getOrderStatusClass`), extract to a composable

#### Component Extraction Candidates

Flag components where a template section is:

- Self-contained (own data + markup) and >30 lines
- Repeated with minor variations across views
- A distinct UI pattern (stat cards, data tables, chart containers)

Common candidates in this codebase:

- **StatCard** — the `stat-card` div pattern repeated across Dashboard, Orders, Restocking, Spending
- **DataTable** — table with thead/tbody repeated across Orders, Inventory, Demand, Restocking
- **PageHeader** — the `page-header` div with h2 + p appears in every view

### Step 4: Generate the Report

Output a structured report with these sections:

```
## Component Overview
| Component | Lines | Template | Script | Style | API Calls | Composables |
|---|---|---|---|---|---|---|

## Critical Issues (fix these)
For each issue:
- **File**: path:line_number
- **Issue**: what's wrong
- **Impact**: performance | correctness | maintainability
- **Fix**: specific code change (before → after)

## Reuse Opportunities
For each opportunity:
- **Pattern**: what's duplicated
- **Found in**: list of components
- **Extraction**: composable name or component name
- **Estimated savings**: lines removed per component

## Performance Wins
For each optimization:
- **Component**: which one
- **Current**: what it does now
- **Optimized**: what it should do
- **Why**: cache hit rate, render count reduction, bundle size

## Summary
- Total components analyzed: N
- Critical issues: N
- Reuse opportunities: N
- Estimated lines reducible: N
```

## What NOT to Flag

- The `export default { setup() }` pattern — this is the project's chosen style, not a problem
- Missing TypeScript — the project uses plain JS intentionally
- Global styles in App.vue — these are the design system, not component styles
- The i18n `t()` function in templates — this is the project's translation system
- Hardcoded data in mock_data.py — backend is intentionally in-memory

## Project-Specific Context

- **Composables**: `useFilters` (shared filter state), `useI18n` (translations + currency), `useAuth` (current user)
- **API client**: `client/src/api.js` — all API methods are centralized here
- **Styling**: Global classes (`.card`, `.stat-card`, `.badge`, `.stats-grid`, `.table-container`) are defined in `App.vue`
- **Routing**: `client/src/main.js` — flat route list, no lazy loading
- **Data flow**: Vue filters → `api.js` → FastAPI → in-memory filtering → Pydantic → computed properties
