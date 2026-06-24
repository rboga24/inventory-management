# Vue Component Analysis — Fix List

> Generated 2026-06-24 by the `vue-analyzer` skill. 17 components analyzed across `client/src/views/` and `client/src/components/`.

---

## Critical Issues

### 1. Missing component import — Dashboard.vue

**File:** `client/src/views/Dashboard.vue`

`PurchaseOrderModal` is referenced in the template but never imported — causes a runtime error.

**Fix:** Add the import or remove the template reference if the modal isn't implemented yet.

```js
// In <script> imports section
import PurchaseOrderModal from "../components/PurchaseOrderModal.vue";
```

---

### 2. Reports.vue — Options API + direct axios

**File:** `client/src/views/Reports.vue`

The only component using Options API (`data()`, `methods`, `mounted`) and raw `axios.get('http://localhost:8001/...')` instead of `api.js`. Also has 9 `console.log` calls and uses `var` throughout.

**Fix:**

- [ ] Rewrite to Composition API (`export default { setup() }`)
- [ ] Move API calls to `client/src/api.js` (add `getQuarterlyReports()` and `getMonthlyTrends()`)
- [ ] Use `useI18n` for translations (all strings are hardcoded English)
- [ ] Use `useFilters` for filter integration
- [ ] Remove all `console.log` calls
- [ ] Replace `var` with `const`/`let`

---

### 3. useI18n() called inside methods

**Files:**

- `client/src/views/Orders.vue:204` — inside `formatDate()`
- `client/src/views/Demand.vue:194` — inside `translatePeriod()`

Creates a new composable instance on every function call.

**Fix:** Use the `currentLocale` already destructured at the top of `setup()`:

```js
// Before (Orders.vue:204)
const formatDate = (dateString) => {
  const { currentLocale } = useI18n()  // BAD — new instance per call
  const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
  ...
}

// After
const { t, currentCurrency, currentLocale, translateProductName, translateCustomerName } = useI18n()

const formatDate = (dateString) => {
  const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'  // reuses setup-level instance
  ...
}
```

---

### 4. useAuth singleton bug

**File:** `client/src/composables/useAuth.js:92`

`isAuthenticated` is declared inside `useAuth()` — each caller gets a separate ref, so logout in one component won't propagate to others.

**Fix:** Move to module scope:

```js
// Before (inside useAuth function)
export function useAuth() {
  const isAuthenticated = ref(true)  // per-caller — BAD
  ...
}

// After (module scope, like useFilters pattern)
const isAuthenticated = ref(true)  // shared singleton

export function useAuth() {
  // ...uses isAuthenticated from outer scope
}
```

---

## Performance Issues

### 5. Method-based filtering called repeatedly in templates

Methods are called multiple times in templates instead of using a single computed that groups the data once.

**Orders.vue** — `getOrdersByStatus()` called 5 times (lines 14-30):

```js
// Before: 5 separate .filter() calls on every render
<div class="stat-value">{{ getOrdersByStatus('Delivered').length }}</div>
<div class="stat-value">{{ getOrdersByStatus('Shipped').length }}</div>
// ...

// After: one computed, O(n) once
const orderCountsByStatus = computed(() => {
  const counts = { Delivered: 0, Shipped: 0, Processing: 0, Backordered: 0, Submitted: 0 }
  for (const order of orders.value) {
    if (counts[order.status] !== undefined) counts[order.status]++
  }
  return counts
})
// Template: {{ orderCountsByStatus.Delivered }}
```

**Demand.vue** — `getForecastsByTrend()` called 9 times (lines 21-64):

```js
// After: group once
const forecastsByTrend = computed(() => {
  const groups = { increasing: [], stable: [], decreasing: [] };
  for (const f of forecasts.value) {
    if (groups[f.trend]) groups[f.trend].push(f);
  }
  return groups;
});
```

**Backlog.vue** — `getBacklogByPriority()` called 3 times (lines 14-22):

```js
// Same pattern — group by priority in one computed
const backlogByPriority = computed(() => {
  const groups = { High: [], Medium: [], Low: [] };
  for (const item of backlogItems.value) {
    if (groups[item.priority]) groups[item.priority].push(item);
  }
  return groups;
});
```

---

### 6. Heavy inline math in Dashboard.vue templates

**File:** `client/src/views/Dashboard.vue` lines 41-54

