<template>
  <div class="tracker-page">
    <div id="header">
      <h1>Monthly Salary & Expense Tracker</h1>
      <label for="months">Month:</label>
      <input type="month" id="months" v-model="selectedMonth" />
      <p v-if="selectedMonth === ''">Select a month</p>
    </div>
    <div id="main">
      <div>
        <label for="salary">Salary:</label>
        <input type="number" id="salary" v-model.number="monthlySalary" min="0" :disabled="selectedMonth === ''"
          step="1" />
        <p v-if="!isSalaryValid && selectedMonth !== ''" class="error-message">Salary should be a whole number greater
          than or
          equal to zero</p>
      </div>
      <div>Expenses:
        <span id="expenses">{{ formatAmount(totalExpenses) }}</span>
      </div>
      <div>Balance:
        <p v-if="remainingSalary === null">Unavailable</p>
        <span id="balance" v-else>{{ formatAmount(remainingSalary) }}</span>
      </div>
      <p v-if="storageError" role="alert" class="error-message">{{ storageError }}</p>
    </div>

    <div id="spending-overall">
      <h2>Spending Overview</h2>
      <p v-if="monthlyExpenses.length === 0">No spending for this month.</p>
      <ul class="category-list" v-else>
        <li v-for="(total, category) in categoryTotals" :key="category">
          {{ category }} {{ formatAmount(total) }} {{ categoryPercentage(total).toFixed(1) + '%' }}
          <div class="category-bar" aria-hidden="true">
            <div class="category-bar-fill"
              :style="{ width: categoryPercentage(total) + '%', backgroundColor: categoryColors[category] ?? '#6b7280' }">
            </div>
          </div>
        </li>
      </ul>
      <!-- Below image is just permanent shoing a pie chart of spending -->
      <!-- <img src="./assets/graph.png" alt="graph" /> -->
    </div>

    <div id="expense-list">
      <h2>Expenses</h2>
      <p v-if="monthlyExpenses.length === 0">No expenses yet</p>
      <div class="table-scroll" v-else>
        <table id="expense-table">
          <thead>
            <tr>
              <th>Date</th>
              <th>Category</th>
              <th>Description</th>
              <th>Amount</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            <!-- Expense rows will be dynamically added here -->
            <tr v-for="expense in monthlyExpenses" :key="expense.id">
              <td>{{ expense.date }}</td>
              <td>{{ expense.category }}</td>
              <td>{{ expense.description }}</td>
              <td>{{ formatAmount(expense.amount) }}</td>
              <td>
                <input type="button" value="Delete" class="table-row-delete" @click="deleteExpense(expense.id)" />
                <input type="button" value="Edit" class="table-row-edit" @click="startEditExpense(expense.id)" />
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div id="add-expense-form">
      <h2>{{ editingExpenseId !== null ? 'Edit Expense' : 'Add Expense' }}</h2>
      <p v-if="expenseError" class="error-message">{{ expenseError }}</p>
      <form @submit.prevent="saveExpense">
        <div class="form-field">
          <label for="date">Date:</label>
          <input type="date" id="date" name="date" required v-model="newExpense.date" />
        </div>
        <div class="form-field">
          <label for="category">Category:</label>
          <select id="category" name="category" required v-model="newExpense.category">
            <option value="" disabled>
              Select Category
            </option>
            <option v-for="category in categories" :key="category" :value="category">
              {{ category }}
            </option>
          </select>
        </div>
        <div class="form-field">
          <label for="description">Description:</label>
          <input type="text" id="description" name="description" required v-model="newExpense.description" />
        </div>
        <div class="form-field">
          <label for="amount">Amount:</label>
          <input type="number" id="amount" name="amount" required v-model.number="newExpense.amount" step="1" />
        </div>
        <div class="form-actions">
          <input type="submit" :value="editingExpenseId !== null ? 'Save Changes' : 'Add Expense'" />
          <input type="button" value="Cancel Edit" v-if="editingExpenseId !== null" @click="cancelEdit" />
        </div>
      </form>
    </div>
    <div id="last-update">
      <p>Last Update: {{ lastUpdated !== null ? formatLastUpdated(lastUpdated) : 'Not saved yet' }} </p>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, computed, watch } from 'vue';

interface Expense {
  id: number;
  date: string;
  category: string;
  description: string;
  amount: number;
}

