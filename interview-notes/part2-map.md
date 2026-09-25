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
| Wed | Downstream 4xx/5xx mapping | Client timeout | **Done W4 Day 3 (2026-09-09)** |
| Thu | Remaining split glue | One integration test for the call | **Done W4 Day 4 (2026-09-10)** |
| Fri | HLD of the three boxes | — | **Done W4 Day 5 (2026-09-11)** |

W4 Mon topics were built on **W3 Thu**. Start Week 4 on **Tue** (third service + correlation-id), or say `Start Week 4 Day 1` meaning those.

---

## Week 5 — gateway + resilience + Docker

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Gateway routes | First service behind it | **Done W5 Day 1 (2026-09-14)** |
| Tue | Resilience4j timeout | Retry (which calls, which not) | **Done W5 Day 2 (2026-09-15)** + cache lab |
| Wed | Circuit breaker on the hot call | Fallback status | **Done W5 Day 3 (2026-09-16)** |
| Thu | Docker Compose for the set | One health check | **Done W5 Day 4 (2026-09-17)** |
| Fri | HLD traffic through gateway | — | **Done W5 Day 5 (2026-09-18)** |

**Spring Cache:** talk + `@Cacheable` / `@CacheEvict` **coded W5 Day 2**. Redis not wired (in-memory today). Interview store is still Redis. Notes: [week-05.md](week-05.md).

---

## Week 6 — async + observability

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Async event | Outbox idea in code | **Done W6 Day 1 (2026-09-21)** |
| Tue | Notification send | Actuator health | **Done W6 Day 2 (2026-09-22)** |
| Wed | Metrics on `book()` | One dashboard query | **Done W6 Day 3 (2026-09-23)** |
| Thu | Remaining async glue | Test | **Done W6 Day 4 (2026-09-24)** — PENDING poller + Mockito test |
| Fri | HLD | — | **Done W6 Day 5 (2026-09-25)** — async board |

Part 3 from this week: LLD class-design drills start.

---

## Week 7 — polish + performance

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | N+1 fix | Index on hot FK |
| Tue | Security pass (secrets/CORS) | README |
| Wed | One perf check **(+ Spring Cache lab if not done)** | Leftover polish |
| Thu | Seat hold + confirm on Event (TTL expiry). Anton asked W6 Fri 2026-09-25 | Test (expiry job) |
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

---

## Week 9 — Hibernate + Docker + Kubernetes + Spring AI (Anton 2026-09-25)

Details and traps: [week-09.md](week-09.md). Compose and health vs ready were W5 Thu — build on them, do not rerun.

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | H1 Hibernate persistence context (lifecycle, first-level cache, dirty checking, flush, `merge`) | H2 Lazy loading + proxies (`getReferenceById`, `LazyInitializationException`, open-in-view off) |
| Tue | H3 N+1 deep (fetch join / `@EntityGraph` / `@BatchSize`, two bags, fetch join + paging, prove query count) | H4 Transactions (propagation, `readOnly`, rollback, isolation) |
| Wed | H5 Mapping / cascade / IDs / batch inserts / locking | Docker: multi-stage, layers, JVM memory in a container |

Hibernate is **five topics** (Anton 2026-09-25: go deeper). SQL logging on for every Hibernate lab.
| Thu | Kubernetes `Deployment` + `Service` + `ConfigMap` / `Secret` (kind or minikube) | Probes on Actuator, requests / limits, rolling update, HPA |
| Fri | Spring AI `ChatClient` endpoint | One read-only tool (no booking tool) |
