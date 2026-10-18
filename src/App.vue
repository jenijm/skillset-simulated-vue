<script setup>
import { computed, onMounted, ref, watch } from 'vue';

const storageKey = 'vue-expense-tracker';
const categories = ['Food', 'Transport', 'Shopping', 'Bills', 'Health', 'Other'];

const expenses = ref([]);
const description = ref('');
const amount = ref('');
const category = ref('Food');
const date = ref(new Date().toISOString().slice(0, 10));
const filter = ref('All');
const error = ref('');
const loaded = ref(false);

onMounted(() => {
  try {
    const saved = JSON.parse(localStorage.getItem(storageKey) || '[]');
    expenses.value = Array.isArray(saved) ? saved : [];
  } catch {
    expenses.value = [];
  }
  loaded.value = true;
});

watch(
  expenses,
  (currentExpenses) => {
    if (loaded.value) {
      localStorage.setItem(storageKey, JSON.stringify(currentExpenses));
    }
  },
  { deep: true },
);

const filteredExpenses = computed(() => {
  const matches = filter.value === 'All'
    ? expenses.value
    : expenses.value.filter((expense) => expense.category === filter.value);

  return [...matches].sort((a, b) => new Date(b.date) - new Date(a.date));
});

const totalSpent = computed(() =>
  expenses.value.reduce((total, expense) => total + expense.amount, 0),
);

const filteredTotal = computed(() =>
  filteredExpenses.value.reduce((total, expense) => total + expense.amount, 0),
);

const highestExpense = computed(() => {
  if (expenses.value.length === 0) return 0;
  return Math.max(...expenses.value.map((expense) => expense.amount));
});

function createId() {
  return globalThis.crypto?.randomUUID?.() ?? `${Date.now()}-${Math.random()}`;
}

function addExpense() {
  const cleanDescription = description.value.trim();
  const numericAmount = Number(amount.value);

  if (!cleanDescription) {
    error.value = 'Please enter an expense description.';
    return;
  }

  if (!Number.isFinite(numericAmount) || numericAmount <= 0) {
    error.value = 'Amount must be a number greater than zero.';
    return;
  }

  if (!date.value) {
    error.value = 'Please select a date.';
    return;
  }

  expenses.value.push({
    id: createId(),
    description: cleanDescription,
    amount: Number(numericAmount.toFixed(2)),
    category: category.value,
    date: date.value,
  });

  description.value = '';
  amount.value = '';
  category.value = 'Food';
  error.value = '';
}

function deleteExpense(expense) {
  if (window.confirm(`Delete “${expense.description}”?`)) {
    expenses.value = expenses.value.filter((item) => item.id !== expense.id);
  }
}

function clearError() {
  error.value = '';
}

function formatCurrency(value) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(value);
}

function formatDate(value) {
  return new Intl.DateTimeFormat('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  }).format(new Date(`${value}T00:00:00`));
}
</script>

<template>
  <main class="app-shell">
    <header class="page-header">
      <div>
        <span class="eyebrow">PERSONAL FINANCE</span>
        <h1>Expense Tracker</h1>
        <p>Keep an eye on where your money goes.</p>
      </div>
      <div class="wallet-icon" aria-hidden="true">$</div>
    </header>

    <section class="summary-grid" aria-label="Expense summary">
      <article class="summary-card primary">
        <span>Total spent</span>
        <strong>{{ formatCurrency(totalSpent) }}</strong>
        <small>Across {{ expenses.length }} {{ expenses.length === 1 ? 'expense' : 'expenses' }}</small>
      </article>
      <article class="summary-card">
        <span>Highest expense</span>
        <strong>{{ formatCurrency(highestExpense) }}</strong>
        <small>Your largest single entry</small>
      </article>
      <article class="summary-card">
        <span>Filtered total</span>
        <strong>{{ formatCurrency(filteredTotal) }}</strong>
        <small>{{ filter === 'All' ? 'All categories' : filter }}</small>
      </article>
    </section>

    <div class="content-grid">
      <section class="panel form-panel" aria-labelledby="add-expense-heading">
        <div class="panel-heading">
          <span class="heading-icon" aria-hidden="true">+</span>
          <div>
            <h2 id="add-expense-heading">Add expense</h2>
            <p>Record a new transaction.</p>
          </div>
        </div>

        <form @submit.prevent="addExpense">
          <label for="description">Description</label>
          <input
            id="description"
            v-model="description"
            maxlength="80"
            placeholder="e.g. Grocery shopping"
            @input="clearError"
          />

          <div class="form-row">
            <div>
              <label for="amount">Amount</label>
              <div class="amount-input">
                <span>$</span>
                <input
                  id="amount"
                  v-model="amount"
                  type="number"
                  min="0.01"
                  step="0.01"
                  placeholder="0.00"
                  @input="clearError"
                />
              </div>
            </div>
            <div>
              <label for="date">Date</label>
              <input id="date" v-model="date" type="date" @input="clearError" />
            </div>
          </div>

          <label for="category">Category</label>
          <select id="category" v-model="category">
            <option v-for="item in categories" :key="item" :value="item">{{ item }}</option>
          </select>

          <p class="form-error" :class="{ visible: error }" role="alert">{{ error }}</p>
          <button class="primary-button" type="submit">Add expense</button>
        </form>
      </section>

      <section class="panel expenses-panel" aria-labelledby="expenses-heading">
        <div class="list-heading">
          <div>
            <h2 id="expenses-heading">Recent expenses</h2>
            <p>{{ filteredExpenses.length }} {{ filteredExpenses.length === 1 ? 'entry' : 'entries' }}</p>
          </div>
          <select v-model="filter" aria-label="Filter expenses by category">
            <option value="All">All categories</option>
            <option v-for="item in categories" :key="item" :value="item">{{ item }}</option>
          </select>
        </div>

        <div v-if="filteredExpenses.length === 0" class="empty-state">
          <div aria-hidden="true">↗</div>
          <h3>No expenses found</h3>
          <p>{{ expenses.length === 0 ? 'Add your first expense using the form.' : 'Try a different category filter.' }}</p>
        </div>

        <ul v-else class="expense-list">
          <li v-for="expense in filteredExpenses" :key="expense.id">
            <span class="category-badge" :data-category="expense.category" aria-hidden="true">
              {{ expense.category.slice(0, 1) }}
            </span>
            <div class="expense-info">
              <strong>{{ expense.description }}</strong>
              <span>{{ expense.category }} · {{ formatDate(expense.date) }}</span>
            </div>
            <strong class="expense-amount">{{ formatCurrency(expense.amount) }}</strong>
            <button
              class="delete-button"
              type="button"
              :aria-label="`Delete ${expense.description}`"
              title="Delete expense"
              @click="deleteExpense(expense)"
            >×</button>
          </li>
        </ul>
      </section>
    </div>
  </main>
</template>
