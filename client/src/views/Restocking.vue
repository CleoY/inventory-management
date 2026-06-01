<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on your available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Section -->
      <div class="card budget-card">
        <div class="budget-label-row">
          <span class="budget-label">Available Budget</span>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>

        <div class="slider-wrapper">
          <input
            type="range"
            class="budget-slider"
            :min="0"
            :max="totalBudgetMax"
            :step="1000"
            v-model.number="budget"
          />
          <div class="slider-labels">
            <span>$0</span>
            <span>{{ formatCurrency(totalBudgetMax) }}</span>
          </div>
        </div>

        <div class="budget-summary">
          {{ recommendedItems.length }} items recommended
          &middot; {{ formatCurrency(recommendedTotal) }} total
          &middot; {{ formatCurrency(budget - recommendedTotal) }} remaining
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
        </div>
        <div class="table-container">
          <table v-if="recommendedItems.length > 0">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td>{{ item.name }}</td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.line_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
          <div v-else class="empty-state">
            No items fit within the current budget.
          </div>
        </div>
      </div>

      <!-- Place Order Button and Feedback -->
      <div class="order-actions">
        <button
          class="place-order-btn"
          :disabled="loading || submitting || recommendedItems.length === 0"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : (orderPlaced ? 'Place Another Order' : 'Place Order') }}
        </button>

        <div v-if="submitError" class="error submit-error">{{ submitError }}</div>

        <div v-if="successMessage" class="success-banner">
          {{ successMessage }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

// Trend priority for sorting: items with increasing demand are recommended first,
// then stable, then decreasing. This ensures the greedy budget selection always
// picks the highest-priority restocks and trims lower-priority ones when the
// slider is moved down.
const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitError = ref(null)
    const successMessage = ref(null)
    const orderPlaced = ref(false)

    // Raw data from parallel API calls
    const forecasts = ref([])
    const inventoryItems = ref([])

    // User-controlled budget; initialised to totalBudgetMax after data loads
    const budget = ref(0)

    // Enriched items: demand forecasts joined with inventory to attach unit_cost,
    // then sorted by trend priority so the greedy selection works correctly.
    const enrichedItems = computed(() => {
      // Build a lookup map from inventory SKU to item for O(1) joins
      const inventoryMap = new Map(
        inventoryItems.value.map(inv => [inv.sku, inv])
      )

      return forecasts.value
        .reduce((acc, forecast) => {
          const inv = inventoryMap.get(forecast.item_sku)
          // Skip forecasts that have no matching inventory entry — no price info
          if (!inv) return acc

          acc.push({
            sku: forecast.item_sku,
            name: forecast.item_name,
            forecasted_demand: forecast.forecasted_demand,
            trend: forecast.trend,
            unit_cost: inv.unit_cost,
            line_cost: forecast.forecasted_demand * inv.unit_cost
          })
          return acc
        }, [])
        .sort((a, b) => {
          const pa = TREND_PRIORITY[a.trend] ?? 99
          const pb = TREND_PRIORITY[b.trend] ?? 99
          return pa - pb
        })
    })

    // Sum of all line costs — used as slider max and initial budget value
    const totalBudgetMax = computed(() =>
      enrichedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    )

    // Greedy selection: iterate priority-sorted items, include each one only if
    // it fits cumulatively within the budget. When the slider is lowered, items
    // at the bottom of the priority list are the first to be dropped.
    const recommendedItems = computed(() => {
      let cumulative = 0
      return enrichedItems.value.filter(item => {
        if (cumulative + item.line_cost <= budget.value) {
          cumulative += item.line_cost
          return true
        }
        return false
      })
    })

    const recommendedTotal = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    )

    const formatCurrency = (value) => {
      return '$' + Math.round(value).toLocaleString()
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        // Fetch both datasets in parallel to minimise wait time
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastsData
        inventoryItems.value = inventoryData
        // Initialise slider to max after enrichedItems/totalBudgetMax are ready
        budget.value = totalBudgetMax.value
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      // When the user clicks "Place Another Order" after a successful submission,
      // reset the success state and return — the button reverts to "Place Order"
      // on the next render so the user can submit again with the same selection.
      if (orderPlaced.value) {
        orderPlaced.value = false
        successMessage.value = null
        submitError.value = null
        return
      }

      submitting.value = true
      submitError.value = null
      try {
        const orderItems = recommendedItems.value.map(i => ({
          sku: i.sku,
          name: i.name,
          quantity: i.forecasted_demand,
          unit_cost: i.unit_cost
        }))
        const response = await api.submitRestockingOrder(orderItems)
        const orderNumber = response.order_number || response.id || ''
        successMessage.value = orderNumber
          ? `Order placed successfully! Check the Orders tab to see your restocking order. (Order #${orderNumber})`
          : 'Order placed successfully! Check the Orders tab to see your restocking order.'
        orderPlaced.value = true
      } catch (err) {
        submitError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      submitting,
      submitError,
      successMessage,
      orderPlaced,
      budget,
      totalBudgetMax,
      recommendedItems,
      recommendedTotal,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-label-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-wrapper {
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.813rem;
  color: #64748b;
}

.budget-summary {
  font-size: 0.875rem;
  color: #475569;
  margin-top: 0.5rem;
}

/* Table empty state */
.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Order actions */
.order-actions {
  margin-top: 0.5rem;
  margin-bottom: 1.5rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border: none;
  border-radius: 8px;
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

.submit-error {
  margin-top: 1rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
