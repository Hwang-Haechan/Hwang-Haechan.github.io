---
title: "Adding a Frontend and Refactoring Routes in My API Project"
date: 2026-06-22 00:00:00 +0900
categories: [Backend, Server]
tags: [nodejs, express, frontend, refactoring, rest-api]
---

## Overview

Built a frontend interface for the existing REST API and refactored the backend by separating route handlers into their own module.

## 1. Frontend Pages for CRUD Operations

Created five HTML pages under the `frontend/` directory, each dedicated to a specific CRUD operation:

- **index.html** — Displays a table of all users fetched from `GET /users`
- **create.html** — A form to add a new user via `POST /users`
- **read.html** — Look up a single user by ID via `GET /users/:id`
- **update.html** — Load an existing user, edit their info, and submit via `PUT /users/:id`
- **delete.html** — Delete a user by ID via `DELETE /users/:id`

All pages share a consistent design with a navigation bar for easy switching between operations. The frontend communicates with the API using `fetch()`.

## 2. Serving Static Files

Added `express.static` middleware in `app.js` to serve the frontend pages directly from the server:

```javascript
app.use(express.static(path.join(__dirname, 'frontend')));
```

This means visiting the server root now loads the frontend UI instead of just returning a JSON response.

## 3. Route Refactoring

Moved all user-related route handlers out of `app.js` and into a dedicated module at `routes/master/users.js`. The main `app.js` now simply mounts the router:

```javascript
const usersRouter = require('./routes/master/users');
app.use('/users', usersRouter);
```

This reduced `app.js` from over 100 lines down to under 20, making the codebase much cleaner and easier to maintain as more routes are added in the future.

## 4. NPM Scripts

Added convenience scripts to `package.json`:

```json
"start": "node app.js",
"dev": "nodemon app.js"
```

Now the server can be started with `npm start` or in development mode with `npm run dev` for automatic restarts on file changes.

## Takeaways

Separating concerns — frontend from backend logic, and route handlers from the main entry point — keeps the project organized as it grows. The frontend pages provide a user-friendly way to interact with the API without needing tools like Postman or curl.
