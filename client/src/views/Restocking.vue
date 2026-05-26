<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
          <input
            type="range"
            class="slider"
            v-model.number="budget"
            :min="0"
            :max="totalRestockCost"
            :step="Math.max(1, Math.round(totalRestockCost / 200))"
          />
          <div class="budget-meta">
            {{ t('restocking.totalBudget') }}: {{ currencySymbol }}{{ totalRestockCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
          </div>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
          <div class="summary-bar">
            {{ t('restocking.selectedSummary', {
              count: selected.length,
              cost: currencySymbol + selectedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }),
              budget: currencySymbol + budget.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })
            }) }}
          </div>
        </div>

        <div v-if="orderPlaced" class="success-message">
          {{ t('restocking.orderPlaced') }}
        </div>

        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.restockCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="c in candidates"
                :key="c.sku"
                :class="{ 'row-unselected': !selectedSet.has(c.sku) }"
              >
                <td><strong>{{ c.sku }}</strong></td>
                <td>{{ c.name }}</td>
                <td>
                  <span :class="['badge', c.trend]">{{ c.trend }}</span>
                </td>
                <td>{{ c.quantity.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ c.unit_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td><strong>{{ currencySymbol }}{{ c.restock_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="place-order-row">
          <div v-if="selected.length === 0" class="no-items-note">
            {{ t('restocking.noItems') }}
          </div>
          <button
            class="btn-primary"
            :disabled="selected.length === 0 || submitting || orderPlaced"
            @click="placeOrder"
          >
            {{ t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(0)
    const orderPlaced = ref(false)
    const submitting = ref(false)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Map and sort forecasts by changePct descending (highest growth first)
    const candidates = computed(() => {
      return forecasts.value
        .map(f => ({
          sku: f.item_sku,
          name: f.item_name,
          trend: f.trend,
          quantity: f.forecasted_demand,
          unit_cost: f.unit_cost,
          restock_cost: f.forecasted_demand * f.unit_cost,
          changePct: f.current_demand > 0
            ? ((f.forecasted_demand - f.current_demand) / f.current_demand * 100)
            : 0
        }))
        .sort((a, b) => b.changePct - a.changePct)
    })

    const totalRestockCost = computed(() =>
      candidates.value.reduce((sum, c) => sum + c.restock_cost, 0)
    )

    // Greedy selection: include items while cumulative cost <= budget
    const selected = computed(() => {
      const result = []
      let running = 0
      for (const c of candidates.value) {
        if (running + c.restock_cost <= budget.value) {
          result.push(c)
          running += c.restock_cost
        }
      }
      return result
    })

    const selectedCost = computed(() =>
      selected.value.reduce((sum, c) => sum + c.restock_cost, 0)
    )

    // O(1) lookup for row highlighting
    const selectedSet = computed(() => new Set(selected.value.map(c => c.sku)))

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
        budget.value = totalRestockCost.value
      } catch (err) {
        error.value = 'Failed to load forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selected.value.length === 0 || submitting.value) return
      try {
        submitting.value = true
        const payload = {
          items: selected.value.map(c => ({
            sku: c.sku,
            name: c.name,
            quantity: c.quantity,
            unit_price: c.unit_cost
          })),
          total_value: selectedCost.value
        }
        await api.createRestockOrder(payload)
        orderPlaced.value = true
        setTimeout(() => {
          orderPlaced.value = false
          budget.value = totalRestockCost.value
        }, 3000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      loading,
      error,
      forecasts,
      budget,
      orderPlaced,
      submitting,
      currencySymbol,
      candidates,
      totalRestockCost,
      selected,
      selectedCost,
      selectedSet,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

/* Budget card */
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-body {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-meta {
  font-size: 0.875rem;
  color: #64748b;
}

/* Summary bar */
.summary-bar {
  font-size: 0.875rem;
  font-weight: 500;
  color: #475569;
  background: #f1f5f9;
  border-radius: 6px;
  padding: 0.5rem 1rem;
  white-space: nowrap;
}

/* Success message */
.success-message {
  margin: 0 1.5rem 1rem;
  padding: 0.875rem 1.25rem;
  background: #d1fae5;
  color: #065f46;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
}

/* Unselected row */
.row-unselected {
  opacity: 0.4;
}

/* Place order row */
.place-order-row {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1rem;
  padding: 1.25rem 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.no-items-note {
  font-size: 0.875rem;
  color: #64748b;
  font-style: italic;
}

/* Primary button */
.btn-primary {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
