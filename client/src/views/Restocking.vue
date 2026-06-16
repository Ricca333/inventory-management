<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget slider -->
      <div class="card budget-card">
        <div class="budget-head">
          <div>
            <div class="card-title">{{ t('restocking.budgetTitle') }}</div>
            <p class="budget-hint">{{ t('restocking.budgetHint') }}</p>
          </div>
          <div class="budget-value">{{ formatCurrency(budget, currency) }}</div>
        </div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="maxBudget"
          :step="step"
          v-model.number="budget"
        />
        <div class="budget-scale">
          <span>{{ formatCurrency(0, currency) }}</span>
          <span>{{ formatCurrency(maxBudget, currency) }}</span>
        </div>
      </div>

      <!-- Summary cards -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.itemsRecommended') }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.totalUnits') }}</div>
          <div class="stat-value">{{ totalUnits.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ formatCurrency(totalCost, currency) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
          <div class="stat-value">{{ formatCurrency(budgetRemaining, currency) }}</div>
        </div>
      </div>

      <!-- Recommendations -->
      <div class="card">
        <div class="card-header recommendations-header">
          <div>
            <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
            <p class="recommendations-hint">{{ t('restocking.recommendationsHint') }}</p>
          </div>
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="placedMessage" class="success-banner">{{ placedMessage }}</div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th class="num">{{ t('restocking.table.currentDemand') }}</th>
                <th class="num">{{ t('restocking.table.forecastedDemand') }}</th>
                <th class="num">{{ t('restocking.table.recommendedQty') }}</th>
                <th class="num">{{ t('restocking.table.unitCost') }}</th>
                <th class="num">{{ t('restocking.table.lineCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rec in recommendations" :key="rec.item_sku">
                <td><strong>{{ rec.item_sku }}</strong></td>
                <td>{{ translateProductName(rec.item_name) }}</td>
                <td class="num">{{ rec.current_demand.toLocaleString() }}</td>
                <td class="num">{{ rec.forecasted_demand.toLocaleString() }}</td>
                <td class="num"><strong>{{ rec.quantity.toLocaleString() }}</strong></td>
                <td class="num">{{ formatCurrency(rec.unit_cost, currency) }}</td>
                <td class="num"><strong>{{ formatCurrency(rec.lineCost, currency) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

const LEAD_TIME_DAYS = 14

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()
    const currency = computed(() => currentCurrency.value)

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(0)
    const placing = ref(false)
    const placedMessage = ref('')

    // Candidate items: demand is growing (forecasted > current). Each carries the
    // demand gap (recommended order quantity) and its full line cost.
    const candidates = computed(() => {
      return forecasts.value
        .map(f => {
          const gap = f.forecasted_demand - f.current_demand
          return { ...f, quantity: gap, lineCost: gap * f.unit_cost }
        })
        .filter(c => c.quantity > 0)
        .sort((a, b) => b.quantity - a.quantity)
    })

    // Total cost to restock every candidate — drives the slider's upper bound.
    const maxBudget = computed(() => {
      const total = candidates.value.reduce((sum, c) => sum + c.lineCost, 0)
      return Math.max(1000, Math.ceil(total / 1000) * 1000)
    })

    const step = computed(() => Math.max(100, Math.round(maxBudget.value / 100 / 100) * 100))

    // Greedily include the highest-growth items that still fit the budget.
    const recommendations = computed(() => {
      let spent = 0
      const picked = []
      for (const c of candidates.value) {
        if (spent + c.lineCost <= budget.value) {
          picked.push(c)
          spent += c.lineCost
        }
      }
      return picked
    })

    const totalUnits = computed(() => recommendations.value.reduce((s, r) => s + r.quantity, 0))
    const totalCost = computed(() => recommendations.value.reduce((s, r) => s + r.lineCost, 0))
    const budgetRemaining = computed(() => Math.max(0, budget.value - totalCost.value))

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
        // Start with the full budget so all recommendations are visible, then
        // the user can drag it down.
        budget.value = maxBudget.value
      } catch (err) {
        error.value = t('restocking.orderError') + ': ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || placing.value) return
      try {
        placing.value = true
        placedMessage.value = ''
        const payload = {
          items: recommendations.value.map(r => ({
            sku: r.item_sku,
            name: r.item_name,
            quantity: r.quantity,
            unit_price: r.unit_cost
          })),
          lead_time_days: LEAD_TIME_DAYS,
          notes: 'Created from Restocking tab'
        }
        const order = await api.createOrder(payload)
        placedMessage.value = t('restocking.orderPlaced', { orderNumber: order.order_number })
      } catch (err) {
        placedMessage.value = ''
        error.value = t('restocking.orderError') + ': ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      currency,
      loading,
      error,
      budget,
      maxBudget,
      step,
      recommendations,
      totalUnits,
      totalCost,
      budgetRemaining,
      placing,
      placedMessage,
      placeOrder,
      formatCurrency,
      translateProductName
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-head {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 1rem;
  margin-bottom: 1rem;
}

.budget-hint {
  font-size: 0.813rem;
  color: #64748b;
  margin: 0.25rem 0 0;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  white-space: nowrap;
}

.budget-slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 999px;
  background: #e2e8f0;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.3);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border: none;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
}

.budget-scale {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.5rem;
}

.recommendations-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 1rem;
}

.recommendations-hint {
  font-size: 0.813rem;
  color: #64748b;
  margin: 0.25rem 0 0;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  margin-bottom: 1rem;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}

th.num,
td.num {
  text-align: right;
}
</style>
