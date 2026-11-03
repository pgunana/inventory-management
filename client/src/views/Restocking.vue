<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set a budget and get recommendations based on demand forecasts. Place an order to submit it for fulfillment.</p>
    </div>

    <!-- Budget Slider -->
    <div class="card budget-card">
      <div class="budget-header">
        <h3 class="card-title">Available Budget</h3>
        <span class="budget-display">${{ budget.toLocaleString() }}</span>
      </div>
      <input
        type="range"
        class="budget-slider"
        min="0"
        max="50000"
        step="500"
        v-model.number="budget"
      />
      <div class="slider-labels">
        <span>$0</span>
        <span>$12,500</span>
        <span>$25,000</span>
        <span>$37,500</span>
        <span>$50,000</span>
      </div>
    </div>

    <!-- Budget Summary -->
    <div class="budget-summary" v-if="!loading && !error">
      <div class="summary-stat">
        <div class="stat-label">Items to Restock</div>
        <div class="stat-value">{{ recommendations.length }}</div>
      </div>
      <div class="summary-stat">
        <div class="stat-label">Total Cost</div>
        <div class="stat-value">${{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
      </div>
      <div class="summary-stat">
        <div class="stat-label">Remaining Budget</div>
        <div class="stat-value" :class="{ positive: remaining >= 0 }">
          ${{ remaining.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
        </div>
      </div>
      <div class="summary-stat">
        <div class="stat-label">Budget Used</div>
        <div class="stat-value">{{ budgetUsedPercent }}%</div>
      </div>
    </div>

    <!-- Recommendations Table -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
        <span class="card-subtitle">Sorted by forecasted demand — highest first</span>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendations.length === 0" class="empty-state">
        No items fit within this budget. Try increasing the budget.
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>SKU</th>
              <th>Item Name</th>
              <th>Trend</th>
              <th>Current Demand</th>
              <th>Forecasted Demand</th>
              <th>Qty to Order</th>
              <th>Unit Cost</th>
              <th>Total Cost</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendations" :key="item.sku">
              <td><strong>{{ item.sku }}</strong></td>
              <td>{{ item.name }}</td>
              <td>
                <span :class="['badge', item.trend]">{{ item.trend }}</span>
              </td>
              <td>{{ item.current_demand }}</td>
              <td><strong>{{ item.forecasted_demand }}</strong></td>
              <td>{{ item.quantity }}</td>
              <td>${{ item.unit_cost.toFixed(2) }}</td>
              <td>${{ item.total_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Place Order -->
    <div class="place-order-row" v-if="!loading && recommendations.length > 0">
      <button
        class="btn-place-order"
        :disabled="submitting || submitted"
        @click="placeOrder"
      >
        <span v-if="submitting">Submitting...</span>
        <span v-else-if="submitted">Order Submitted</span>
        <span v-else>Place Order — ${{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
      </button>
      <span v-if="submitted" class="success-msg">
        Order {{ lastOrderNumber }} placed. View it in the Orders tab.
      </span>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(25000)
    const loading = ref(false)
    const error = ref(null)
    const recommendations = ref([])
    const remaining = ref(0)
    const submitting = ref(false)
    const submitted = ref(false)
    const lastOrderNumber = ref('')

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const budgetUsedPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, Math.round((totalCost.value / budget.value) * 100))
    })

    let debounceTimer = null
    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data.recommendations
        remaining.value = data.remaining
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Debounce slider changes so we don't fire on every pixel drag
    watch(budget, () => {
      clearTimeout(debounceTimer)
      submitted.value = false
      debounceTimer = setTimeout(loadRecommendations, 300)
    }, { immediate: true })

    const placeOrder = async () => {
      if (submitting.value || recommendations.value.length === 0) return
      try {
        submitting.value = true
        const order = await api.createRestockingOrder(recommendations.value, budget.value)
        lastOrderNumber.value = order.order_number
        submitted.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    return {
      budget,
      loading,
      error,
      recommendations,
      remaining,
      totalCost,
      budgetUsedPercent,
      submitting,
      submitted,
      lastOrderNumber,
      placeOrder,
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1rem;
}

.budget-display {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-summary {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.summary-stat {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
}

.summary-stat .stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.4rem;
}

.summary-stat .stat-value {
  font-size: 1.35rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-stat .stat-value.positive {
  color: #059669;
}

.card-subtitle {
  font-size: 0.8rem;
  color: #94a3b8;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.place-order-row {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  margin-top: 1.5rem;
}

.btn-place-order {
  padding: 0.75rem 2rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-place-order:hover:not(:disabled) {
  background: #2563eb;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-msg {
  color: #059669;
  font-weight: 500;
  font-size: 0.9rem;
}
</style>
