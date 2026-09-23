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

**Calendar:** Part 1 + Part 2 **closed**. Part 3 **open** (carry). **Next weekday:** Week 6 Day 3 — carried Factory + library LLD, then metrics on `book()` + dashboard query. *(Day 3 closed — see below.)*

---

## Week 6 Day 3 — tree fill + metrics

**Date:** 2026-09-23 (Wed)

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LC-Practice | **#199** overtime (coach filled). **#100** overtime (he coded). **#112** overtime (coach filled). **Coding done.** |
| **Part 1 Design (LC-SD)** | Course card talk | **Chapter 9** batching / timeout (CDN skip). **Done.** |
| **Part 2** | Spring | Metrics on `book()` + one dashboard query. **Done.** |
| **Part 3** | OOP + LLD | Carried Factory `@Bean` + library LLD. **Done.** (Anton asked Part 2 before Part 3.) |

Extra detail: [LC-Practice `notes/week-06-day-03.md`](https://github.com/antondahdal/LC-Practice/blob/master/notes/week-06-day-03.md).

---

### Part 2 — Metrics on `book()`

**Goal:** Count successful tickets. Ops can see load without reading logs. Health is not this number.

`MeterRegistry` is a Spring bean (comes with Actuator). Constructor DI into `BookingServiceImpl`. After a successful `save`, `counter("bookings.created").increment()`.

Event **409** throws in `reserveSeats` — never reaches the increment. Do not bump on a sold-out path.

**60-sec:** Counter = how many times `book()` finished. Sold out → no bump. Not a health probe.

**Weak:** Increment before `reserveSeats`. Health **200** means someone booked. Bump on every POST even when Event fails.

---

### Part 2 — One dashboard query

**Goal:** Read that counter once via Actuator.

`management.endpoints.web.exposure.include=health,metrics`. GET `/actuator/metrics/bookings.created` (JWT required — only `/actuator/health/**` is public). After one successful book, `COUNT` = `1.0`.

Lab note: live book hit **500** while `@TimeLimiter` ran `EventClient.reserveSeats` on another thread — `RequestContextHolder` null. Commented TimeLimiter for the demo, then put it back. Proper fix (copy request context) is not today’s topic. PENDING poller stays Thu.

**60-sec:** Expose metrics. Hit the named counter. Health **200** with count **0** means nobody booked yet.

**Weak:** Metrics path is a ticket. Forget to expose `metrics`. Empty Bearer → **403** (same as missing `$ATT`).

---

### Part 3 — Factory as `@Bean`

`@Bean` on a method in `@Configuration`: Spring calls the method at startup and registers the **return value** as a bean. The method is the factory; the object is what others inject.

Spring Security finds your chain by **type** (`SecurityFilterChain`), not by the class name `SecurityConfig`. Scan finds `@Configuration`; Security asks the container for that type on each request. You do not call `filterChain` from a controller.

`@Service` on `SecurityFilterChain` does not work — it is not your class to annotate. Factory vs Singleton: factory = who creates; singleton = how many (default one shared instance).

---

### Part 3 — LLD library

Whiteboard. Not this app. Do not rerun parking lot.

**Actors:** Member (Guest), Librarian.

**Classes:** Book, Shelf, Library, Member, Librarian, Loan.

**Loan fields:** Book, Member, from, till. (Optional later: returned.)

**Loan methods:** `returnBook`, `extendTill`, `isOverdue`. Fields at create time are not methods.

**Has-a:** Loan has-a Book (not is-a).

**Checkout:** `checkout(member, book)` on **Librarian** (desk admin). Loan does not loan itself. Same idea as `initiateTicket` on `ParkingLot`.

Next LLD product: **hotel rooms**. Do not rerun this library sketch.

**Calendar:** Day 3 **closed**. **Next weekday:** Week 6 Day 4 — PENDING poller + test.