export default defineComponent({
  name: 'App',

  setup() {

    const amountFormatter = new Intl.NumberFormat('ja-JP', {
      style: 'currency',
      currency: 'JPY',
    })

    const categories: string[] = [
      'Food',
      'Transportation',
      'Housing',
      'Utilities',
      'Entertainment',
      'Insurance',
      'Others'
    ]

    const categoryColors: Record<string, string> = {
      Food: '#F5CC27',
      Transportation: '#27F2F5',
      Housing: '#F78B8B',
      Utilities: '#2768F5',
      Entertainment: '#580BA3',
      Insurance: '#630419',
      Others: '#5D4B66'

    }

    const storageError = ref<string>('')

    const loadFailed = ref<boolean>(false)

    const expenseError = ref<string>('')

    const selectedMonth = ref<string>('2026-10')

    const lastUpdated = ref<string | null>(null)

    const salariesByMonth = reactive<Record<string, number>>({})

    const monthlySalary = computed({
      get() {
        return salariesByMonth[selectedMonth.value]
      },
      set(value: number | string) {
        if (selectedMonth.value === '') {
          return
        }
        if (typeof value === 'number') {
          salariesByMonth[selectedMonth.value] = value
          if (Number.isSafeInteger(value) && value >= 0) {
            saveData()
          }
        } else if (value === '') {
          delete salariesByMonth[selectedMonth.value]
          saveData()
        }
      }
    })

    const newExpense = reactive({
      date: '',
      category: '',
      description: '',
      amount: 0
    })

    const expenses = reactive<Expense[]>([])

    const monthlyExpenses = computed(() => {
      return expenses.filter(expense => {
        const expenseMonth = expense.date.slice(0, 7);
        return expenseMonth === selectedMonth.value
      });
    })

    const editingExpenseId = ref<number | null>(null)

    const totalExpenses = computed(() => {
      // console.log('Calculating total expenses...');
      const total = monthlyExpenses.value.reduce((total, expense) => total + expense.amount, 0);
      // console.log('Total expenses calculated:', total);
      return total;
    })

    const isSalaryValid = computed(() => {
      if (typeof monthlySalary.value === 'number' && Number.isSafeInteger(monthlySalary.value) && monthlySalary.value >= 0) {
        return true
      } else {
        return false
      }
    })

    const remainingSalary = computed(() => {
      const salary = monthlySalary.value
      if (typeof salary === 'number') {
        if (isSalaryValid.value) {
          return salary - totalExpenses.value;
        } else {
          return null;
        }
      } return null;
    })

    const categoryTotals = computed(() => {
      const totals: Record<string, number> = {}
      for (const expense of monthlyExpenses.value) {
        const currentTotal = totals[expense.category] ?? 0;
        totals[expense.category] = currentTotal + expense.amount
      }
      return totals
    })

    function formatAmount(amount: number) {
      return amountFormatter.format(amount)
    }

    function categoryPercentage(total: number) {
      if (totalExpenses.value === 0) {
        return 0
      }
      // console.log(total / totalExpenses.value * 100)
      return total / totalExpenses.value * 100

    }

    function formatLastUpdated(timestamp: string) {
      const date = new Date(timestamp)
      return date.toLocaleString('ja-JP')
    }

    function deleteExpense(id: number) {
      const index = expenses.findIndex(expense => expense.id === id);
      if (index !== -1) {
        expenses.splice(index, 1);
        saveData()
        if (editingExpenseId.value === id) {
          cancelEdit()
        }
      }

    }

    function startEditExpense(id: number) {
      const item = expenses.find(expense => expense.id === id)
      expenseError.value = ''
      if (item) {
        newExpense.date = item.date;
        newExpense.category = item.category;
        newExpense.description = item.description;
        newExpense.amount = item.amount
        editingExpenseId.value = item.id
        console.log(editingExpenseId.value)
      }
    }

    function saveExpense() {

      expenseError.value = ''

      if (!newExpense.date ||
        !newExpense.category.trim() ||
        !newExpense.description.trim() ||
        !(newExpense.amount > 0) ||
        typeof newExpense.amount !== 'number' ||
        !Number.isSafeInteger(newExpense.amount)
      ) {
        expenseError.value = 'Fields should not be blank. Amount must be greater than 0';
        return;
      }

      if (editingExpenseId.value !== null) {
        const item = expenses.find(expense => expense.id === editingExpenseId.value)
        if (!item) {
          editingExpenseId.value = null;
          return
        }
        // Update Expense
        item.date = newExpense.date;
        item.category = newExpense.category.trim().replace(/\s+/g, " ");
        item.description = newExpense.description.trim().replace(/\s+/g, " ");
        item.amount = newExpense.amount;


      } else {
        // Add Expense
        const newId = expenses.length ? Math.max(...expenses.map(expense => expense.id)) + 1 : 1;
        expenses.push({
          id: newId,
          date: newExpense.date,
          category: newExpense.category.trim().replace(/\s+/g, " "),
          description: newExpense.description.trim().replace(/\s+/g, " "),
          amount: newExpense.amount
        });
      }
      saveData()
      // Reset the form fields
      resetForm()

    }

    function saveData() {

      if (loadFailed.value) {
        storageError.value = 'Saving is blocked because saved data could not be loaded.'
        return
      }

      storageError.value = ''

      for (const salary of Object.values(salariesByMonth)) {
        if (!(typeof salary === 'number') || !(Number.isSafeInteger(salary)) || salary < 0) {
          storageError.value = 'Changes could not be saved. Correct invalid salaries first.'
          console.log('Rejected saved salary', salary)
          return
        }
      }
      const savedAt: string = new Date().toISOString();
      const dataUse = {
        expenses,
        salariesByMonth,
        lastUpdated: savedAt
      }
      try {
        // throw new Error('Test storage failure');
        localStorage.setItem('monthly-expense-tracker', JSON.stringify(dataUse));
        lastUpdated.value = savedAt;
      } catch (error) {
        storageError.value = 'Changes could not be saved. They may be lost after refreshing.'
        console.error('Could not save tracker data.', error);
      }
    }

    function loadData() {
      try {
        // throw new Error('Test storage failure');
        const result = localStorage.getItem('monthly-expense-tracker')
        if (result === null) {
          return
        }
        const parsedData: unknown = JSON.parse(result)

        if (typeof parsedData !== 'object' || parsedData === null || Array.isArray(parsedData)) {
          loadFailed.value = true;
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }
        if (!('expenses' in parsedData) || !('salariesByMonth' in parsedData)) {
          loadFailed.value = true;
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }
        if (!(Array.isArray(parsedData.expenses))) {
          loadFailed.value = true;
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }
        if (typeof parsedData.salariesByMonth !== 'object' || parsedData.salariesByMonth === null || Array.isArray(parsedData.salariesByMonth)) {
          loadFailed.value = true;
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }

        const loadedSalaries: Record<string, number> = {}

        for (const [month, salary] of Object.entries(parsedData.salariesByMonth)) {
          if (!(typeof salary === 'number') || !(Number.isSafeInteger(salary)) || salary < 0) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            console.log('Rejected saved salary')
            return
          }
          if (!(/^\d{4}-(0[1-9]|1[0-2])$/.test(month))) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          loadedSalaries[month] = salary
        }

        let loadedLastUpdated: string | null = null;

        if ('lastUpdated' in parsedData) {
          if (typeof parsedData.lastUpdated !== 'string') {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return;
          }

          const savedDate = new Date(parsedData.lastUpdated);

          if (Number.isNaN(savedDate.getTime())) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return;
          }

          loadedLastUpdated = parsedData.lastUpdated;
        }

        const loadedExpenses: Expense[] = []
        const seenIds = new Set<number>()

        for (const expense of parsedData.expenses) {
          const item: unknown = expense;
          if ((Array.isArray(item)) || typeof item !== 'object' || item === null) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            console.log('Invalid data found', item)
            return
          }
          if (!('id' in item) || !('date' in item) || !('category' in item) || !('description' in item) || !('amount' in item)) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            console.log('Expens field is missing', item)
            return
          }
          if (typeof item.amount !== 'number' || !(Number.isSafeInteger(item.amount)) || item.amount <= 0) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (typeof item.description !== 'string' || item.description.trim() === '') {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (typeof item.category !== 'string' || !(categories.includes(item.category))) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (typeof item.id !== 'number' || !(Number.isSafeInteger(item.id)) || item.id <= 0) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (seenIds.has(item.id)) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          } else {
            seenIds.add(item.id)
          }
          if (typeof item.date !== 'string' || !(/^\d{4}-\d{2}-\d{2}$/.test(item.date))) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          const expenseDate = new Date(item.date)
          if (Number.isNaN(expenseDate.getTime())) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          const normalizedDate = expenseDate.toISOString().slice(0, 10);
          if (normalizedDate !== item.date) {
            loadFailed.value = true;
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          loadedExpenses.push({
            id: item.id,
            date: item.date,
            category: item.category.trim().replace(/\s+/g, " "),
            description: item.description.trim().replace(/\s+/g, " "),
            amount: item.amount
          })
        }
        for (const month of Object.keys(salariesByMonth)) {
          delete salariesByMonth[month]
        }
        Object.assign(salariesByMonth, loadedSalaries)
        expenses.splice(0, expenses.length, ...loadedExpenses);
        lastUpdated.value = loadedLastUpdated;
        console.log('loadedSalaries', loadedSalaries)
        console.log('loadedExpenses', loadedExpenses)
        console.log('ParsedData', parsedData)
      } catch (error) {
        loadFailed.value = true;
        storageError.value = 'Saved data could not be loaded.'
        console.error('Saved data could not be loaded.', error)
      }
    }

    function cancelEdit() {
      resetForm()
    }

    function resetForm() {
      newExpense.date = '';
      newExpense.category = '';
      newExpense.description = '';
      newExpense.amount = 0;
      editingExpenseId.value = null;
      expenseError.value = ''
    }

    loadData()

    watch(selectedMonth, () => {
      resetForm()
    })

    return {
      categories,
      categoryColors,
      lastUpdated,
      selectedMonth,
      expenseError,
      storageError,
      monthlySalary,
      salariesByMonth,
      remainingSalary,
      categoryTotals,
      monthlyExpenses,
      isSalaryValid,
      editingExpenseId,
      totalExpenses,
      expenses,
      newExpense,
      formatAmount,
      categoryPercentage,
      formatLastUpdated,
      saveExpense,
      startEditExpense,
      deleteExpense,
      cancelEdit
    }
  }
})


