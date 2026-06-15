<script setup>
import { ref, computed, watch } from 'vue'

// --- 1. СТЕЙТ ТА ІНІЦІАЛІЗАЦІЯ ДАНИХ ---
// Ціль тепер динамічна, за замовчуванням 80000, якщо немає в пам'яті
const goalTarget = ref(Number(localStorage.getItem('goal_target')) || 80000)
const isEditingGoal = ref(false)
const goalInput = ref(goalTarget.value)

// Загальна сума та історія
const totalSaved = ref(Number(localStorage.getItem('saved_money')) || 0)
const history = ref(JSON.parse(localStorage.getItem('money_history')) || [])
const inputAmount = ref(null)

// --- 2. ЖУРНАЛИ ЗБЕРЕЖЕННЯ (WATCH) ---
watch(goalTarget, (newGoal) => {
  localStorage.setItem('goal_target', newGoal)
})

watch(totalSaved, (newVal) => {
  localStorage.setItem('saved_money', newVal)
})

watch(history, (newHistory) => {
  localStorage.setItem('money_history', JSON.stringify(newHistory))
}, { deep: true })

// --- 3. ОБЧИСЛЮВАЛЬНІ ВЛАСТИВОСТІ (COMPUTED) ---
const remainingAmount = computed(() => {
  const diff = goalTarget.value - totalSaved.value
  return diff > 0 ? diff : 0
})

const progressPercentage = computed(() => {
  const percent = (totalSaved.value / goalTarget.value) * 100
  return percent > 100 ? 100 : Math.round(percent * 10) / 10
})

// Нова аналітика: Середній чек та кількість кроків, що залишилися
const analytics = computed(() => {
  if (history.value.length === 0) return { average: 0, stepsLeft: '—' }
  
  // Рахуємо середній внесок
  const sum = history.value.reduce((acc, item) => acc + item.amount, 0)
  const average = Math.round(sum / history.value.length)
  
  // Рахуємо скільки приблизно таких внесків ще треба зробити
  const stepsLeft = average > 0 ? Math.ceil(remainingAmount.value / average) : '—'
  
  return { average, stepsLeft }
})

// --- 4. МЕТОДИ (МЕНЕДЖМЕНТ ДАНИХ) ---
// Збереження нової цілі
const saveGoal = () => {
  if (goalInput.value && goalInput.value > 0) {
    goalTarget.value = goalInput.value
    isEditingGoal.value = false
  }
}

// Додавання грошей у конверт
const addAmount = () => {
  if (!inputAmount.value || inputAmount.value <= 0) return

  totalSaved.value += inputAmount.value

  const now = new Date()
  const formattedDate = now.toLocaleDateString('uk-UA', {
    day: '2-digit',
    month: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })

  history.value.unshift({
    id: Date.now(),
    amount: inputAmount.value,
    date: formattedDate
  })

  inputAmount.value = null
}

// Нова фіча: видалення помилкового внеску
const deleteAmount = (id, amount) => {
  if (confirm(`Видалити цей внесок на ${amount.toLocaleString('uk-UA')} грн?`)) {
    totalSaved.value -= amount
    history.value = history.value.filter(item => item.id !== id)
  }
}
</script>

<template>
  <div class="app-container">
    <div class="tracker-card">
      <header class="tracker-card__header">
        <h1>Мій Фінансовий Трекер 💸</h1>
        
        <div class="goal-setting">
          <div v-if="!isEditingGoal" class="goal-setting__view">
            <p>Ціль: <strong>{{ goalTarget.toLocaleString('uk-UA') }} грн</strong></p>
            <button @click="isEditingGoal = true; goalInput = goalTarget" class="btn-icon">✏️</button>
          </div>
          <form v-else @submit.prevent="saveGoal" class="goal-setting__edit">
            <input v-model.number="goalInput" type="number" min="1" required />
            <button type="submit" class="btn-save">Зберегти</button>
          </form>
        </div>
      </header>

      <div class="tracker-card__stats">
        <div class="stat-box">
          <span class="stat-box__label">Зібрано</span>
          <span class="stat-box__value saved">{{ totalSaved.toLocaleString('uk-UA') }} грн</span>
        </div>
        <div class="stat-box">
          <span class="stat-box__label">Залишилось</span>
          <span class="stat-box__value remaining">{{ remainingAmount.toLocaleString('uk-UA') }} грн</span>
        </div>
      </div>

      <div class="tracker-card__progress-wrapper">
        <div class="progress-bar" :style="{ width: progressPercentage + '%' }"></div>
        <span class="progress-text">{{ progressPercentage }}%</span>
      </div>

      <div class="tracker-card__analytics" v-if="history.length">
        <div class="analytics-item">
          <span class="analytics-label">Середній внесок:</span>
          <span class="analytics-value">{{ analytics.average.toLocaleString('uk-UA') }} грн</span>
        </div>
        <div class="analytics-item">
          <span class="analytics-label">Залишилось кроків:</span>
          <span class="analytics-value highlight">🎯 ще близько {{ analytics.stepsLeft }}</span>
        </div>
      </div>

      <form @submit.prevent="addAmount" class="tracker-card__form">
        <input 
          v-model.number="inputAmount"
          type="number" 
          placeholder="Введіть суму, яку відклали (н-р: 1250)"
          required
        />
        <button type="submit">У конверт 💵</button>
      </form>

      <div class="tracker-card__history">
        <h3>Історія внесків</h3>
        <ul v-if="history.length">
          <li v-for="item in history" :key="item.id" class="history-item">
            <span class="history-date">{{ item.date }}</span>
            <div class="history-right">
              <span class="history-amount">+{{ item.amount.toLocaleString('uk-UA') }} грн</span>
              <button @click="deleteAmount(item.id, item.amount)" class="btn-delete" title="Видалити запис">🗑️</button>
            </div>
          </li>
        </ul>
        <p v-else class="empty-text">Конверт порожній. Час додати перший заробіток! ☀️</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #f5f7fb;
  font-family: system-ui, -apple-system, sans-serif;
  padding: 20px;
}

