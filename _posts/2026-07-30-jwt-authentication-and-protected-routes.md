---
title: "Adding Signup Password Hashing, JWT Login, and Protected Routes"
date: 2026-07-30 00:00:00 +0900
categories: [Backend, Authentication]
tags: [express, jwt, bcrypt, authentication, react]
---

## Why This Was Needed

Up to this point, `myapi` had no concept of a logged-in user. Any client could call `PUT /users/:id` or `DELETE /users/:id` for *any* user ID, not just their own. Passwords were already hashed with `bcrypt` on signup, but there was no login endpoint and nothing verifying who was making a request. This update closes that gap: a real login flow backed by JWTs, and route-level checks so users can only modify their own data.

## Fail Fast Without a JWT Secret

Since every protected route depends on `process.env.JWT_SECRET` being set, `app.js` now checks for it before mounting any routers:

```js
if (typeof process.env.JWT_SECRET !== 'string' || process.env.JWT_SECRET.trim() === '') {
  console.error('JWT_SECRET이 설정되지 않았습니다. .env에 JWT_SECRET을 추가하세요 (.env.example 참고)');
  process.exit(1);
}
```

Rather than let the server boot and silently fail (or crash) the first time someone hits a protected route, it refuses to start at all. `.env.example` was added alongside this so the required variable is documented, with a one-liner for generating a secret:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

## The Login Endpoint

`routes/master/auth.js` adds `POST /login`, which looks up the user by email and compares the submitted password against the stored bcrypt hash:

```js
if (results.length === 0) {
  res.status(401).json({ message: '이메일 또는 비밀번호가 일치하지 않습니다' });
  return;
}

const user = results[0];
if (!bcrypt.compareSync(password, user.password)) {
  res.status(401).json({ message: '이메일 또는 비밀번호가 일치하지 않습니다' });
  return;
}
```

A deliberate detail here: an unknown email and a wrong password return the exact same message and status code. Distinguishing them would let an attacker enumerate which emails are registered just by watching the response change — collapsing both cases into one generic message avoids leaking that.

On success, the endpoint signs a JWT with a 1 hour expiry and returns it alongside basic user info:

```js
const token = jwt.sign(
  { user_id: user.user_id, email: user.email },
  process.env.JWT_SECRET,
  { expiresIn: '1h' }
);
```

## Verifying Tokens on Protected Routes

`middleware/verifyToken.js` reads the `Authorization: Bearer <token>` header, verifies it, and attaches the decoded payload to `req.user`:

```js
const token = header.slice('Bearer '.length).trim();
const payload = jwt.verify(token, process.env.JWT_SECRET);
req.user = { user_id: payload.user_id, email: payload.email };
```

Missing headers, malformed headers, and invalid/expired tokens all return `401` before the request ever reaches a route handler.

In `routes/master/users.js`, `PUT /:id` and `DELETE /:id` are now wrapped with this middleware, plus an ownership check:

```js
if (req.user.user_id !== id) {
  res.status(403).json({ message: '본인의 정보만 수정할 수 있습니다' });
  return;
}
```

Being logged in is no longer enough — being logged in *as that specific user* is what's required to edit or delete their record.

### Route Ordering Gotcha

A new `GET /users/me` route returns the current user's own info from the token, without needing an ID in the URL. It has to be registered **before** `GET /users/:id`:

```js
// '/:id' 보다 반드시 먼저 등록해야 한다. 뒤에 두면 '/me' 요청이 ':id'에 먼저 걸린다.
router.get('/me', verifyToken, (req, res) => { ... });
router.get('/:id', (req, res) => { ... });
```

Express matches routes in registration order, so if `/:id` came first, a request to `/users/me` would match it with `id = "me"` instead of ever reaching the intended handler.

## Frontend: Storing and Reacting to Auth State

`frontend/src/auth.js` centralizes token/user storage in `localStorage` and dispatches a custom event whenever auth state changes, so any component can subscribe without prop drilling:

```js
export function saveAuth({ token, user_id, user_name, email }) {
  localStorage.setItem(TOKEN_KEY, token);
  localStorage.setItem(USER_KEY, JSON.stringify({ user_id, user_name, email }));
  window.dispatchEvent(new Event(AUTH_EVENT));
}
```

A new `Login` page posts credentials to `/login` and calls `saveAuth` on success. `App.jsx` subscribes to auth changes and shows the logged-in username with a logout button in the nav bar when a session exists. `UpdateUser` and `DeleteUser` now check `isLoggedIn()` before submitting and attach `authHeaders()` (the `Bearer` token) to their requests — client-side checks for a fast error message, with the server-side `verifyToken` middleware as the actual enforcement.

## Result

Editing or deleting a user account now requires being logged in as that exact user, both in the UI and at the API layer. The next natural step is adding a signup-to-login redirect and handling token expiry gracefully in the frontend instead of just failing the next request.

## Key Takeaways

- Never let an auth endpoint's response distinguish "wrong password" from "no such account" — collapse both into one generic message to avoid user enumeration.
- Checking `isLoggedIn()` in the UI is a UX nicety, not security — the real enforcement has to live in server-side middleware, since a client can always be bypassed.
- Express routes match in registration order: a specific path like `/me` must be declared before a param route like `/:id`, or the param route swallows it.
- Fail fast at startup if a required secret (like `JWT_SECRET`) is missing, rather than letting the server run in a half-broken state.
