---
title: "Origin, CORS, and Why Frontend/Backend Domains Are Split: A Complete Guide"
date: 2026-08-04 00:00:00 +0900
categories: [Web, Networking]
tags: [cors, origin, bff, backend, web]
---

# Origin, CORS, and Why Frontend/Backend Domains Are Split: A Complete Guide

Anyone who's done web development has run into a CORS error at some point. This post covers what an Origin actually is, why CORS errors happen, why frontend and backend are usually split across different domains in practice, and finally the BFF pattern.

## 1. What is an Origin?

Let's break down the URL `https://sub.domain.com:12345`.

```
https://sub.domain.com:12345
  ↑          ↑           ↑
scheme      host        port
```

**Origin = scheme + host + port**

If even one of these three differs, the browser treats it as a "different origin."

| URL | Same origin as the original? | Reason |
|---|---|---|
| `http://sub.domain.com:12345` | ❌ No | scheme differs (http vs https) |
| `https://sub.domain.com:8080` | ❌ No | port differs |
| `https://domain.com:12345` | ❌ No | host differs (no "sub") |
| `https://sub.domain.com:12345/api/users` | ✅ Yes | path isn't part of the origin |

The path, query string, and hash are not included in the origin comparison at all.

## 2. What is CORS (Cross-Origin Resource Sharing)?

Browsers enforce a fundamental security rule called the **SOP (Same-Origin Policy)**:

> JavaScript running on one origin should not be able to send a request to a different origin and freely read the response.

### Why this rule exists

Without this rule, the following scenario would be possible:

1. You're logged into your bank's site (`https://mybank.com`) and its login cookie is stored in your browser
2. Without realizing it, you visit a malicious site (`https://evil.com`)
3. `evil.com`'s script secretly sends a request to `https://mybank.com/api/account`
4. The browser automatically attaches your login cookie to that request
5. The response — your account info — gets read directly by `evil.com`'s script

To prevent this, the browser's default behavior is: "the cross-origin request may still be sent, but reading the response from script is blocked by default."

**CORS = a mechanism where the server tells the browser, "requests from this origin are allowed."**

It's essentially an exception-approval process layered on top of SOP's default blocking behavior.

## 3. Why does a CORS error occur?

> A CORS error happens when the server's response doesn't include the "this origin is allowed" header, so the browser refuses to hand the response over to the script.

### The sequence of events

1. Frontend JS (on `https://myapp.com`) calls `fetch('https://api.myapp.com/users')`
2. The browser actually sends the request to the server (the request itself does go out!)
3. The server sends back a response
4. The browser checks the response headers — specifically whether `Access-Control-Allow-Origin` exists and whether it includes `https://myapp.com`
5. If the condition is satisfied → the response is passed to the JS code normally
6. If not → a **CORS error** appears in the console, and the response body is blocked from the JS code

### A common misconception

> ⚠️ "If I get a CORS error, it means the server never received the request." → **Wrong.**

The server received the request, processed it, and sent a response just fine. The browser simply refused to hand that response over to the JS. That's why server logs show the request as successfully handled, while the frontend console shows an error.

### Preflight requests

If a request is a `PUT`/`DELETE`, or includes a custom header like `Content-Type: application/json` (a so-called "non-simple" request), the browser first sends an `OPTIONS` request asking "is this okay to send?" before the actual request.

```
OPTIONS /users HTTP/1.1
Origin: https://myapp.com
Access-Control-Request-Method: PUT
```

If the server doesn't respond with the right approval headers, the actual request never even gets sent. That's why you'll often see only an `OPTIONS` call in the Network tab, with no actual `PUT` request following it.

### How backend developers fix it

