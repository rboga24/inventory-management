# Dashboard Debug Fixes

> Generated 2026-06-24 by the `debugger` agent. Investigated runtime errors on the Dashboard page (`/`).

---

## Critical Issues

### 1. PurchaseOrderModal — missing component

**Error:** `[Vue warn]: Failed to resolve component: PurchaseOrderModal`
**File:** `client/src/views/Dashboard.vue:289` (template), `Dashboard.vue:306-313` (components registration)

**Root Cause:** `PurchaseOrderModal` is used in the template and has setup logic (`showPOModal`, `selectedBacklogForPO`, `poModalMode`, `openPOModal`, `viewPO`, `handlePOCreated`) but is never imported — and the component file doesn't exist in `client/src/components/`.

**Fix (Option A — create the component):**

- [ ] Create `client/src/components/PurchaseOrderModal.vue` with props `isOpen`, `backlogItem`, `mode` and emit `close`, `created`
- [ ] Add the import and registration in Dashboard.vue:

```diff
 import ProductDetailModal from '../components/ProductDetailModal.vue'
 import BacklogDetailModal from '../components/BacklogDetailModal.vue'
+import PurchaseOrderModal from '../components/PurchaseOrderModal.vue'

 export default {
   name: 'Dashboard',
   components: {
     ProductDetailModal,
     BacklogDetailModal,
+    PurchaseOrderModal,
   },
```

**Fix (Option B — remove the dead code):**

- [ ] Remove template lines 289-295 (`<PurchaseOrderModal ... />`)
- [ ] Remove setup refs: `showPOModal`, `selectedBacklogForPO`, `poModalMode`
- [ ] Remove setup methods: `openPOModal`, `viewPO`, `handlePOCreated`
- [ ] Remove "Create PO" / "View PO" buttons from the backlog table (lines 211-226)
- [ ] Remove from return statement

**Verification:** Open browser console at `http://localhost:3000`. The `Failed to resolve component: PurchaseOrderModal` warning should disappear.

---

### 2. Missing `/api/purchase-orders` backend endpoints

**Error:** `AxiosError: Request failed with status code 404` on any PO action
**Files:** `client/src/api.js:97-105` (client), `server/main.py` (missing routes)

**Root Cause:** `api.js` defines `createPurchaseOrder()` (POST) and `getPurchaseOrderByBacklogItem()` (GET) but `server/main.py` has no matching endpoints. The `purchase_orders` data is loaded from JSON but never exposed.

**Fix:** Add endpoints to `server/main.py`:

```python
@app.post("/api/purchase-orders")
def create_purchase_order(request: CreatePurchaseOrderRequest):
    new_po = {
        "id": f"PO-{len(purchase_orders) + 1:04d}",
        "backlog_item_id": request.backlog_item_id,
        "supplier_name": request.supplier_name,
        "quantity": request.quantity,
        "unit_cost": request.unit_cost,
        "expected_delivery_date": request.expected_delivery_date,
        "status": "Pending",
        "created_date": datetime.now().strftime("%Y-%m-%dT%H:%M:%S"),
        "notes": request.notes,
    }
    purchase_orders.append(new_po)
    return new_po


@app.get("/api/purchase-orders/{backlog_item_id}")
def get_purchase_order_by_backlog_item(backlog_item_id: str):
    po = next((po for po in purchase_orders if po["backlog_item_id"] == backlog_item_id), None)
    if not po:
        raise HTTPException(status_code=404, detail="Purchase order not found")
    return po
```

Note: The `CreatePurchaseOrderRequest` Pydantic model already exists in `main.py` (~line 115). The `PurchaseOrder` response model may need to be created.

**Verification:**

```bash
curl -s -X POST http://localhost:8001/api/purchase-orders \
  -H "Content-Type: application/json" \
  -d '{"backlog_item_id":"1","supplier_name":"Test","quantity":100,"unit_cost":10.0,"expected_delivery_date":"2025-10-01"}' \
  | python3 -m json.tool
```

---

## Medium Issues

### 3. Data model mismatch — `purchase_order_id` vs `has_purchase_order`

**Error:** "Create PO" button always shows, even if a PO exists for a backlog item
**File:** `client/src/views/Dashboard.vue:213`

