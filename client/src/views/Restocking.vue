<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Review demand-driven restocking suggestions and allocate budget to place orders.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget Allocation</h3>
        </div>
        <div class="budget-section">
          <div class="budget-display">
            <span class="budget-label">Available Budget</span>
            <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
          </div>
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="500000"
            step="5000"
            v-model.number="budget"
            @input="applyBudgetAllocation"
          />
          <div class="budget-range-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}500,000</span>
          </div>

          <div class="budget-progress-area">
            <div class="budget-summary">
              <span class="budget-used-label">Selected Total</span>
              <span :class="['budget-used-value', selectedTotal > budget ? 'over-budget' : '']">
                {{ currencySymbol }}{{ selectedTotal.toLocaleString() }}
                <span class="budget-of"> of {{ currencySymbol }}{{ budget.toLocaleString() }}</span>
              </span>
            </div>
            <div class="progress-bar-track">
              <div
                class="progress-bar-fill"
                :class="{ 'over-budget': selectedTotal > budget }"
                :style="{ width: Math.min((selectedTotal / Math.max(budget, 1)) * 100, 100) + '%' }"
              ></div>
            </div>
            <div class="budget-remaining">
              <span>{{ selectedCount }} item{{ selectedCount !== 1 ? 's' : '' }} selected</span>
              <span :class="remainingBudget < 0 ? 'over-budget' : 'remaining-positive'">
                {{ remainingBudget < 0 ? 'Over budget by ' : 'Remaining: ' }}
                {{ currencySymbol }}{{ Math.abs(remainingBudget).toLocaleString() }}
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Success / Error messages -->
      <div v-if="submitSuccess" class="submit-success">
        Order placed successfully. Order number: <strong>{{ submittedOrderNumber }}</strong>
      </div>
      <div v-if="submitError" class="submit-error">{{ submitError }}</div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ sortedRecommendations.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="selectedCount === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-sku">SKU</th>
                <th class="col-category">Category</th>
                <th class="col-trend">Trend</th>
                <th class="col-gap">Demand Gap</th>
                <th class="col-unit-cost">Unit Cost</th>
                <th class="col-qty">Rec. Qty</th>
                <th class="col-total">Total Cost</th>
                <th class="col-lead">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedRecommendations"
                :key="item.id"
                :class="{ 'row-disabled': item.demand_gap <= 0, 'row-selected': checkedItems[item.id] }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="!!checkedItems[item.id]"
                    :disabled="item.demand_gap <= 0"
                    @change="onCheckboxChange(item, $event.target.checked)"
                  />
                </td>
                <td class="col-name">
                  <span :class="{ 'text-muted': item.demand_gap <= 0 }">{{ item.item_name }}</span>
                  <span v-if="item.demand_gap <= 0" class="no-restock-label">No restock needed</span>
                </td>
                <td class="col-sku">
                  <code class="sku-code">{{ item.item_sku }}</code>
                </td>
                <td class="col-category">{{ item.category }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-gap">
                  <span :class="item.demand_gap > 0 ? 'gap-positive' : 'gap-neutral'">
                    {{ item.demand_gap > 0 ? '+' : '' }}{{ item.demand_gap.toLocaleString() }}
                  </span>
                </td>
                <td class="col-unit-cost">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-qty">{{ item.recommended_quantity.toLocaleString() }}</td>
                <td class="col-total">
                  <strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong>
                </td>
                <td class="col-lead">{{ item.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, reactive } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(100000)
    const checkedItems = reactive({})
    const submitting = ref(false)
    const submitSuccess = ref(false)
    const submitError = ref(null)
    const submittedOrderNumber = ref(null)

    const sortedRecommendations = computed(() => {
      return [...recommendations.value].sort((a, b) => b.priority_score - a.priority_score)
    })

    const selectedItems = computed(() => {
      return sortedRecommendations.value.filter(item => checkedItems[item.id])
    })

    const selectedCount = computed(() => selectedItems.value.length)

    const selectedTotal = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const remainingBudget = computed(() => budget.value - selectedTotal.value)

    const applyBudgetAllocation = () => {
      // Clear all checked items and re-apply greedy allocation
      sortedRecommendations.value.forEach(item => {
        checkedItems[item.id] = false
      })

      let remaining = budget.value
      for (const item of sortedRecommendations.value) {
        if (item.demand_gap <= 0) continue
        if (item.total_cost <= remaining) {
          checkedItems[item.id] = true
          remaining -= item.total_cost
        }
      }
    }

    const onCheckboxChange = (item, checked) => {
      checkedItems[item.id] = checked
    }

    const placeOrder = async () => {
      if (selectedCount.value === 0) return

      submitting.value = true
      submitSuccess.value = false
      submitError.value = null
      submittedOrderNumber.value = null

      try {
        const result = await api.submitRestockingOrder({
          items: selectedItems.value,
          total_cost: selectedTotal.value,
          budget: budget.value
        })
        submittedOrderNumber.value = result.order_number
        submitSuccess.value = true

        // Uncheck all after successful order
        sortedRecommendations.value.forEach(item => {
          checkedItems[item.id] = false
        })
      } catch (err) {
        submitError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations()
        applyBudgetAllocation()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      currencySymbol,
      loading,
      error,
      sortedRecommendations,
      budget,
      checkedItems,
      selectedCount,
      selectedTotal,
      remainingBudget,
      submitting,
      submitSuccess,
      submitError,
      submittedOrderNumber,
      applyBudgetAllocation,
      onCheckboxChange,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget section */
.budget-section {
  padding: 0.5rem 0 0.25rem;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 1rem;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  appearance: none;
  -webkit-appearance: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 0 0 1px #2563eb;
  transition: background 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #1d4ed8;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 0 0 1px #2563eb;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.375rem;
  margin-bottom: 1.25rem;
}

.budget-progress-area {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem;
}

.budget-summary {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.625rem;
}

.budget-used-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.4px;
}

.budget-used-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-used-value.over-budget {
  color: #dc2626;
}

.budget-of {
  font-size: 0.875rem;
  font-weight: 400;
  color: #64748b;
}

.progress-bar-track {
  width: 100%;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.625rem;
}

.progress-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.progress-bar-fill.over-budget {
  background: #dc2626;
}

.budget-remaining {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.remaining-positive {
  color: #059669;
  font-weight: 600;
}

.over-budget {
  color: #dc2626;
  font-weight: 600;
}

/* Submit messages */
.submit-success {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.submit-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

/* Place Order button */
.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

/* Recommendations table */
.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.col-check { width: 44px; }
.col-name { width: 200px; }
.col-sku { width: 120px; }
.col-category { width: 130px; }
.col-trend { width: 100px; }
.col-gap { width: 110px; }
.col-unit-cost { width: 100px; }
.col-qty { width: 90px; }
.col-total { width: 120px; }
.col-lead { width: 100px; }

.row-disabled td {
  opacity: 0.5;
}

.row-selected {
  background: #eff6ff;
}

.row-selected:hover {
  background: #dbeafe !important;
}

.text-muted {
  color: #94a3b8;
}

.no-restock-label {
  display: block;
  font-size: 0.75rem;
  color: #94a3b8;
  font-style: italic;
  margin-top: 0.125rem;
}

.sku-code {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.8rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.gap-positive {
  color: #059669;
  font-weight: 600;
}

.gap-neutral {
  color: #94a3b8;
}
</style>