.tracker-card {
  background: #ffffff;
  max-width: 450px;
  width: 100%;
  padding: 30px;
  border-radius: 16px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.05);
}

.tracker-card__header {
  text-align: center;
  margin-bottom: 25px;
}

.tracker-card__header h1 {
  font-size: 1.6rem;
  color: #1e293b;
  margin: 0 0 4px 0;
}

.goal-setting {
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 0.95rem;
  color: #64748b;
}

.goal-setting__view {
  display: flex;
  align-items: center;
  gap: 8px;
}

.goal-setting__view p {
  margin: 0;
}

.goal-setting__edit {
  display: flex;
  gap: 6px;
}

.goal-setting__edit input {
  width: 100px;
  padding: 4px 8px;
  border: 1px solid #cbd5e1;
  border-radius: 4px;
  font-size: 0.9rem;
}

.btn-icon {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 0.9rem;
  padding: 2px;
  transition: transform 0.2s;
}
.btn-icon:hover {
  transform: scale(1.2);
}

.btn-save {
  background-color: #64748b;
  color: white;
  border: none;
  padding: 4px 10px;
  border-radius: 4px;
  font-size: 0.8rem;
  cursor: pointer;
}

.tracker-card__stats {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.stat-box {
  display: flex;
  flex-direction: column;
}

.stat-box__label {
  font-size: 0.8rem;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 4px;
}

.stat-box__value {
  font-size: 1.25rem;
  font-weight: 700;
}

.stat-box__value.saved {
  color: #10b981;
}

.stat-box__value.remaining {
  color: #334155;
}

.tracker-card__progress-wrapper {
  position: relative;
  height: 24px;
  background-color: #f1f5f9;
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 18px;
}

.progress-bar {
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  background: linear-gradient(90deg, #10b981, #059669);
  transition: width 0.4s ease;
}

.progress-text {
  position: relative;
  font-size: 0.85rem;
  font-weight: 600;
  color: #1e293b;
  z-index: 2;
}

/* Стилі для аналітики */
.tracker-card__analytics {
  background-color: #f8fafc;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 20px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  border-left: 4px solid #3b82f6;
}

.analytics-item {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: #475569;
}

.analytics-value {
  font-weight: 600;
}
.analytics-value.highlight {
  color: #2563eb;
}

.tracker-card__form {
  display: flex;
  gap: 10px;
  margin-bottom: 25px;
}

.tracker-card__form input {
  flex: 1;
  padding: 12px;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  outline: none;
}

.tracker-card__form input:focus {
  border-color: #10b981;
}

.tracker-card__form button {
  background-color: #10b981;
  color: white;
  border: none;
  padding: 12px 16px;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  white-space: nowrap;
}

.tracker-card__form button:hover {
  background-color: #059669;
}

.tracker-card__history h3 {
  font-size: 1rem;
  color: #334155;
  margin: 0 0 12px 0;
  padding-bottom: 6px;
  border-bottom: 1px solid #f1f5f9;
}

.tracker-card__history ul {
  list-style: none;
  padding: 0;
  margin: 0;
  max-height: 180px;
  overflow-y: auto;
}

.history-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  border-bottom: 1px dashed #f1f5f9;
  font-size: 0.9rem;
}

.history-right {
  display: flex;
  align-items: center;
  gap: 10px;
}

.history-date {
  color: #64748b;
}

.history-amount {
  font-weight: 600;
  color: #10b981;
}

.btn-delete {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 0.85rem;
  padding: 2px;
  opacity: 0.4;
  transition: opacity 0.2s;
}

.history-item:hover .btn-delete {
  opacity: 1;
}

.empty-text {
  text-align: center;
  color: #94a3b8;
  font-size: 0.85rem;
  font-style: italic;
  margin: 10px 0 0 0;
}
</style>