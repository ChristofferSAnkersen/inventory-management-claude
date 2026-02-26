<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Recommend items to restock based on demand forecast gaps within your available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Control -->
      <div class="card budget-card">
        <div class="budget-header">
          <div>
            <div class="card-title">Available Budget</div>
            <div class="budget-sub">Drag to set your restocking budget</div>
          </div>
          <div class="budget-display">
            <span class="budget-amount">${{ budget.toLocaleString() }}</span>
          </div>
        </div>
        <div class="slider-wrapper">
          <span class="slider-label">$1,000</span>
          <input
            type="range"
            min="1000"
            max="100000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="slider-label">$100,000</span>
        </div>
        <div class="budget-bar-track">
          <div
            class="budget-bar-fill"
            :class="{ 'over-budget': totalSelected > budget }"
            :style="{ width: Math.min(100, (totalSelected / budget) * 100) + '%' }"
          ></div>
        </div>
        <div class="budget-usage">
          <span :class="{ 'over-budget-text': totalSelected > budget }">
            ${{ totalSelected.toLocaleString(undefined, { maximumFractionDigits: 0 }) }} used
          </span>
          <span class="budget-remaining">
            ${{ Math.max(0, budget - totalSelected).toLocaleString(undefined, { maximumFractionDigits: 0 }) }} remaining
          </span>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommendations
            <span class="rec-count">{{ autoSelected.length }} of {{ recommendations.length }} items fit budget</span>
          </h3>
          <div class="header-actions">
            <button class="btn-secondary" @click="deselectAll">Clear all</button>
            <button class="btn-secondary" @click="selectAll">Select all</button>
          </div>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>Item</th>
                <th>Category</th>
                <th class="col-num">Current Demand</th>
                <th class="col-num">Forecasted</th>
                <th class="col-num">Gap</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Qty to Order</th>
                <th class="col-num">Est. Cost</th>
                <th class="col-num">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-selected': selected[item.item_sku], 'row-disabled': !selected[item.item_sku] && !canAfford(item) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selected[item.item_sku]"
                    @change="toggleItem(item)"
                    class="row-checkbox"
                  />
                </td>
                <td>
                  <div class="item-name">{{ item.item_name }}</div>
                  <div class="item-sku">{{ item.item_sku }}</div>
                </td>
                <td><span class="category-chip">{{ item.category }}</span></td>
                <td class="col-num">{{ item.current_demand.toLocaleString() }}</td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num">
                  <span class="gap-badge">+{{ item.demand_gap.toLocaleString() }}</span>
                </td>
                <td class="col-num">${{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num">{{ item.recommended_quantity.toLocaleString() }}</td>
                <td class="col-num"><strong>${{ item.estimated_cost.toLocaleString() }}</strong></td>
                <td class="col-num">
                  <span class="lead-badge">{{ item.lead_time_days }}d</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Order Summary & Submit -->
      <div class="card order-footer">
        <div class="footer-summary">
          <div class="summary-item">
            <span class="summary-label">Selected items</span>
            <span class="summary-value">{{ selectedItems.length }}</span>
          </div>
          <div class="summary-item">
            <span class="summary-label">Total units</span>
            <span class="summary-value">{{ totalUnits.toLocaleString() }}</span>
          </div>
          <div class="summary-item">
            <span class="summary-label">Total cost</span>
            <span class="summary-value" :class="{ 'over-budget-text': totalSelected > budget }">
              ${{ totalSelected.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}
            </span>
          </div>
          <div class="summary-item">
            <span class="summary-label">Max lead time</span>
            <span class="summary-value">{{ maxLeadTime }} days</span>
          </div>
        </div>
        <button
          class="btn-place-order"
          :disabled="selectedItems.length === 0 || placing || totalSelected > budget"
          @click="placeOrder"
        >
          <span v-if="placing">Placing order...</span>
          <span v-else>Place Order</span>
        </button>
      </div>

      <!-- Success Banner -->
      <div v-if="lastOrder" class="success-banner">
        <div class="success-icon">&#10003;</div>
        <div class="success-text">
          <strong>Order {{ lastOrder.order_number }} submitted</strong>
          <span>Estimated delivery in {{ lastOrder.lead_time_days }} days — visible in the Orders tab under Submitted Orders.</span>
        </div>
        <button class="dismiss-btn" @click="lastOrder = null">&#x2715;</button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(25000)
    const selected = ref({})
    const placing = ref(false)
    const lastOrder = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        const data = await api.getRestockingRecommendations()
        recommendations.value = data
        // Auto-select items that fit in default budget, greedy
        applyAutoSelect()
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const applyAutoSelect = () => {
      const newSelected = {}
      let spent = 0
      for (const item of recommendations.value) {
        if (spent + item.estimated_cost <= budget.value) {
          newSelected[item.item_sku] = true
          spent += item.estimated_cost
        }
      }
      selected.value = newSelected
    }

    const selectedItems = computed(() =>
      recommendations.value.filter(r => selected.value[r.item_sku])
    )

    const autoSelected = computed(() => {
      const result = []
      let spent = 0
      for (const item of recommendations.value) {
        if (spent + item.estimated_cost <= budget.value) {
          result.push(item)
          spent += item.estimated_cost
        }
      }
      return result
    })

    const totalSelected = computed(() =>
      selectedItems.value.reduce((sum, i) => sum + i.estimated_cost, 0)
    )

    const totalUnits = computed(() =>
      selectedItems.value.reduce((sum, i) => sum + i.recommended_quantity, 0)
    )

    const maxLeadTime = computed(() => {
      if (!selectedItems.value.length) return 0
      return Math.max(...selectedItems.value.map(i => i.lead_time_days))
    })

    const canAfford = (item) => {
      return totalSelected.value + item.estimated_cost <= budget.value
    }

    const toggleItem = (item) => {
      const next = { ...selected.value }
      if (next[item.item_sku]) {
        delete next[item.item_sku]
      } else {
        next[item.item_sku] = true
      }
      selected.value = next
    }

    const selectAll = () => {
      const next = {}
      recommendations.value.forEach(r => { next[r.item_sku] = true })
      selected.value = next
    }

    const deselectAll = () => {
      selected.value = {}
    }

    const placeOrder = async () => {
      if (!selectedItems.value.length) return
      placing.value = true
      try {
        const payload = {
          items: selectedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.recommended_quantity,
            unit_cost: i.unit_cost,
            category: i.category,
          })),
          total_cost: Math.round(totalSelected.value * 100) / 100,
        }
        const order = await api.createRestockingOrder(payload)
        lastOrder.value = order
        selected.value = {}
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      recommendations,
      budget,
      selected,
      placing,
      lastOrder,
      selectedItems,
      autoSelected,
      totalSelected,
      totalUnits,
      maxLeadTime,
      canAfford,
      toggleItem,
      selectAll,
      deselectAll,
      placeOrder,
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.budget-sub {
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.slider-wrapper {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.slider-label {
  font-size: 0.75rem;
  color: #64748b;
  white-space: nowrap;
  min-width: 56px;
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
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0,0,0,0.2);
}

.budget-bar-track {
  height: 4px;
  background: #f1f5f9;
  border-radius: 2px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 2px;
  transition: width 0.2s ease;
}

.budget-bar-fill.over-budget {
  background: #dc2626;
}

.budget-usage {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #475569;
}

.budget-remaining {
  color: #059669;
  font-weight: 500;
}

.over-budget-text {
  color: #dc2626 !important;
  font-weight: 600;
}

/* Table enhancements */
.col-check {
  width: 40px;
  text-align: center;
}

.col-num {
  text-align: right;
}

.row-checkbox {
  cursor: pointer;
  width: 15px;
  height: 15px;
  accent-color: #2563eb;
}

.row-selected {
  background: #eff6ff;
}

.row-disabled {
  opacity: 0.45;
}

.item-name {
  font-weight: 500;
  color: #0f172a;
  font-size: 0.875rem;
}

.item-sku {
  font-size: 0.75rem;
  color: #64748b;
  font-family: monospace;
  margin-top: 2px;
}

.category-chip {
  font-size: 0.72rem;
  padding: 2px 8px;
  background: #f1f5f9;
  color: #475569;
  border-radius: 4px;
  border: 1px solid #e2e8f0;
  white-space: nowrap;
}

.gap-badge {
  font-size: 0.8rem;
  font-weight: 600;
  color: #059669;
}

.lead-badge {
  font-size: 0.75rem;
  padding: 2px 8px;
  background: #e0e7ff;
  color: #3730a3;
  border-radius: 4px;
}

/* Header actions */
.header-actions {
  display: flex;
  gap: 0.5rem;
}

.btn-secondary {
  padding: 0.375rem 0.875rem;
  font-size: 0.813rem;
  font-weight: 500;
  border: 1px solid #e2e8f0;
  background: #fff;
  color: #475569;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.15s;
}

.btn-secondary:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}

.rec-count {
  font-size: 0.813rem;
  font-weight: 400;
  color: #64748b;
  margin-left: 0.5rem;
}

/* Footer */
.order-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1.5rem;
}

.footer-summary {
  display: flex;
  gap: 2rem;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: #64748b;
}

.summary-value {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.btn-place-order {
  padding: 0.75rem 2rem;
  font-size: 0.938rem;
  font-weight: 600;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;
  white-space: nowrap;
  flex-shrink: 0;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 10px;
  padding: 1rem 1.25rem;
  margin-top: 0;
}

.success-icon {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: #059669;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  flex-shrink: 0;
}

.success-text {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.success-text strong {
  font-size: 0.938rem;
  color: #065f46;
}

.success-text span {
  font-size: 0.813rem;
  color: #047857;
}

.dismiss-btn {
  background: none;
  border: none;
  font-size: 1rem;
  color: #6b7280;
  cursor: pointer;
  padding: 0.25rem;
  line-height: 1;
}

.dismiss-btn:hover {
  color: #374151;
}
</style>
