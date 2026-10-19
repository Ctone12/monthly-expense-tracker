# Monthly Salary & Expense Tracker

A personal application for tracking monthly salary and expenses in Japanese yen. Built as a learning project to practice Vue 3, TypeScript, and Git.

## Technology

- Vue 3 with the Composition API
- TypeScript
- HTML and CSS
- Vite

## Features

- Separate salaries for each month
- Add, edit, and delete expenses
- Expense categories and monthly filtering
- Total expenses and remaining salary
- Category spending totals, percentages, and colored bars
- Japanese yen formatting and whole-yen validation
- localStorage persistence with saved-data validation
- Last successful save timestamp
- Responsive layouts and keyboard focus indicators
- Retrying failed loads and blocking saves until recovery succeeds

## Run Locally

Install dependencies:

```sh
npm install
```

Start the development server:

```sh
npm run dev
```

Open the local URL printed in the terminal.

## Checks

Check TypeScript types:

```sh
npm run type-check
```

Create a production build, including type-checking:

```sh
npm run build
```

## Limitations

- Data is stored in the current browser’s localStorage for the site address.
- Clearing that browser storage removes the saved data.
- Data does not sync between browsers or devices.
- There is no backend, database, or authentication.
- Last Update shows the last successful local save, not a live clock.
- LocalStorage is still not backuped no import/export yet.

## Learning Goals

Practice reactive state, computed values, watchers, forms, TypeScript interfaces, data validation, browser storage, responsive CSS, and Git.
