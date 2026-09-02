---
title: "Planning a 'N Stops Away' Bus Alert App: From First Draft to a Validation-First Design"
date: 2026-09-02 00:00:00 +0900
categories: [Side Project, Backend]
tags: [bus-api, public-data, push-notification, nodejs, mysql, flutter, system-design]
---

This post records how a one-line app idea — *"tell me when my bus is 3 stops away"* — turned into a design document over three revisions. It covers what the public bus APIs actually provide, the core detection logic, the feedback that reshaped the plan, and why the tech stack ended up as Node.js + MySQL on a single VM.

## 1. The idea

Pick a region (Seoul or Gyeonggi) and a bus number, load the route's stop list, mark "my stop", and get a push notification when the bus reaches the stop **N stops before** mine (default N = 3). That's it. Existing bus apps show "arrives in 4 min," but you have to keep opening them; this app should let you leave the house without looking at your phone.

It's a personal project and the first user is me, so market analysis and differentiation were dropped early. The only success criterion: *does the alert arrive at the right moment every weekday morning?*

## 2. What the public APIs give you

Both regions publish free REST APIs through the Korean public data portal (data.go.kr). A dev-tier key allows **1,000 calls/day** per service; production tier requires registering a use case and requesting more.

| Need | Seoul (TOPIS, `ws.bus.go.kr/api/rest/`) | Gyeonggi (GBIS, `apis.data.go.kr/6410000/`, v2) |
|---|---|---|
| Search route by number | `busRouteInfo/getBusRouteList` → `busRouteId` | `busrouteservice/v2/getBusRouteListv2` → `routeId` |
| Stops on a route | `busRouteInfo/getStaionByRoute` (sic) → `seq`, `transYn` (turnaround) | `busrouteservice/v2/getBusRouteStationListv2` → `stationSeq`, `turnYn` |
| Live vehicle positions | `buspos/getBusPosByRtid` → `vehId`, **`sectOrd`**, `stopFlag`, `dataTm` | `buslocationservice/v2/getBusLocationListv2` → `vehId`, **`stationSeq`**, `stateCd`, `queryTime` |
| Arrival info at a stop | `arrive/getArrInfoByRoute` → `arrmsg1`, `exps1`, `sectOrd1`, `staOrd` | `busarrivalservice/v2/getBusArrivalItemv2` → `predictTime1`, `locationNo1` |

The key observation: each vehicle comes back with a **stop sequence number** (`sectOrd` / `stationSeq`). If my stop is sequence 20 and N = 3, the trigger is "a vehicle's sequence just reached 17." That's a much more stable signal than a minutes-based ETA.

Seoul's API is XML-first; Gyeonggi's takes `format=json`. Field names differ everywhere, so the two are wrapped behind a single `BusDataProvider` interface and the rest of the system never sees region-specific fields.

## 3. Core design decisions (v1)

- **The server does the watching, not the app.** iOS effectively forbids background polling, and it drains battery on Android. The backend polls vehicle positions per route and sends FCM/APNs pushes; the app only registers subscriptions and receives notifications.
- **Poll per route, not per user.** Subscriptions are grouped by `routeId`, so 100 users on the same route cost one API call per poll.
- **Trigger on sequence crossing.** Notify when `prevSeq < targetSeq − N ≤ curSeq < targetSeq`, once per vehicle per subscription. Using `≤` instead of `==` handles the case where a bus skips two stops between polls.

## 4. The feedback that reshaped v2

A review of the first draft pushed the priority from "build features" to "prove the core assumption." The main points:

1. **Nobody has verified what `sectOrd` and `stationSeq` actually mean.** Do they increment exactly once per stop? Does the value flip at arrival or departure? Do they reset at the turnaround? Does a vehicle keep its `vehId` after dropping out of the feed? Treating the two fields as identical because the names look similar is a gamble.
2. **1,000 calls/day is a hard wall for development.** One route polled every 30 s for 18 h is 2,160 calls. The detection logic must be testable *without* the live API.
3. **A 20-second polling interval had no justification.** Real latency is a chain: API refresh → poll → response → server → FCM → device.
4. **"95% accuracy" wasn't defined.** What counts as success when the bus skips the trigger stop between polls, or the API returns a bogus value?
5. **`vehId → lastSeq` in Redis is too thin.** Vehicles vanish, reappear, and turn around; that needs explicit states.
6. The live-position screen is not needed for MVP; a weekday/time-window schedule is.
7. A bare `X-Device-Id` header isn't authentication.

## 5. What v2 / v3 look like

### Phase 0: validate the data before writing product code

Record real API responses (positions + arrival info) for one Seoul and one Gyeonggi route during rush hour, 30 s apart, into JSONL. That's ~1,900 calls, within two dev keys. Then answer eight questions (V-1…V-8) about sequence semantics, turnarounds, dropouts, and skip frequency. Every judgment constant — `TRIGGER_OFFSET`, `MAX_MISSING_DURATION`, `MAX_SEQ_JUMP`, `RESET_SEQ_DROP`, `POLL_INTERVAL` — is a config value fixed only after this step.

