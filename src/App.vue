<template>

  <div id="header">
    <h1>Monthly Salary & Expense Tracker</h1>
    <label for="months">Month:</label>
    <input type="month" id="months" v-model="selectedMonth" />
    <p v-if="selectedMonth === ''">Select a month</p>
  </div>
  <div id="main">
    <div>
      <label for="salary">Salary:</label>
      <input type="number" id="salary" v-model.number="monthlySalary" min="0" :disabled="selectedMonth === ''" />
      <p v-if="!isSalaryValid && selectedMonth !== ''">Salary should be a number greater than or equal to zero</p>
    </div>
    <div>Expenses:
      <span id="expenses">{{ totalExpenses }}</span>
    </div>
    <div>Balance:
      <p v-if="remainingSalary === null">Check salary again</p>
      <span id="balance" v-else>{{ remainingSalary }}</span>
    </div>
    <p v-if="storageError" role="alert">{{ storageError }}</p>
  </div>

  <div id="spending-overall">
    <span>Spending Overview</span>
    <!-- Below image is just permanent shoing a pie chart of spending -->
    <!-- <img src="./assets/graph.png" alt="graph" /> -->
  </div>

  <div id="expense-list">
    <h2>Expenses</h2>
    <p v-if="monthlyExpenses.length === 0">No expenses yet</p>
    <table id="expense-table" border="1" v-else>
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
          <td>{{ expense.amount }}</td>
          <td>
            <input type="button" value="Delete" @click="deleteExpense(expense.id)" />
            <input type="button" value="Edit" @click="startEditExpense(expense.id)" />
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  <div id="add-expense-form">
    <h2>{{ editingExpenseId !== null ? 'Edit Expense' : 'Add Expense' }}</h2>
    <p v-if="expenseError">{{ expenseError }}</p>
    <form @submit.prevent="saveExpense">
      <label for="date">Date:</label>
      <input type="date" id="date" name="date" required v-model="newExpense.date" />
      <label for="category">Category:</label>
      <select id="category" name="category" required v-model="newExpense.category">
        <option value="" disabled>
          Select Category
        </option>
        <option v-for="category in categories" :key="category" :value="category">
          {{ category }}
        </option>
      </select>
      <label for="description">Description:</label>
      <input type="text" id="description" name="description" required v-model="newExpense.description" />
      <label for="amount">Amount:</label>
      <input type="number" id="amount" name="amount" required v-model.number="newExpense.amount" />
      <input type="submit" :value="editingExpenseId !== null ? 'Save Changes' : 'Add Expense'" />
      <input type="button" value="Cancel Edit" v-if="editingExpenseId !== null" @click="cancelEdit" />
    </form>
  </div>
  <div id="last-update">
    <span>Last Update: 2026-10-01 12:00 PM</span>
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

    const categories: string[] = [
      'Food',
      'Transportation',
      'Housing',
      'Utilities',
      'Entertainment',
      'Insurance',
      'Others'
    ]

    const storageError = ref<string>('')

    const expenseError = ref<string>('')

    const selectedMonth = ref<string>('2026-11')

    const salariesByMonth = reactive<Record<string, number>>(
      {
        '2026-10': 250000,
        '2026-11': 260000
      },
    )

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
          if (Number.isFinite(value) && value >= 0) {
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

    const expenses = reactive<Expense[]>([
      {
        id: 1,
        date: '2026-10-01',
        category: 'Food',
        description: 'Lunch',
        amount: 1000
      },
      {
        id: 2,
        date: '2026-10-02',
        category: 'Transportation',
        description: 'Train Ticket',
        amount: 880
      },
      {
        id: 3,
        date: '2026-11-01',
        category: 'Food',
        description: 'Lunch',
        amount: 2000
      },
      {
        id: 4,
        date: '2026-11-02',
        category: 'Transportation',
        description: 'Train Ticket',
        amount: 1000
      },
    ])


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
      if (typeof monthlySalary.value === 'number' && Number.isFinite(monthlySalary.value) && monthlySalary.value >= 0) {
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
        // Finite-number check
        !Number.isFinite(newExpense.amount)
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

      const dataUse = {
        expenses,
        salariesByMonth
      }
      storageError.value = ''

      for (const salary of Object.values(salariesByMonth)) {
        if (!(typeof salary === 'number') || !(Number.isFinite(salary)) || salary < 0) {
          storageError.value = 'Changes could not be saved. Correct invalid salaries first.'
          console.log('Rejected saved salary', salary)
          return
        }
      }

      try {
        localStorage.setItem('monthly-expense-tracker', JSON.stringify(dataUse))
      } catch (error) {
        storageError.value = 'Changes could not be saved. They may be lost after refreshing.'
        console.error('Could not save tracker data.', error);
      }
    }

    function loadData() {
      try {
        const result = localStorage.getItem('monthly-expense-tracker')
        if (result === null) {
          return
        }
        const parsedData: unknown = JSON.parse(result)
        if (typeof parsedData !== 'object' || parsedData === null || Array.isArray(parsedData)) {
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }
        if (!('expenses' in parsedData) || !('salariesByMonth' in parsedData)) {
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }
        if (!(Array.isArray(parsedData.expenses))) {
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }
        if (typeof parsedData.salariesByMonth !== 'object' || parsedData.salariesByMonth === null || Array.isArray(parsedData.salariesByMonth)) {
          storageError.value = 'Saved data is invalid and could not be loaded.';
          return;
        }

        const loadedSalaries: Record<string, number> = {}

        for (const [month, salary] of Object.entries(parsedData.salariesByMonth)) {
          if (!(typeof salary === 'number') || !(Number.isFinite(salary)) || salary < 0) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            console.log('Rejected saved salary')
            return
          }
          if (!(/^\d{4}-(0[1-9]|1[0-2])$/.test(month))) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          loadedSalaries[month] = salary
        }
        const loadedExpenses: Expense[] = []
        const seenIds = new Set<number>()

        for (const expense of parsedData.expenses) {
          const item: unknown = expense;
          if ((Array.isArray(item)) || typeof item !== 'object' || item === null) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            console.log('Invalid data found', item)
            return
          }
          if (!('id' in item) || !('date' in item) || !('category' in item) || !('description' in item) || !('amount' in item)) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            console.log('Expens field is missing', item)
            return
          }
          if (typeof item.amount !== 'number' || !(Number.isFinite(item.amount)) || item.amount <= 0) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (typeof item.description !== 'string' || item.description.trim() === '') {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (typeof item.category !== 'string' || !(categories.includes(item.category))) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (typeof item.id !== 'number' || !(Number.isSafeInteger(item.id)) || item.id <= 0) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          if (seenIds.has(item.id)) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          } else {
            seenIds.add(item.id)
          }
          if (typeof item.date !== 'string' || !(/^\d{4}-\d{2}-\d{2}$/.test(item.date))) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          const expenseDate = new Date(item.date)
          if (Number.isNaN(expenseDate.getTime())) {
            storageError.value = 'Saved data is invalid and could not be loaded.';
            return
          }
          const normalizedDate = expenseDate.toISOString().slice(0, 10);
          if (normalizedDate !== item.date) {
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
        console.log('loadedSalaries', loadedSalaries)
        console.log('loadedExpenses', loadedExpenses)
        console.log('ParsedData', parsedData)
      } catch (error) {
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
      selectedMonth,
      expenseError,
      storageError,
      monthlySalary,
      salariesByMonth,
      remainingSalary,
      monthlyExpenses,
      isSalaryValid,
      editingExpenseId,
      totalExpenses,
      expenses,
      newExpense,
      saveExpense,
      startEditExpense,
      deleteExpense,
      cancelEdit
    }
  }
})


</script>

<style scoped></style>
