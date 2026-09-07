# Part 2 map — Spring (two topics per weekday)

**Day order (from Fri 2026-09-04):** LC first ([part1-map.md](part1-map.md)). Then this app. Then Part 3.

Anton asked (2026-09-02): a day is **two map topics**, not one small lab. Friday = **one** long board. Sat/Sun **off**.

If a day feels thin, **pull the next map day’s Spring into today**. Do not invent extra hover on the same code.

---

## Week 3 — concurrency

| Day | Topics (2, or Fri = 1 board) | Status |
|---|---|---|
| Mon | Row lock on `book()` | Done |
| Tue | `@Version` lab (`findById`) | Done |
| Wed 2026-09-02 | Two-thread test **+** `findByIdForUpdate` back on `book()` **+** title `PATCH` | Done |
| Thu 2026-09-03 | **`WebClient` bean** + **Booking calls Event over HTTP** (next unused code pair; do not empty the day) | Done |
| Fri | Long HLD board (last seat 1 / 10 / 100) | Done |

---

## Week 4 — split + WebClient

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Booking calls Event over HTTP | `WebClient` bean | *(pulled to W3 Thu)* |
| Tue | Third service (auth/users) | Correlation-id header | **Done W4 Day 1 (2026-09-07)** |
| Wed | Downstream 4xx/5xx mapping | Client timeout |
| Thu | Remaining split glue | One integration test for the call |
| Fri | HLD of the three boxes | — |

W4 Mon topics were built on **W3 Thu**. Start Week 4 on **Tue** (third service + correlation-id), or say `Start Week 4 Day 1` meaning those.

---

## Week 5 — gateway + resilience + Docker

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Gateway routes | First service behind it |
| Tue | Resilience4j timeout | Retry (which calls, which not) |
| Wed | Circuit breaker on the hot call | Fallback status |
| Thu | Docker Compose for the set | One health check |
| Fri | HLD traffic through gateway | — |

---

## Week 6 — async + observability

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Async event | Outbox idea in code |
| Tue | Notification send | Actuator health |
| Wed | Metrics on `book()` | One dashboard query |
| Thu | Remaining async glue | Test |
| Fri | HLD | — |

Part 3 from this week: LLD class-design drills start.

---

## Week 7 — polish + performance

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | N+1 fix | Index on hot FK |
| Tue | Security pass (secrets/CORS) | README |
| Wed | One perf check | Leftover polish |
| Thu | Remaining polish | Test |
| Fri | HLD/recap | — |

---

## Week 8 — interview ready

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Demo path | Architecture recap notes |
| Tue | Weak-topic drill | One fix |
| Wed | Mock prep | Leftover |
| Thu | Dry-run answers | Polish |
| Fri | Long mock (HLD + LLD) | — |