```vue
<!-- Before — recalculated on every render -->
<div class="kpi-goal">{{ (fillRate - 95).toFixed(2) }}%</div>
<div
  :style="{
    width:
      Math.min((summary.total_orders_value / revenueGoal) * 100, 100) + '%',
  }"
></div>

<!-- After — move to computed -->
const fillRateDelta = computed(() => (fillRate.value - 95).toFixed(2)) const
revenueProgress = computed(() => Math.min((summary.value.total_orders_value /
revenueGoal.value * 100), 100))
```

---

### 7. Reports.vue getBarHeight is O(n²)

**File:** `client/src/views/Reports.vue` lines 259-263

Iterates all `monthlyData` to find max on every bar render.

```js
// Before — O(n) per bar = O(n²) total
getBarHeight(revenue) {
  const maxRevenue = Math.max(...this.monthlyData.map(m => m.revenue))
  return (revenue / maxRevenue * 100)
}

// After — precompute once
const maxRevenue = computed(() => Math.max(...monthlyData.value.map(m => m.revenue)))
// Template: :style="{ height: (item.revenue / maxRevenue * 100) + '%' }"
```

---

### 8. v-for with `:key="index"` — 5 occurrences

| File          | Line | Fix                                                        |
| ------------- | ---- | ---------------------------------------------------------- |
| `Orders.vue`  | 61   | `:key="item.sku + '-' + idx"` (order items lack unique ID) |
| `Orders.vue`  | 109  | `:key="item.sku + '-' + idx"`                              |
| `Reports.vue` | 28   | `:key="quarter.quarter"`                                   |
| `Reports.vue` | 51   | `:key="month.month"`                                       |
| `Reports.vue` | 82   | `:key="month.month"`                                       |

---

### 9. Unused computed properties — Dashboard.vue

Remove these — defined but never used in the template:

- `orderTrendData`
- `maxOrderCount`
- `revenueGoalDisplay`

---

### 10. No-op watcher — Spending.vue:374

```js
// Remove this entirely — empty callback does nothing
watch([selectedPeriod], () => {
  // data will automatically update via computed properties
});
```

---

## Reuse Opportunities

### R1. Add `currencySymbol` and `formatDate` to useI18n

**Duplicated in:** Orders, Restocking, InventoryDetailModal, ProductDetailModal, CostDetailModal (currencySymbol); Orders, ProfileDetailsModal, ProductDetailModal, BacklogDetailModal (formatDate)

```js
// Add to client/src/composables/useI18n.js

const currencySymbol = computed(() => currentLocale.value === 'ja' ? '¥' : '$')

function formatDate(dateString) {
  const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
  return new Date(dateString).toLocaleDateString(locale, {
    year: 'numeric', month: 'short', day: 'numeric'
  })
}

// Export both in the return object
return { t, setLocale, currentLocale, currentCurrency, currencySymbol, formatDate, ... }
```

Then remove the local `currencySymbol` computed and `formatDate` function from each component and use the shared one.

**Savings:** ~3 lines × 5 components (currencySymbol) + ~10 lines × 4 components (formatDate) = **~55 lines**

---

### R2. Extract BaseModal.vue component

**Duplicated in:** TasksModal, InventoryDetailModal, CostDetailModal, BacklogDetailModal, ProductDetailModal, ProfileDetailsModal

Each modal duplicates ~150-200 lines of identical CSS (`.modal-overlay`, `.modal-container`, `.modal-header`, `.close-button`, transitions).

```vue
<!-- client/src/components/BaseModal.vue -->
<template>
  <Teleport to="body">
    <div v-if="isOpen" class="modal-overlay" @click.self="$emit('close')">
      <div class="modal-container">
        <div class="modal-header">
          <h3 class="modal-title"><slot name="title" /></h3>
          <button class="close-button" @click="$emit('close')">x</button>
        </div>
        <div class="modal-body">
          <slot />
        </div>
        <div v-if="$slots.footer" class="modal-footer">
          <slot name="footer" />
        </div>
      </div>
    </div>
  </Teleport>
</template>

<script>
export default {
  name: "BaseModal",
  props: { isOpen: { type: Boolean, default: false } },
  emits: ["close"],
};
</script>

<style scoped>
/* All shared modal styles go here ONCE */
</style>
```