</script>

<style scoped>
.tracker-page {
  max-width: 1000px;
  margin-inline: auto;
  padding: 24px 16px;
}

#main {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 16px;
  margin-block: 30px;
}

#main>p {
  grid-column: 1 / -1;
}

.error-message {
  color: #b42318;
  margin-block: 8px;
}

#add-expense-form form {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px
}

.form-field {
  display: grid;
  gap: 6px;
  min-width: 0;
}

.form-field input,
.form-field select {
  width: 100%;
  min-width: 0;
  box-sizing: border-box;
  padding: 8px 10px;
  border: 1px solid #044692;
  border-radius: 4px;
  font: inherit;
}

.form-actions {
  grid-column: 1 / -1;
  display: flex;
  flex-wrap: wrap;
  gap: 8px
}

#expense-table {
  width: 100%;
  border-collapse: collapse;
}

#expense-table th,
#expense-table td {
  padding: 8px;
  border-bottom: 1px solid #d1d5db;
  text-align: left;
}

#expense-table td:nth-child(3) {
  overflow-wrap: anywhere;
}

#expense-table td:nth-child(1),
#expense-table td:nth-child(5) {
  white-space: nowrap;
}


#expense-table th:nth-child(4),
#expense-table td:nth-child(4) {
  text-align: right;
}

