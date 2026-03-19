<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Recommend items to restock within budget</p>
    </div>

    <div v-if="loading" class="loading">Loading demand forecasts...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Section -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-section">
          <div class="budget-display">
            <div class="budget-stat">
              <span class="budget-stat-label">Budget</span>
              <span class="budget-stat-value budget-total">${{ budget.toLocaleString() }}</span>
            </div>
            <div class="budget-divider"></div>
            <div class="budget-stat">
              <span class="budget-stat-label">Allocated</span>
              <span class="budget-stat-value budget-allocated">${{ totalCost.toLocaleString() }}</span>
            </div>
            <div class="budget-divider"></div>
            <div class="budget-stat">
              <span class="budget-stat-label">Remaining</span>
              <span :class="['budget-stat-value', remaining < 0 ? 'budget-over' : 'budget-remaining']">
                ${{ remaining.toLocaleString() }}
              </span>
            </div>
          </div>
          <div class="budget-slider-row">
            <span class="slider-label">$0</span>
            <input
              type="range"
              v-model.number="budget"
              min="0"
              max="500000"
              step="1000"
              class="budget-slider"
            />
            <span class="slider-label">$500,000</span>
          </div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="successOrder" class="success-banner">
        <div class="success-content">
          <span class="success-icon">&#10003;</span>
          <div>
            <strong>Order {{ successOrder.order_number }} placed.</strong>
            Expected delivery: {{ new Date(successOrder.expected_delivery).toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' }) }}
          </div>
        </div>
        <router-link to="/orders" class="success-link">View in Orders tab</router-link>
      </div>

      <!-- Recommended Items -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="card-subtitle">Sorted by increasing trend, then by demand gap</span>
        </div>

        <div v-if="sortedForecasts.length === 0" class="empty-state">
          No demand forecast data available.
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-sku">SKU</th>
                <th class="col-trend">Trend</th>
                <th class="col-qty">Forecast Qty</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-total">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="forecast in sortedForecasts"
                :key="forecast.item_sku"
                :class="{ 'selected-row': selectedSkus.has(forecast.item_sku) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(forecast.item_sku)"
                    @change="toggleItem(forecast.item_sku)"
                    class="row-checkbox"
                  />
                </td>
                <td class="col-name">{{ forecast.item_name }}</td>
                <td class="col-sku"><strong>{{ forecast.item_sku }}</strong></td>
                <td class="col-trend">
                  <span :class="['badge', forecast.trend]">{{ forecast.trend }}</span>
                </td>
                <td class="col-qty">{{ forecast.forecasted_demand }}</td>
                <td class="col-cost">${{ forecast.unit_cost.toLocaleString() }}</td>
                <td class="col-total">
                  <strong>${{ (forecast.forecasted_demand * forecast.unit_cost).toLocaleString() }}</strong>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="footer-row">
          <div class="footer-summary">
            <span class="footer-count">{{ selectedSkus.size }} item{{ selectedSkus.size !== 1 ? 's' : '' }} selected</span>
            <span class="footer-sep">|</span>
            <span class="footer-total">
              Total: <strong>${{ totalCost.toLocaleString() }}</strong>
              <span class="footer-budget-of"> / ${{ budget.toLocaleString() }} budget</span>
            </span>
          </div>
          <button
            class="place-order-btn"
            :disabled="selectedSkus.size === 0 || budget === 0 || placingOrder"
            @click="placeOrder"
          >
            <span v-if="placingOrder">Placing Order...</span>
            <span v-else>Place Order</span>
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const forecasts = ref([])
    const loading = ref(true)
    const error = ref(null)
    const budget = ref(100000)
    const selectedSkus = ref(new Set())
    const userOverrides = ref(new Set())
    const placingOrder = ref(false)
    const successOrder = ref(null)

    const sortedForecasts = computed(() => {
      return [...forecasts.value].sort((a, b) => {
        // Increasing trend first
        const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
        const trendA = trendOrder[a.trend] ?? 1
        const trendB = trendOrder[b.trend] ?? 1
        if (trendA !== trendB) return trendA - trendB

        // Then by demand gap descending
        const gapA = a.forecasted_demand - a.current_demand
        const gapB = b.forecasted_demand - b.current_demand
        return gapB - gapA
      })
    })

    const autoSelect = () => {
      let running = 0
      const next = new Set(selectedSkus.value)

      for (const forecast of sortedForecasts.value) {
        const sku = forecast.item_sku
        // Skip SKUs that the user has manually toggled
        if (userOverrides.value.has(sku)) {
          if (next.has(sku)) {
            running += forecast.forecasted_demand * forecast.unit_cost
          }
          continue
        }

        const lineCost = forecast.forecasted_demand * forecast.unit_cost
        if (running + lineCost <= budget.value) {
          next.add(sku)
          running += lineCost
        } else {
          next.delete(sku)
        }
      }

      selectedSkus.value = next
    }

    const selectedItems = computed(() => {
      return forecasts.value.filter(f => selectedSkus.value.has(f.item_sku))
    })

    const totalCost = computed(() => {
      return selectedItems.value.reduce(
        (sum, f) => sum + f.forecasted_demand * f.unit_cost,
        0
      )
    })

    const remaining = computed(() => budget.value - totalCost.value)

    const toggleItem = (sku) => {
      userOverrides.value.add(sku)
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const placeOrder = async () => {
      if (selectedSkus.value.size === 0 || budget.value === 0) return
      placingOrder.value = true
      successOrder.value = null
      try {
        const items = selectedItems.value.map(f => ({
          sku: f.item_sku,
          name: f.item_name,
          quantity: f.forecasted_demand,
          unit_cost: f.unit_cost
        }))
        const response = await api.submitRestockOrder(items)
        successOrder.value = response
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandForecasts()
        forecasts.value = data
      } catch (err) {
        error.value = 'Failed to load demand forecasts'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Reset overrides and re-run auto-select when budget changes
    watch(budget, () => {
      userOverrides.value = new Set()
      autoSelect()
    })

    // Run auto-select once forecasts are loaded
    watch(sortedForecasts, (val) => {
      if (val.length > 0) {
        autoSelect()
      }
    })

    onMounted(loadForecasts)

    return {
      forecasts,
      loading,
      error,
      budget,
      selectedSkus,
      sortedForecasts,
      selectedItems,
      totalCost,
      remaining,
      placingOrder,
      successOrder,
      toggleItem,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget Section */
.budget-section {
  padding: 0.5rem 0;
}

.budget-display {
  display: flex;
  align-items: center;
  gap: 0;
  margin-bottom: 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem 1.5rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  flex: 1;
  text-align: center;
}

.budget-divider {
  width: 1px;
  height: 40px;
  background: #e2e8f0;
  flex-shrink: 0;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  letter-spacing: -0.025em;
}

.budget-total {
  color: #0f172a;
}

.budget-allocated {
  color: #2563eb;
}

.budget-remaining {
  color: #059669;
}

.budget-over {
  color: #dc2626;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  flex-shrink: 0;
  min-width: 60px;
}

.slider-label:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  accent-color: #2563eb;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.15);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

/* Success Banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.success-content {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  color: #065f46;
  font-size: 0.938rem;
}

.success-icon {
  font-size: 1.25rem;
  font-weight: 700;
  color: #059669;
  flex-shrink: 0;
}

.success-link {
  color: #065f46;
  font-weight: 600;
  font-size: 0.875rem;
  text-decoration: underline;
  white-space: nowrap;
  flex-shrink: 0;
}

.success-link:hover {
  color: #047857;
}

/* Card subtitle */
.card-subtitle {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 400;
}

/* Table */
.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 44px;
}

.col-name {
  width: 220px;
}

.col-sku {
  width: 130px;
}

.col-trend {
  width: 110px;
}

.col-qty {
  width: 120px;
}

.col-cost {
  width: 110px;
}

.col-total {
  width: 130px;
}

.row-checkbox {
  cursor: pointer;
  width: 16px;
  height: 16px;
  accent-color: #2563eb;
}

.selected-row {
  background-color: #eff6ff !important;
}

.selected-row:hover {
  background-color: #dbeafe !important;
}

/* Footer Row */
.footer-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 0.75rem;
  border-top: 2px solid #e2e8f0;
  margin-top: 0.5rem;
  gap: 1rem;
}

.footer-summary {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.938rem;
  color: #334155;
  flex-wrap: wrap;
}

.footer-count {
  font-weight: 600;
  color: #0f172a;
}

.footer-sep {
  color: #cbd5e1;
}

.footer-total {
  color: #334155;
}

.footer-budget-of {
  color: #64748b;
  font-size: 0.875rem;
}

/* Place Order Button */
.place-order-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  font-size: 0.938rem;
  font-weight: 600;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
  white-space: nowrap;
  flex-shrink: 0;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
  opacity: 0.7;
}

/* Empty State */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