**Root Cause:** Template checks `v-if="!item.purchase_order_id"` but the `/api/backlog` endpoint returns a boolean `has_purchase_order`, not a string `purchase_order_id`. Works by accident (undefined is falsy → `!undefined` is `true` → "Create PO" shows) but would break if the API ever returns `has_purchase_order: true`.

**Fix:**

```diff
  <button
-   v-if="!item.purchase_order_id"
+   v-if="!item.has_purchase_order && !item.purchase_order_id"
    @click.stop="openPOModal(item)"
    class="po-button create"
  >
```

This checks both the API-provided boolean and the runtime-assigned ID from `handlePOCreated`.

**Verification:** If `purchase_orders.json` has entries linked to backlog items, those items should show "View PO" instead of "Create PO" on page load.

---

## Low Issues

### 4. Dead `/api/tasks` code — 404 on every page load

**Error:** `AxiosError: Request failed with status code 404` (fires on every page load from `App.vue:loadTasks()`)
**Files:** `client/src/api.js:77-95` (client methods), `client/src/App.vue:93` (calls `api.getTasks()`), `server/main.py` (no task routes)

**Root Cause:** Four task API methods (`getTasks`, `createTask`, `deleteTask`, `toggleTask`) are defined in `api.js` and called from `App.vue`, but no `/api/tasks` endpoint exists in the backend. The TasksModal uses mock data from `useAuth` as a fallback, so the feature appears to work — but a 404 fires silently on every page load.

**Fix (Option A — remove dead API code):**

```diff
  // client/src/api.js — remove lines 77-95
- async getTasks() {
-   const response = await axios.get(`${API_BASE_URL}/tasks`)
-   return response.data
- },
-
- async createTask(taskData) {
-   const response = await axios.post(`${API_BASE_URL}/tasks`, taskData)
-   return response.data
- },
-
- async deleteTask(taskId) {
-   const response = await axios.delete(`${API_BASE_URL}/tasks/${taskId}`)
-   return response.data
- },
-
- async toggleTask(taskId) {
-   const response = await axios.patch(`${API_BASE_URL}/tasks/${taskId}`)
-   return response.data
- },
```

Then update `App.vue` to remove the `loadTasks` call and the `apiTasks` ref, relying solely on the mock data from `useAuth`.

**Fix (Option B — add backend endpoints):** Create `/api/tasks` CRUD endpoints in `server/main.py` with an in-memory tasks list.

**Verification:** Open browser console. The `Failed to load resource: 404` error for `/api/tasks` should disappear.

---

### 5. `useI18n()` called inside `formatDate` method

**Error:** No visible error — performance waste (new composable instance per call)
**File:** `client/src/views/Dashboard.vue:637`

**Root Cause:** `formatDate` calls `const { currentLocale } = useI18n()` inside the method body. While the underlying refs are shared (singleton pattern), this creates unnecessary wrapper objects on every call. The `currentLocale` should be destructured once in setup.

**Fix:**

```diff
  // Line ~315: add currentLocale to the existing destructure
- const { t, currentCurrency, translateProductName, translateWarehouse } = useI18n()
+ const { t, currentCurrency, currentLocale, translateProductName, translateWarehouse } = useI18n()

  // Line ~637: remove the inner useI18n call
  const formatDate = (dateString) => {
    if (!dateString) return '-'
-   const { currentLocale } = useI18n()
    const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
    const date = new Date(dateString)
    return date.toLocaleDateString(locale, { month: 'short', day: 'numeric', year: 'numeric' })
  }
```

**Verification:** The formatDate function still returns locale-aware dates. No functional change — just removes the per-call overhead.

---

## Priority Order

| Priority | Fix                                              | Impact                            | Effort    |
| -------- | ------------------------------------------------ | --------------------------------- | --------- |
| 1        | Fix #1 — PurchaseOrderModal (Option B: remove)   | Eliminates console warning        | 10 min    |
| 2        | Fix #4 — Remove dead tasks API code              | Eliminates 404 on every page load | 10 min    |
| 3        | Fix #5 — useI18n inside formatDate               | Performance cleanup               | 2 min     |
| 4        | Fix #3 — purchase_order_id vs has_purchase_order | Correctness for future data       | 2 min     |
| 5        | Fix #1 + #2 — (Option A: build PO feature)       | Full PO feature working           | 1-2 hours |

Fixes #1 (Option B), #4, #5, and #3 can all be done in under 30 minutes and will clean up the console completely.