**Savings:** ~150 lines × 6 modals = **~900 lines**

---

### R3. Extract useAsyncData composable

**Duplicated in:** Dashboard, Orders, Inventory, Spending, Demand, Backlog, Restocking

Every view has the same `loading/error/data` + `try/catch/finally` boilerplate.

```js
// client/src/composables/useAsyncData.js
import { ref } from "vue";

export function useAsyncData(fetchFn) {
  const data = ref(null);
  const loading = ref(true);
  const error = ref(null);

  const load = async (...args) => {
    try {
      loading.value = true;
      error.value = null;
      data.value = await fetchFn(...args);
    } catch (err) {
      error.value = "Failed to load: " + err.message;
    } finally {
      loading.value = false;
    }
  };

  return { data, loading, error, load };
}
```

Usage:

```js
const {
  data: orders,
  loading,
  error,
  load: loadOrders,
} = useAsyncData(() => api.getOrders(getCurrentFilters()));
```

**Savings:** ~15 lines × 7 views = **~105 lines**

---

### R4. Extract useClickOutside composable

**Duplicated in:** ProfileMenu (line 96), LanguageSwitcher (line 78)

Both use a fragile `setTimeout(200)` in `handleBlur` to close dropdowns.

```js
// client/src/composables/useClickOutside.js
import { onMounted, onUnmounted } from "vue";

export function useClickOutside(elementRef, callback) {
  const handler = (event) => {
    if (elementRef.value && !elementRef.value.contains(event.target)) {
      callback();
    }
  };
  onMounted(() => document.addEventListener("click", handler));
  onUnmounted(() => document.removeEventListener("click", handler));
}
```

**Savings:** ~5 lines × 2 components + eliminates race condition

---

### R5. Extract PageHeader component

**Duplicated in:** All 8 views

```vue
<!-- client/src/components/PageHeader.vue -->
<template>
  <div class="page-header">
    <h2>{{ title }}</h2>
    <p v-if="description">{{ description }}</p>
  </div>
</template>

<script>
export default {
  name: "PageHeader",
  props: {
    title: { type: String, required: true },
    description: { type: String, default: "" },
  },
};
</script>
```

**Savings:** ~5 lines × 8 views = **~40 lines**

---

### R6. Extract StatCard component

**Duplicated in:** Dashboard, Orders, Restocking, Spending

```vue
<!-- client/src/components/StatCard.vue -->
<template>
  <div :class="['stat-card', variant]">
    <div class="stat-label">{{ label }}</div>
    <div class="stat-value">
      <slot>{{ value }}</slot>
    </div>
  </div>
</template>

<script>
export default {
  name: "StatCard",
  props: {
    label: { type: String, required: true },
    value: { type: [String, Number], default: "" },
    variant: { type: String, default: "" }, // 'success', 'warning', 'danger', 'info'
  },
};
</script>
```

**Savings:** ~5 lines × 4 views = **~20 lines**

---

## Priority Order

| Priority | Fix                                                  | Impact            | Effort      |
| -------- | ---------------------------------------------------- | ----------------- | ----------- |
| 1        | Fix #1 — missing PurchaseOrderModal import           | Runtime error     | 1 line      |
| 2        | Fix #3 — useI18n inside methods                      | Performance bug   | 5 min each  |
| 3        | Fix #4 — useAuth singleton                           | Correctness bug   | 5 min       |
| 4        | Fix #5 — method-to-computed in Orders/Demand/Backlog | Performance       | 15 min each |
| 5        | R1 — currencySymbol + formatDate in useI18n          | Consistency + DRY | 30 min      |
| 6        | Fix #10 + #9 — remove dead code                      | Cleanliness       | 5 min       |
| 7        | R2 — BaseModal extraction                            | ~900 lines saved  | 1-2 hours   |
| 8        | R3 — useAsyncData composable                         | ~105 lines saved  | 30 min      |
| 9        | Fix #2 — Reports.vue rewrite                         | Consistency       | 1-2 hours   |
| 10       | R4, R5, R6 — small extractions                       | Polish            | 30 min each |

---

## Summary

| Metric                    | Count  |
| ------------------------- | ------ |
| Components analyzed       | 17     |
| Critical issues           | 4      |
| Performance issues        | 7      |
| Reuse opportunities       | 6      |
| Estimated lines reducible | ~1,120 |