.table-row-delete {
  background-color: #e0000b;
  color: rgb(230, 234, 236);
  padding: 5px;
  margin-right: 5px;
  font: inherit;
  border: 1px solid;
  border-radius: 4px;
  cursor: pointer;
}

.table-row-delete:focus-visible {
  outline: 2px solid #e0000b;
  outline-offset: 2px;
}

.table-row-edit {
  background-color: #286dbb;
  color: rgb(230, 234, 236);
  padding: 5px;
  margin-left: 5px;
  font: inherit;
  border: 1px solid;
  border-radius: 4px;
  cursor: pointer;
}

.table-row-edit:focus-visible {
  outline: 2px solid #286dbb;
  outline-offset: 2px;
}

.form-actions input {
  padding: 10px;
  font: inherit;
  border: 1px solid;
  border-radius: 4px;
  cursor: pointer;
}

.form-actions input[type="submit"] {
  background-color: #286dbb;
  font: inherit;
  color: rgb(230, 234, 236)
}

.form-actions input[type="submit"]:focus-visible {
  outline: 2px solid #286dbb;
  outline-offset: 2px;
}

.form-actions input[type="button"] {
  background-color: #e0000b;
  font: inherit;
  color: rgb(230, 234, 236)
}

.form-actions input[type="button"]:focus-visible {
  outline: 2px solid #e0000b;
  outline-offset: 2px;
}

.table-scroll {
  overflow-x: auto;
}

#expense-table {
  min-width: 600px;
}

.category-list {
  list-style: none;
  padding: 0;
}

.category-bar {
  height: 10px;
  background-color: #e5e7eb;
  margin-block: 6px 16px;
}

.category-bar-fill {
  height: 100%;
  background-color: #167047;
}


@media (max-width: 600px) {
  #main {
    grid-template-columns: 1fr;
    padding: 16px;
    column-gap: 24px;
    row-gap: 12px;
  }

  #add-expense-form form {
    grid-template-columns: 1fr;
  }

}
</style>
