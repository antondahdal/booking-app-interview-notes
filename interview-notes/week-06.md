# Interview notes — Week 6

Async + observability: events, notifications, actuator, metrics.

**Packed:** **two** Part 1 topics each Mon–Thu. Friday = one HLD. Full calendar: [part1-map.md](part1-map.md).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Async event | Outbox idea in code |
| Tue | Notification send | Actuator health |
| Wed | Metrics on `book()` | One dashboard query |
| Thu | Remaining async glue | Test |
| Fri | HLD | — |

**Part 3 from this week:** LLD class-design drills start (whiteboard classes / fields / method signatures for a small product — not implement). Weeks 1–5 stay API + this-app OOP + Friday HLD.

---

## Week 6 Day 1 — trees + async event + outbox

**Date:** 2026-09-21 (Mon)

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LC-Practice | **#104**, **#226** on-time. **#102** overtime (coach filled). **Done.** |
| **Part 1 Design (LC-SD)** | Course card talk | **Chapter 5** — queue + at-least-once. **Done.** |
| **Part 2** | Spring | Async event + outbox row. **Done.** |
| **Part 3** | OOP + this-app design + LLD | Observer + outbox board + parking-lot first pass. **Done.** |

---

### Part 2 — Async event

**Goal:** `book()` does not send mail. After `save`, publish. Listener after commit, then `@Async` so **201** is not the print.

`BookingCreatedEvent` is not `@Entity` and not in `model`. It is the in-memory shout (`bookingId`).

`ApplicationEventPublisher` is a Spring bean. `publishEvent` runs methods in **this** JVM that want that type.

`sendEmail()` inside `book()` keeps `@Transactional` open and delays **201**. Event already took seats (`reserveSeats` before `save`).

`@TransactionalEventListener(phase = AFTER_COMMIT)`: DB commit, then listener. Default is still the **same thread**, so a slow print still sits **between** commit and **201**.

`@EnableAsync` on the app (next to `@EnableCaching`). `@Async` on the **listener** method, not on `book()`. `book()` must take seats, save, return the id.

**60-sec:** After commit, mail work is `@Async`. Do not put `@Async` on `book()`.

**Weak:** “201 first” without `@Async`. `@Async` on `book()`. `AFTER_COMMIT` mixed with “outbox wrote the row.”

---

### Part 2 — Outbox in code

**Goal:** Same commit as the ticket, a `PENDING` row. Kill after **201** must not lose the “send later” work.

`OutboxMessage` **is** an entity (`bookingId`, `status`). Repo in Booking, not a `WebClient` client — a hop cannot join this transaction.

`book()`: `save` ticket, then `populateMessageAndSave` (`PENDING`), then `publishEvent`. Listener: print, `findByBookingId`, `"SENT"`. Not Event’s `FOR UPDATE` finder. JPQL field is `bookingId`.

**201** = ticket + `PENDING` committed. Print does not have to have run. `AFTER_COMMIT` starts the listener; it does not insert the row.

At-least-once: die after print, before `"SENT"` → one row still `PENDING`. A worker that scans `PENDING` can print **again**. Unique on `bookingId` blocks a **second row**, not a retry of that row. Not a second Book. This app’s listener does **not** wake on restart (shout died). Poller = Thu.

**60-sec:** Commit ticket and outbox together, then **201**. Same `PENDING` row may run twice.

**Weak:** Event writes the ticket. Unique = no second print. `populateMessageAndSave` is `@Async`. Second Book as the duplicate.

---

### Part 3 — Observer

`publishEvent` does not look up `BookingCreatedListener` by name. Startup: `@Component` + `@TransactionalEventListener` + argument `BookingCreatedEvent`. `book()` does not name the listener.

---

### Part 3 — Outbox board

Ticket = Booking `save`. Seats = Event. `PENDING` = `populateMessageAndSave` in `book()`. Print = listener on another thread.

**201** ≠ print already ran. Event **409** throws in `reserveSeats` — never reaches save/outbox. No transactional story needed.

---

### Part 3 — LLD parking lot (first pass)

LLD = Low-Level Design (classes / fields / methods). Whiteboard. Not this app.

**Actors:** driver. **Classes:** Car, CarSpot, Driver, ParkingLot, Ticket.

**Ticket fields:** id, from, till, price, Car, paid.

**Methods:** `payTicket`, `ExtendTill` on Ticket. `initiateTicket` belongs on `ParkingLot`.

**Has-a:** Ticket has-a Car (not is-a). Both directions optional.

Next LLD product: **library**. Do not rerun this parking-lot sketch.

**Calendar:** Day 1 **closed**. **Next weekday:** Week 6 Day 2 — #98 / #230, notification send, Actuator, library LLD.

---

## Week 6 Day 2 — trees + notification send + Actuator

**Date:** 2026-09-22 (Tue)

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LC-Practice | **#98**, **#230** overtime (he coded). Third Easy skipped. **Done.** |
| **Part 1 Design (LC-SD)** | Course card talk | **Chapter 8** timeout / retry / click id recap. **Done.** |
| **Part 2** | Spring | Notification send + Actuator probes. **Done.** |
| **Part 3** | OOP + LLD | **Skipped** (Anton). Carry: Factory `@Bean` + library LLD. |

---

### Part 2 — Notification send

**Goal:** Listener does not own the “tell the user” print. A `BookingNotifier` does. Mark outbox `SENT` only after `send` returns.

`BookingNotifier` is `@Service`. Constructor DI into `BookingCreatedListener`. `EventIdPublisher` calls `send`, then updates the row.

If `send` throws: ticket already committed (listener is after commit). Outbox row stays `PENDING`.

**60-sec:** After commit, notifier tells; then `SENT`. Fail tell → ticket stays, row stays `PENDING`.

**Weak:** `send` inside `book()`. Mark `SENT` before `send`. Print still on the listener.

---

### Part 2 — Actuator health

**Goal:** Live ≠ ready. Ops probe is not `book()`.

`management.endpoint.health.probes.enabled=true`. Security: `/actuator/health/**` public (exact `/actuator/health` is not enough for liveness/readiness paths).

DB down, JVM up → **alive**, **not ready**. Health **200** is not a ticket.

**60-sec:** Liveness = process up. Readiness = may send Book (DB). Probes on; health path open under `/**`.

**Weak:** Health **200** → ticket. Kill / restart on failed readiness only as if it were liveness. Exact matcher blocks probe URLs.

---

### Part 3 — skipped

Factory `@Bean` + library LLD **not run**. Do not skip the slot next weekday — run these first, then Wed map items.

**Calendar:** Part 1 + Part 2 **closed**. Part 3 **open** (carry). **Next weekday:** Week 6 Day 3 — carried Factory + library LLD, then metrics on `book()` + dashboard query.
