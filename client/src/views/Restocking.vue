<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Automatically select items to restock based on demand forecasts and your available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading demand forecasts...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success Banner -->
      <div v-if="successOrder" class="success-banner">
        <div class="success-banner-content">
          <div class="success-banner-text">
            <strong>Order placed successfully.</strong>
            Order #{{ successOrder.id }} has been created. Expected delivery by {{ deliveryDate }}.
          </div>
          <button class="dismiss-btn" @click="successOrder = null">Dismiss</button>
        </div>
      </div>

      <!-- Error Banner -->
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
          <div class="budget-display">{{ formatCurrency(budget) }}</div>
        </div>
        <div class="slider-section">
          <div class="slider-labels">
            <span>$10,000</span>
            <span>$500,000</span>
          </div>
          <input
            type="range"
            class="budget-slider"
            min="10000"
            max="500000"
            step="10000"
            v-model.number="budget"
            @input="successOrder = null"
          />
          <div class="budget-usage-bar">
            <div
              class="budget-usage-fill"
              :style="{ width: Math.min((totalCost / budget) * 100, 100) + '%' }"
              :class="{ 'over-budget': totalCost > budget }"
            ></div>
          </div>
          <div class="budget-usage-labels">
            <span class="usage-spent">Allocated: {{ formatCurrency(totalCost) }}</span>
            <span class="usage-remaining" :class="{ 'remaining-danger': remainingBudget < 0 }">
              Remaining: {{ formatCurrency(remainingBudget) }}
            </span>
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="item-count">{{ recommendations.length }} item{{ recommendations.length !== 1 ? 's' : '' }}</span>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items can be recommended within this budget.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ formatUnitCost(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Footer -->
        <div class="table-footer">
          <div class="summary-row">
            <span class="summary-label">Total Spend</span>
            <span class="summary-value">{{ formatCurrency(totalCost) }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Remaining Budget</span>
            <span class="summary-value" :class="{ 'remaining-danger': remainingBudget < 0 }">
              {{ formatCurrency(remainingBudget) }}
            </span>
          </div>
        </div>
      </div>

      <!-- Place Order Button -->
      <div class="action-row">
        <button
          class="place-order-btn"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder"
        >
          <span v-if="submitting">Placing Order...</span>
          <span v-else>Place Order</span>
        </button>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(100000)
    const submitting = ref(false)
    const submitError = ref(null)
    const successOrder = ref(null)

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandForecasts()
        forecasts.value = data
      } catch (err) {
        error.value = 'Failed to load demand forecasts. Please try again.'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const recommendations = computed(() => {
      const sorted = [...forecasts.value].sort((a, b) => {
        const trendDiff = (TREND_ORDER[a.trend] ?? 3) - (TREND_ORDER[b.trend] ?? 3)
        if (trendDiff !== 0) return trendDiff
        const gapA = a.forecasted_demand - a.current_demand
        const gapB = b.forecasted_demand - b.current_demand
        return gapB - gapA
      })

      let remaining = budget.value
      const selected = []

      for (const item of sorted) {
        const totalCost = item.forecasted_demand * item.unit_cost
        if (totalCost <= remaining) {
          selected.push({ ...item, total_cost: totalCost })
          remaining -= totalCost
        }
      }

      return selected
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const deliveryDate = computed(() => {
      if (!successOrder.value) return ''
      const date = new Date()
      date.setDate(date.getDate() + 14)
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    })

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const formatUnitCost = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return

      submitting.value = true
      submitError.value = null
      successOrder.value = null

      try {
        const payload = {
          items: recommendations.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_cost: item.unit_cost,
            total_cost: item.total_cost,
            trend: item.trend
          })),
          total_cost: totalCost.value
        }
        const result = await api.createRestockingOrder(payload)
        successOrder.value = result
      } catch (err) {
        submitError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      loading,
      error,
      budget,
      recommendations,
      totalCost,
      remainingBudget,
      submitting,
      submitError,
      successOrder,
      deliveryDate,
      formatCurrency,
      formatUnitCost,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget Slider */
.slider-section {
  padding-top: 0.5rem;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 0.375rem;
}

.budget-slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  margin-bottom: 0.875rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

/* Budget Usage Bar */
.budget-usage-bar {
  width: 100%;
  height: 8px;
  background: #f1f5f9;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.budget-usage-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.3s ease, background 0.2s ease;
}

.budget-usage-fill.over-budget {
  background: #dc2626;
}

.budget-usage-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.usage-spent {
  font-weight: 500;
}

.remaining-danger {
  color: #dc2626;
  font-weight: 600;
}

/* Table Footer */
.table-footer {
  border-top: 1px solid #e2e8f0;
  padding: 0.875rem 0.75rem 0.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.875rem;
}

.summary-label {
  color: #64748b;
  font-weight: 500;
}

.summary-value {
  font-weight: 700;
  color: #0f172a;
}

/* Item Count */
.item-count {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  background: #f1f5f9;
  padding: 0.25rem 0.625rem;
  border-radius: 999px;
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 2.5rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Action Row */
.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 0.5rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 2rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success Banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
}

.success-banner-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}

.success-banner-text {
  color: #065f46;
  font-size: 0.938rem;
}

.dismiss-btn {
  background: none;
  border: 1px solid #34d399;
  color: #065f46;
  border-radius: 6px;
  padding: 0.375rem 0.875rem;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}
</style>
