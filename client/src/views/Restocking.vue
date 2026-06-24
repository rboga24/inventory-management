<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t("restocking.title") }}</h2>
      <p>{{ t("restocking.description") }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t("common.loading") }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Configuration -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t("restocking.budgetConfiguration") }}</h3>
        </div>
        <div class="budget-controls">
          <label class="budget-label">{{
            t("restocking.allocatedBudget")
          }}</label>
          <div class="budget-display">
            {{ currencySymbol }}{{ budget.toLocaleString() }}
          </div>
          <input
            type="range"
            class="budget-slider"
            min="10000"
            max="500000"
            step="5000"
            v-model.number="budget"
            @input="onBudgetChange"
          />
          <div class="slider-labels">
            <span>{{ currencySymbol }}10K</span>
            <span>{{ currencySymbol }}500K</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Summary -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t("restocking.recommendedItems") }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t("restocking.itemsWithinBudget") }}</div>
          <div class="stat-value">{{ itemsWithinBudget.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t("restocking.estimatedCost") }}</div>
          <div class="stat-value">
            {{ currencySymbol
            }}{{
              totalCostOfBudgetItems.toLocaleString(undefined, {
                minimumFractionDigits: 2,
                maximumFractionDigits: 2,
              })
            }}
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t("restocking.remainingBudget") }}</div>
          <div class="stat-value">
            {{ currencySymbol
            }}{{
              remainingBudget.toLocaleString(undefined, {
                minimumFractionDigits: 2,
                maximumFractionDigits: 2,
              })
            }}
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t("restocking.recommendationsSummary") }}
          </h3>
        </div>
        <div v-if="recommendations.length === 0" class="no-recommendations">
          {{ t("restocking.noRecommendations") }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>#</th>
                <th>{{ t("restocking.table.sku") }}</th>
                <th>{{ t("restocking.table.itemName") }}</th>
                <th>{{ t("restocking.table.currentDemand") }}</th>
                <th>{{ t("restocking.table.forecastedDemand") }}</th>
                <th>{{ t("restocking.table.demandGap") }}</th>
                <th>{{ t("restocking.table.quantityOnHand") }}</th>
                <th>{{ t("restocking.table.unitCost") }}</th>
                <th>{{ t("restocking.table.estimatedCost") }}</th>
                <th>{{ t("restocking.table.status") }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(item, index) in recommendations"
                :key="item.item_sku"
                :class="{ 'excluded-row': !item.included_in_budget }"
              >
                <td>
                  <span v-if="item.included_in_budget" class="priority-badge">{{
                    index + 1
                  }}</span>
                  <span v-else class="priority-badge excluded">-</span>
                </td>
                <td>
                  <strong>{{ item.item_sku }}</strong>
                </td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.current_demand.toLocaleString() }}</td>
                <td>
                  <strong>{{ item.forecasted_demand.toLocaleString() }}</strong>
                </td>
                <td>{{ item.demand_gap.toLocaleString() }}</td>
                <td>{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td>
                  {{ currencySymbol
                  }}{{
                    item.unit_cost.toLocaleString(undefined, {
                      minimumFractionDigits: 2,
                      maximumFractionDigits: 2,
                    })
                  }}
                </td>
                <td>
                  {{ currencySymbol
                  }}{{
                    item.estimated_cost.toLocaleString(undefined, {
                      minimumFractionDigits: 2,
                      maximumFractionDigits: 2,
                    })
                  }}
                </td>
                <td>
                  <span v-if="item.included_in_budget" class="badge success">{{
                    t("restocking.withinBudget")
                  }}</span>
                  <span v-else class="badge danger">{{
                    t("restocking.exceedsBudget")
                  }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order -->
      <div v-if="itemsWithinBudget.length > 0" class="order-actions">
        <button
          class="place-order-btn"
          :disabled="submitting"
          @click="submitRestockingOrder"
        >
          {{ submitting ? t("common.loading") : t("restocking.placeOrder") }}
        </button>
      </div>

      <div v-if="submitSuccess" class="submit-success">
        {{ t("restocking.orderSubmitted") }}
      </div>
      <div v-if="submitError" class="submit-error">
        {{ submitError }}
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from "vue";
import { api } from "../api";
import { useI18n } from "../composables/useI18n";

export default {
  name: "Restocking",
  setup() {
    const { t, currentCurrency } = useI18n();

    // State
    const loading = ref(true);
    const error = ref(null);
    const budget = ref(100000);
    const recommendations = ref([]);
    const submitting = ref(false);
    const submitSuccess = ref(false);
    const submitError = ref(null);

    // Debounce timer for slider
    let debounceTimer = null;

    const currencySymbol = computed(() =>
      currentCurrency.value === "JPY" ? "¥" : "$",
    );

    const loadRecommendations = async () => {
      try {
        loading.value = true;
        error.value = null;
        recommendations.value = await api.getRestockingRecommendations(
          budget.value,
        );
      } catch (err) {
        error.value = t("restocking.failedToLoad") + ": " + err.message;
      } finally {
        loading.value = false;
      }
    };

    // Debounced handler for slider — 300ms delay to avoid flooding API
    const onBudgetChange = () => {
      if (debounceTimer) clearTimeout(debounceTimer);
      debounceTimer = setTimeout(() => {
        loadRecommendations();
      }, 300);
    };

    // Computed
    const itemsWithinBudget = computed(() =>
      recommendations.value.filter((r) => r.included_in_budget),
    );
    const totalCostOfBudgetItems = computed(() =>
      itemsWithinBudget.value.reduce((sum, r) => sum + r.estimated_cost, 0),
    );
    const remainingBudget = computed(
      () => budget.value - totalCostOfBudgetItems.value,
    );

    // Submit
    const submitRestockingOrder = async () => {
      if (itemsWithinBudget.value.length === 0) {
        submitError.value = t("restocking.noItemsToOrder");
        return;
      }
      try {
        submitting.value = true;
        submitError.value = null;
        submitSuccess.value = false;
        await api.submitRestockingOrder({
          items: itemsWithinBudget.value.map((item) => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.quantity_needed,
            unit_price: item.unit_cost,
          })),
          total_cost: totalCostOfBudgetItems.value,
          approved_budget: budget.value,
        });
        submitSuccess.value = true;
        setTimeout(() => {
          submitSuccess.value = false;
          loadRecommendations();
        }, 2000);
      } catch (err) {
        submitError.value = t("restocking.failedToSubmit") + ": " + err.message;
      } finally {
        submitting.value = false;
      }
    };

    onMounted(loadRecommendations);

    return {
      t,
      loading,
      error,
      budget,
      recommendations,
      submitting,
      submitSuccess,
      submitError,
      currencySymbol,
      itemsWithinBudget,
      totalCostOfBudgetItems,
      remainingBudget,
      onBudgetChange,
      submitRestockingOrder,
    };
  },
};
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1rem 0;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.5rem;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  max-width: 600px;
  height: 8px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 4px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 3px solid #ffffff;
  box-shadow: 0 2px 6px rgba(37, 99, 235, 0.3);
  transition: box-shadow 0.2s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 2px 10px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 24px;
  height: 24px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 3px solid #ffffff;
  box-shadow: 0 2px 6px rgba(37, 99, 235, 0.3);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  width: 100%;
  max-width: 600px;
  margin-top: 0.5rem;
  font-size: 0.813rem;
  color: #64748b;
}

.no-recommendations {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.excluded-row {
  opacity: 0.6;
}

.priority-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #2563eb;
  color: #ffffff;
  font-size: 0.75rem;
  font-weight: 700;
}

.priority-badge.excluded {
  background: #cbd5e1;
  color: #64748b;
}

.order-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.5rem;
}

.place-order-btn {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.3);
}

.place-order-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.submit-success {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.submit-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
