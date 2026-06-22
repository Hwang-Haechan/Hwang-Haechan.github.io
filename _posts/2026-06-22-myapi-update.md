---
title: "Converting the Frontend from Vanilla HTML to React"
date: 2026-06-22 00:00:00 +0900
categories: [Frontend, React]
tags: [react, vite, react-router, javascript, spa]
---

## Overview

Converted the frontend of the API project from plain HTML pages to a single-page application (SPA) built with **React**, **Vite**, and **React Router**. The original five static HTML files have been replaced by React components, while the backend API remains unchanged.

## 1. Tech Stack

| Tool | Version | Role |
|------|---------|------|
| React | 19 | UI library |
| React Router | 7 | Client-side routing |
| Vite | 8 | Dev server & build tool |

All dependencies are declared in `frontend/package.json`. Running `npm run dev` starts Vite's dev server with hot module replacement, and `npm run build` produces an optimized production bundle in `frontend/dist`.

## 2. Project Structure

```
frontend/
├── src/
│   ├── main.jsx          # Entry point — mounts <App /> inside BrowserRouter
│   ├── App.jsx           # Top-level layout with NavLink navigation and Routes
│   ├── App.css           # Global styles
│   └── pages/
│       ├── AllUsers.jsx   # GET /users — lists all users in a table
│       ├── CreateUser.jsx # POST /users — form to create a new user
│       ├── FindUser.jsx   # GET /users/:id — look up a user by ID
│       ├── UpdateUser.jsx # PUT /users/:id — load then edit a user
│       └── DeleteUser.jsx # DELETE /users/:id — delete with confirmation
├── vite.config.js        # Vite config with API proxy
└── package.json
```

## 3. Client-Side Routing

Previously, each CRUD operation lived in its own HTML file (`create.html`, `read.html`, etc.) and navigation required full page reloads. Now, `App.jsx` defines all routes in one place using React Router:

```jsx
<Routes>
  <Route path="/" element={<AllUsers />} />
  <Route path="/create" element={<CreateUser />} />
  <Route path="/find" element={<FindUser />} />
  <Route path="/update" element={<UpdateUser />} />
  <Route path="/delete" element={<DeleteUser />} />
</Routes>
```

Navigation between pages is instant because React Router swaps components in the browser without a server round-trip. The `<NavLink>` component automatically highlights the active route.

## 4. API Proxy with Vite

During development, the React app runs on Vite's dev server (port 5173 by default), while the Express API runs on port 3400. To avoid CORS issues, Vite proxies API requests:

```js
server: {
  proxy: {
    '/users': 'http://localhost:3400',
  },
},
```

This means `fetch('/users')` in the React code is transparently forwarded to the Express backend.

## 5. Component Highlights

- **AllUsers** — Uses `useEffect` to fetch users on mount and displays them in a table with a loading state.
- **CreateUser** — Controlled form inputs with `useState`; clears the form on success and shows the created user's details.
- **FindUser** — Fetches a single user by ID and renders the result card below the form.
- **UpdateUser** — Two-step flow: first load the user, then display an edit form pre-filled with current values.
- **DeleteUser** — Calls `window.confirm()` before sending the DELETE request as a safeguard.

## Takeaways

Moving from static HTML pages to React introduced component-based architecture, declarative routing, and state management with hooks. Vite's fast dev server and API proxy made the development experience smooth, and the SPA approach eliminated full-page reloads for a more responsive user interface.
