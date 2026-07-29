<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget.title') }}</h3>
        </div>
        <div class="budget-row">
          <span class="budget-label">{{ t('restocking.budget.label') }}</span>
          <input
            v-model.number="budget"
            type="range"
            class="budget-slider"
            min="1000"
            max="50000"
            step="500"
          />
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
      </div>

      <div v-if="successMessage" class="success-banner">{{ successMessage }}</div>
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations.title') }}</h3>
        </div>
        <div v-if="recommendations.length === 0" class="no-items">
          {{ t('restocking.recommendations.noItems') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-select">{{ t('restocking.recommendations.table.select') }}</th>
                <th>{{ t('restocking.recommendations.table.itemName') }}</th>
                <th>{{ t('restocking.recommendations.table.sku') }}</th>
                <th>{{ t('restocking.recommendations.table.currentStock') }}</th>
                <th>{{ t('restocking.recommendations.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.recommendations.table.gap') }}</th>
                <th>{{ t('restocking.recommendations.table.trend') }}</th>
                <th>{{ t('restocking.recommendations.table.unitCost') }}</th>
                <th>{{ t('restocking.recommendations.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td class="col-select">
                  <input v-model="item.selected" type="checkbox" class="select-checkbox" />
                </td>
                <td>{{ translateProductName(item.item_name) }}</td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td><strong>{{ item.gap }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td><strong>{{ currencySymbol }}{{ item.totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('dashboard.summary.title') }}</h3>
        </div>
        <div class="stats-grid">
          <div class="stat-card info">
            <div class="stat-label">{{ t('restocking.summary.itemsSelected') }}</div>
            <div class="stat-value">{{ selectedItems.length }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">{{ t('restocking.summary.totalCost') }}</div>
            <div class="stat-value">{{ currencySymbol }}{{ totalSelectedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
          </div>
          <div class="stat-card" :class="remainingBudget < 0 ? 'danger' : 'success'">
            <div class="stat-label">{{ t('restocking.summary.remainingBudget') }}</div>
            <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
          </div>
        </div>

        <p v-if="selectedItems.length === 0" class="hint-text">{{ t('restocking.noSelection') }}</p>

        <button
          class="place-order-btn"
          :disabled="!canPlaceOrder"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.placingOrder') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const recommendations = ref([])

    const budget = ref(10000)

    const submitting = ref(false)
    const successMessage = ref(null)
    const submitError = ref(null)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        // This view is independent of the global warehouse/category/status/period
        // filters -- the budget slider drives everything here, so fetch unfiltered.
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Join demand forecasts to inventory by SKU and compute each item's
    // restock gap (forecasted demand minus what's currently on hand).
    // Items with a gap of 0 are already well-stocked -- ordering more of
    // them would just waste budget -- so they're excluded entirely rather
    // than shown with a "0 needed" row. The remaining items are sorted by
    // gap descending so the biggest shortfalls are recommended first.
    const buildRecommendations = () => {
      const inventoryBySku = new Map(inventoryItems.value.map(i => [i.sku, i]))

      const items = []
      for (const forecast of demandForecasts.value) {
        const inventoryItem = inventoryBySku.get(forecast.item_sku)
        if (!inventoryItem) continue // no matching inventory record, skip gracefully

        const gap = Math.max(forecast.forecasted_demand - inventoryItem.quantity_on_hand, 0)
        if (gap === 0) continue // already well-stocked, nothing to recommend

        items.push({
          sku: inventoryItem.sku,
          item_name: forecast.item_name,
          quantity_on_hand: inventoryItem.quantity_on_hand,
          forecasted_demand: forecast.forecasted_demand,
          gap,
          trend: forecast.trend,
          unit_cost: inventoryItem.unit_cost,
          totalCost: gap * inventoryItem.unit_cost,
          selected: false
        })
      }

      items.sort((a, b) => b.gap - a.gap)
      recommendations.value = items
    }

    // Greedy budget auto-selection: walk the gap-sorted (highest priority
    // first) list and select every item that still fits in the remaining
    // budget. This intentionally does NOT stop at the first item that
    // doesn't fit -- a cheaper, lower-priority item further down the list
    // may still fit into whatever budget is left, so every item is checked.
    const applyAutoSelection = () => {
      let runningTotal = 0
      for (const item of recommendations.value) {
        if (runningTotal + item.totalCost <= budget.value) {
          item.selected = true
          runningTotal += item.totalCost
        } else {
          item.selected = false
        }
      }
    }

    watch(budget, () => {
      applyAutoSelection()
    })

    // Selection totals are derived reactively from each row's `selected`
    // flag via computed() -- manually toggling a checkbox never re-runs the
    // greedy algorithm, it only recalculates these totals.
    const selectedItems = computed(() => recommendations.value.filter(item => item.selected))

    const totalSelectedCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.totalCost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalSelectedCost.value)

    const canPlaceOrder = computed(() =>
      !submitting.value && selectedItems.value.length > 0 && remainingBudget.value >= 0
    )

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return

      submitting.value = true
      successMessage.value = null
      submitError.value = null

      try {
        const items = selectedItems.value.map(item => ({
          sku: item.sku,
          name: item.item_name,
          quantity: item.gap,
          unit_cost: item.unit_cost
        }))
        await api.submitRestockingOrder(items)
        successMessage.value = t('restocking.orderSuccess')
        // Clear selections after a successful submission so already-ordered
        // items don't keep showing as selected in the summary.
        recommendations.value.forEach(item => { item.selected = false })
      } catch (err) {
        submitError.value = t('restocking.orderError')
      } finally {
        submitting.value = false
      }
    }

    onMounted(async () => {
      await loadData()
      buildRecommendations()
      applyAutoSelection()
    })

    return {
      t,
      loading,
      error,
      budget,
      currencySymbol,
      recommendations,
      selectedItems,
      totalSelectedCost,
      remainingBudget,
      canPlaceOrder,
      submitting,
      successMessage,
      submitError,
      placeOrder,
      translateProductName
    }
  }
}
</script>

<style scoped>
.budget-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.25);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.25);
}

.budget-slider::-moz-range-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 100px;
  text-align: right;
}

.no-items {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.col-select {
  width: 60px;
  text-align: center;
}

.select-checkbox {
  width: 16px;
  height: 16px;
  accent-color: #2563eb;
  cursor: pointer;
}

.success-banner {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}

.hint-text {
  color: #64748b;
  font-size: 0.875rem;
  margin-bottom: 1rem;
}

.place-order-btn {
  padding: 0.75rem 1.75rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
