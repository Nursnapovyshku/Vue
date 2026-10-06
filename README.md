# SIS 2 Vue Task Management App

Task Management App for the JS Framework Vue SIS 2 assignment.

The application components, task logic and styles are in `App.vue`.
`src/main.js` registers the global button component and mounts the app.

## Run Locally

```bash
npm install
npm run dev
```

## Build

```bash
npm run build
```

## Browser Tests

Start the development server with `npm run dev -- --port 5174`, then run `npm test`.
The tests use Microsoft Edge.

## Features

- Add, view, edit, complete, delete, search, and filter tasks.
- Filter tasks by priority and active or completed status.
- Change task priority directly from the task list.
- View task statistics for total, active, and completed tasks.
- Uses reusable Vue components, props, custom events, methods, computed properties, watchers, lifecycle hooks, slots, and a global component.