```
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

- `Access-Control-Allow-Origin: *` → allows all origins (but this can't be used for requests that include cookies/credentials)
- To send cookies or other credentials, you also need `Access-Control-Allow-Credentials: true`, and in that case `Allow-Origin` must specify the exact origin instead of `*`

## 4. Why split frontend and backend into different subdomains?

The pattern of `myapp.com` (frontend) and `api.myapp.com` (backend) being on separate domains isn't done to create CORS on purpose — it's a structure that **naturally emerges from how services are actually operated**.

### Deployment and scaling units differ

| | Frontend (`myapp.com`) | Backend API (`api.myapp.com`) |
|---|---|---|
| What it is | Static files (HTML/CSS/JS bundle) | A running application server |
| Where it's deployed | CDN | Actual servers/containers |
| Scaling method | CDN caching | Auto-scaling |
| Deploy frequency | Often (every UI change) | Whenever the API spec changes |

Since these are fundamentally different programs running in physically different places, it's natural for the domains to be different too.

### Multiple clients share the same API

A backend API is often shared across:

- The web frontend (`myapp.com`)
- Mobile apps (iOS/Android)
- Admin dashboard (`admin.myapp.com`)
- External partner integrations

When multiple clients share the same API server, it shouldn't be tied to any single frontend's domain. Having its own address (`api.myapp.com`) makes clear that it's the shared data gateway for the whole service.

### Team/organizational separation

In practice, frontend and backend teams often have separate CI/CD pipelines, server configs, and sometimes even different cloud regions. Splitting subdomains makes the infrastructure boundary between teams explicit.

### Caching strategies are completely different

- Frontend static files → cached long-term at the CDN
- API responses → mostly can't be cached (real-time data)

Since the caching policies are essentially opposite, it's much easier to manage them as separate infrastructure from the ground up.

### CORS is just a side effect

```
The service structure benefits from separation
    → this automatically triggers browser SOP → CORS issues
    → which are resolved with CORS headers on the server
```

A CORS error isn't a downside of this structure — it's a side effect of the browser's security rules, and it can be resolved with a few header settings. That's not a good enough reason to give up the infrastructural and organizational benefits described above.

## 5. Unifying origins: reverse proxy vs. BFF

If you'd rather not deal with CORS at all, you can put another server in front of the frontend so the browser sees everything as "the same origin." A **simple reverse proxy** and a **BFF** look similar here, but they play different roles.

```
[Simple Reverse Proxy] ---------- [BFF] ---------- [Full Backend]
   just forwards requests      aggregates/transforms      handles all
   (traffic relay)             requests, handles auth      business logic
```

### Simple reverse proxy (e.g., Nginx)

Just forwards requests and responses as-is — a "relay station" that doesn't touch the content.

```
Browser → requests myapp.com/api/users
        → Nginx forwards it as-is to api.myapp.com/users
        → the response is passed back to the browser unchanged
```

```nginx
location /api/ {
    proxy_pass https://api.myapp.com/;
}
```

No code required — just a config file.

### BFF (Backend For Frontend)

This is a real server that **aggregates and transforms** data from multiple APIs, rather than just forwarding requests.

For example, to render a mobile app's home screen, you might need to call four separate APIs: user profile, recent orders, recommendations, and notification count.

- **With a proxy**: the frontend has to make all 4 requests itself; the proxy just forwards each one
- **With a BFF**: the frontend calls `GET /home-summary` once, and the BFF internally calls all 4 APIs, combines the results, and returns a single response

```javascript
// Example BFF server code (conceptual)
app.get('/home-summary', async (req, res) => {
  const [profile, orders, recommendations, notifications] = await Promise.all([
    fetch('https://api.myapp.com/profile'),
    fetch('https://api.myapp.com/orders/recent'),
    fetch('https://api.myapp.com/recommendations'),
    fetch('https://api.myapp.com/notifications/count'),
  ]);

  res.json({
    userName: profile.name,
    recentOrderCount: orders.length,
    recommendedItems: recommendations.slice(0, 3),
    unreadNotifications: notifications.count,
  });
});
```

### Summary

| | Simple Reverse Proxy | BFF |
|---|---|---|
| What it does | Forwards requests as-is | Calls multiple APIs and aggregates/transforms them |
| Code required? | Barely any (just config) | Yes — real application logic |
| Purpose | Mainly origin unification, routing | Serving data optimized for the frontend |
| Solves CORS? | ✅ Yes | ✅ Yes |

Both give you the same effect of "unifying origins to sidestep CORS," but a BFF goes one step further by acting as "a dedicated server that pre-assembles exactly the data a given frontend screen needs." Think of a BFF as a proxy's forwarding capability plus real business logic layered on top.

## TL;DR

- **Origin**: the combination of scheme + host + port
- **CORS**: a mechanism where the server explicitly grants, via headers, permission for the browser to hand over a cross-origin response — a response the browser's SOP would otherwise block
- **Frontend/backend split**: naturally arises from differences in deployment, scaling, organizational structure, and caching strategy — CORS is just a side effect of it
- **Proxy vs. BFF**: both unify origins to sidestep CORS, but a proxy only forwards requests, while a BFF aggregates and transforms responses from multiple APIs
