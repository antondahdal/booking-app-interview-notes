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

---

## Week 6 Day 4 — tree fill + rate limit recap

**Date:** 2026-09-24 (Thu)

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LC-Practice | **#236** LCA coach filled (before 0). **#101** Symmetric overtime (coach filled). Third (#637) skipped. **Coding done.** |
| **Part 1 Design (LC-SD)** | Course card talk | **Chapter 11** rate limit drill. **Done.** |
| **Part 2** | Spring | PENDING poller + test. **Done.** |
| **Part 3** | OOP + LLD | Overloading vs overriding + hotel rooms LLD. **Done.** |

Extra detail: [LC-Practice `notes/week-06-day-04.md`](https://github.com/antondahdal/LC-Practice/blob/master/notes/week-06-day-04.md). DFS orders look-up: [`notes/dfs-orders.md`](https://github.com/antondahdal/LC-Practice/blob/master/notes/dfs-orders.md).

---

### Part 1 coding — trees

**#236 LCA:** post-order. Each call returns `p`/`q`/answer or null. Both sides non-null → this node. Else pass the non-null side up.

**#101 Symmetric:** two queues in lockstep. Check the **polled pair** (both null → continue; one null / values differ → false). Offer outer (`a.left` / `b.right`) then inner (`a.right` / `b.left`).

**Weak:** pre vs in vs post mixed (first said pre-order for LCA). "How do p and q connect" — it is two non-null returns at one node. Symmetric: pre-checking kids instead of the polled pair (same as Day 3 Same Tree); right side offered in Same Tree order, not mirror.

---

### Part 1 Design — Chapter 11 rate limit drill

Retry already recapped Tue. W4 Day 1 Ch 11 was explained, not drilled.

Prompt: bot on one account fires POST Book 50/s for event 7, seats left.

Anton: count per save / commit; 4XX too many; separate error response.

Right: 4XX family (**429**). Separate response so it is not sold out (**409**).

Wrong: per commit is too late — flood already hit `book()`, Event, DB. Failed Books (409) never commit, so never counted. Where was vague.

Fix: count per caller (JWT user, else IP) per window. Check before the handler (gateway / filter). Shared counter (Redis) so N instances ≠ N × cap.

**Trap:** count inside `book()` or on commit. 429 must never look like 409 or a ticket.

**60-sec:** Rate-limit per user per window with a shared counter at the edge, before `book()`. Over the cap is 429. Sold out stays 409.

**Calendar:** Part 1 **closed**. Part 2 (PENDING poller + test) and Part 3 (hotel rooms LLD) **open**.

---

### Part 2 — PENDING poller

**Goal:** Listener shout dies on crash/restart. Row stays `PENDING`. Something must pick it up without an event.

`findByStatus(String)` on the outbox repo (derived query, no `@Query`). `@EnableScheduling` on the app. `OutboxPoller` (`@Component`, `events`): `@Scheduled(fixedDelay = 10000)` → each `PENDING` row: `send(bookingId)`, then `SENT`, `save`.

`fixedDelay` = 10 s after the previous run **finishes**. `fixedRate` = every 10 s from start. Method is `void`, no args (no caller).

Crash after `send`, before `SENT` → still `PENDING` → next run sends again → user gets it **twice**. At-least-once: never lost, may duplicate.

**Bugs he hit:** `@Scheduled` with no timing (startup fails). `"Pending"` ≠ `"PENDING"`. No `send` call. Outbox `getId()` passed where booking id is needed.

**60-sec:** Poller scans `PENDING` on a timer, sends, then marks `SENT`. Crash between → sent twice, never lost.

**Weak:** Answered “row gets reprocessed” but missed “user gets it twice” until told.

---

### Part 2 — Test

`OutboxPollerTest` — plain Mockito (`@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`), no Spring context. Coach wrote it (Anton asked). Row outbox id **7**, booking id **42**. Call `checkMail()`. Verify `send(42)`, status `SENT`, `save(message)`. Passes.

Why two numbers: same value would hide `getId()` vs `getBookingId()`. Wrong id on `send` → the `verify(send(42))` line fails (lookup still uses booking id). Real cost: wrong/no user told, row marked `SENT`, never retried.

**Weak:** Said the lookup would fail — it is the `send` verify.

**Command:** `.\mvnw.cmd test -Dtest=OutboxPollerTest`

---

### Part 3 — OOP: overloading vs overriding

Leftover (OOP list empty). Overload = same **name**, different params, can live in one class. Override = subclass replaces inherited method, needs a parent, may call `super`.

**Weak:** said overload = “same signature.” Signature = name + params.

---

### Part 3 — LLD hotel rooms

**Actors:** Guest, Manager/Admin (prices, rooms), Receptionist, Housekeeping. “Owner” / “Workers” too vague at first.

**Classes:** Guest, Room (floor as field), Hotel, Worker → `Receptionist`, `Housekeeper` subclasses (no enum on top), **Reservation** (missed first — found with Ticket/Loan hint), Payment.

**Reservation fields:** Guest, Room, from, till, price, Payment (amount, partial). **Methods:** `pay`, `changeRoom`, `changeDates`, `cancel` (blocked < 72 h). Guest changes on the reservation; Hotel/service decides availability.

**Status:** one enum, not booleans. `BOOKED → CHECKED_IN | CANCELLED (>72 h)`. `CHECKED_IN → CHECKED_OUT`. `CANCELLED`, `CHECKED_OUT` terminal.

**Owner:** `Receptionist.checkIn(reservation)`. Receptionist is-a Worker. Reservation has-a Guest / Room / Payment.

**Change (room types):** price calc in `Room`. Only numbers differ → multiplier field via constructor. Behavior differs (Suite breakfast) → subclass + **override** `calculatePrice`. Not overloading.

**Sequence:** browse → pick Deluxe + dates → price → Book → BookingService validates → `BOOKED` → id to guest. Availability checked on browse and again on book.

**Weak:** booleans for arrived/left/cancelled. “Overloading” for per-type price. “Check twice = no race” — second check needs a lock (`FOR UPDATE` / unique room+date) or two guests both pass.

Next LLD: **W7 critique format** (food-delivery order Mon). Do not rerun hotel.

**Calendar:** Day 4 **closed**. **Next weekday:** Week 6 Day 5 (Fri) — HLD async.

---

## Week 6 Day 5 — HLD async

**Date:** 2026-09-25 (Fri)

| Name | What it is | Today |
|---|---|---|
| **Part 3** | HLD + 2 min OOP | Async HLD board + Builder. **Done.** |

---

### Part 3 — HLD async

**Boxes.** Book path: Phone → Gateway → Booking (JWT filter, `AuthClient`, `EventClient` reserve, ticket `save`, metric, `populateMessageAndSave`, `publishEvent`) → **201**. Notify path: `@Async` listener (fast path) + `OutboxPoller` (backup) → `BookingNotifier.send` → `SENT`. Store: outbox table in Booking's DB.

**Truth / commits.** Seats = Event's DB, committed when the HTTP call returns. Ticket + outbox row = Booking's DB, **one** commit. No transaction across both.

**Event took seats, Booking `save` fails.** Booking rolls back (no ticket, no outbox). Phone **500**. Seats stuck in Event, nobody owns them. Fix = **compensating** release via Event's API (saga). Release can fail → durable `RELEASE_PENDING` row + poller. Retry is at-least-once → Event must be **idempotent**: key on the attempt id (reservation id, unique), not `eventId`. Only Event can dedupe — Booking never saw the lost reply.

Alternative: **hold + confirm** — Event writes `HELD` + `expiresAt`, Booking confirms after commit, Event `@Scheduled` expires unconfirmed holds. Anton asked to build it → **W7 Thu Part 2**.

**10× + mail down 1 h.** Book unaffected, **201** keeps coming, ~60k `PENDING` pile up. Fetch in batches, backoff/retry count.

**3 Booking pods, same poller.** All read the same `PENDING` → mail ×3. Plain `FOR UPDATE` → others wait (no dupes, no speedup). `SKIP LOCKED` → each pod takes different rows. Do not hold the lock during `send()` (Week 3 pay-after-lock rule — holds a pool connection that `book()` needs). **Claim:** `PENDING → SENDING` + commit, send, `SENT` in a short tx. Crash while `SENDING` → `claimedAt` + timeout returns rows to the scan (slow pod may still send → at-least-once).

**60-sec:** Gateway → Booking: Auth check, seats in Event (its own commit), ticket + `PENDING` outbox in one Booking commit, **201**. `@Async` listener sends after commit; poller retries `PENDING`. Mail down never blocks Book. At-least-once; claim rows across pods. Booking fails after Event took seats → idempotent release keyed by reservation id.

**Weak:** `publishEvent` "populates the DB" (it is `populateMessageAndSave`). Booking-side status table to stop a double release (reply was lost — only Event knows). "Batch it" / "slower" for lock held during `send()` — needed the connection-pool cost and the claim step. "Same thread" for three pods.

---

### Part 3 — OOP: Builder (leftover, list empty)

Long constructor: same-type params (`eventId`, `userId`) swap and still compile. Setters: half-built object, forgotten field = `null`, no `final`. Builder: named steps, `build()` checks required fields once, immutable result.

**Weak:** setters half not known.

**Calendar:** Week 6 **closed**. **Next weekday:** Week 7 Day 1 (Mon) — see [week-07.md](week-07.md).