### Record / Replay providers

```
[live API] ──record──▶ fixtures/*.jsonl ──replay──▶ ReplayProvider
                                                        │
                                          Poller → validate() → decide() → Notifier(mock)
```

`recordingProvider` wraps a real client and saves timestamped responses; `replayProvider` plays them back at 1x/10x/instant. Unit tests, replay tests, and a tiny contract test cover the logic; the live API is only touched in field tests.

### Observation validation, then a state machine

Before any state change, each observation is checked: out-of-range sequence, jump larger than `MAX_SEQ_JUMP`, small backward drift, stale `sourceTime`, or missing fields → keep state, log `GHOST`. A large backward drop is not invalid — it's a turnaround/restart signal.

Valid observations feed a pure function `decide(subscription, vehicleState, observation, now, config)` with per-(subscription, vehicle) states:

```
NOT_SEEN → BEFORE_TRIGGER → TRIGGERED (push sent here, once) → PASSED
   ▲                                                              │
   └──── now − lastSeenAt > MAX_MISSING_DURATION, or seq drop ────┘
```

Expiry is **time-based** (`lastSeenAt`), not "N missed polls," so changing the poll interval doesn't silently change reset behaviour. An 18-row test-case table (T-01…T-18) maps 1:1 to unit tests.

### Accuracy, defined

An alert succeeds if it fired within `[N−1, N+1]` stops, arrived before the server observed the bus at my stop, and was the only alert for that vehicle. Failures are classified: `LATE`, `EARLY`, `SKIPPED`, `DUPLICATE`, `GHOST`, `MISSED_END`. The log stores `observed_arrival_at` (when the *server saw* the bus reach the stop — not the true arrival) plus an optional hand-entered `manual_arrival_at` from field tests.

### Position-based vs. arrival-info-based detection

Both approaches are kept on the table and compared on the same recorded data by accuracy, call volume, latency, and complexity. If the sequence assumptions from Phase 0 hold, position-based wins (one call per route); otherwise the arrival-info API's `locationNo` ("n stops away") becomes the primary signal.

## 6. Stack: why not Spring Boot / PostgreSQL / AWS ECS?

The first draft assumed a team-style stack. For a one-person, I/O-bound service the answers changed:

| Area | Chosen | Why |
|---|---|---|
| Backend | **Node.js 22 + Express (JavaScript)** | The server only polls HTTP, parses XML/JSON, touches Redis, and sends FCM. All I/O-bound; Spring's strengths (transactions, JPA, large-team structure) go unused. `firebase-admin` treats Node as a first-class SDK. Familiarity wins. |
| DB | **MySQL 8** | Three tables and one user; none of Postgres's advantages (JSONB indexing, PostGIS) would be felt. Gotchas noted: `utf8mb4`, `JSON` column for schedules, and **store every `DATETIME` in UTC** — MySQL doesn't keep time zones, and this project measures latency in seconds. |
| Infra | **Single VM + Docker Compose** (app, MySQL, Redis, Caddy) | The one hard requirement is that the poller never sleeps during the alert window. Free-tier PaaS suspends idle apps; serverless schedulers bottom out at 1 min and fight short-interval state tracking; ECS/K8s is overkill for one user; a home server ties uptime to a residential router. A $0–10/month VM with `restart: unless-stopped`, a daily `mysqldump` cron, and SSH-based GitHub Actions deploy is the simplest thing that always runs. |

Because the code is plain JavaScript, state and action names live in `Object.freeze` constants and the T-01…T-18 table is transcribed verbatim into tests to catch typos the type system would otherwise catch.

## 7. Build order (no dates)

0. Data validation (Phase 0) → constants and A/B decision fixed
1. `validate()` + `decide()` with unit and replay tests → 100% match on recorded data
2. Minimal server: adapters, subscription CRUD, token auth, poller (schedule window only), FCM → a `curl`-registered subscription pushes to my phone
3. Flutter MVP: six screens, reports push receipt time
4. Deploy: VM, Compose, HTTPS, backups
5. Two weeks of real commutes → success rate ≥ 95%, median latency ≤ 30 s
6. Schedule feature, failure alerts, production API tier
7. Everything else (live map, low-floor filter, other regions)

## TL;DR

- Seoul and Gyeonggi bus APIs both expose a per-vehicle **stop sequence number**; "N stops away" is a sequence comparison, not an ETA guess
- Watch from the **server**, push to the phone — mobile background polling isn't viable
- The dev-tier limit (1,000 calls/day) means the detection logic must run on **recorded, replayed data**, not the live API
- Don't assume `sectOrd` ≡ `stationSeq` — **validate with real vehicle traces first**, then fix the constants
- Validate observations before the state machine; expire vehicle state by **time**, not poll count; name the recorded arrival `observed_arrival_at` because that's what it is
- For a one-person I/O-bound service: **Node.js + Express, MySQL 8 (UTC everywhere), one VM with Docker Compose** — the cheapest thing that never sleeps
