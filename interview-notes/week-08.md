# Interview notes — Week 8

Interview ready: demo, architecture recap, mocks.

**Packed:** **two** Part 1 topics each Mon–Thu. Friday = long mock. Full calendar: [part1-map.md](part1-map.md).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Demo path | Architecture recap notes |
| Tue | Weak-topic drill | One fix |
| Wed | Mock prep | Leftover |
| Thu | Dry-run answers | Polish |
| Fri | Long mock (HLD + LLD) | — |

**Part 3:** one mock includes a 15–20 min LLD (classes / fields / methods for a small product). Plus demo + HLD recap.

---

## Week 8 Day 1 (Mon 2026-10-05)

### Part 2 — W7 leftovers (seat hold + confirm), closed

- `HoldExpiryJob` (Event side): `@Component`, `@Scheduled(fixedDelay = 5000)`. Finder `findByStatusAndExpiresAtBefore(HELD, now())`, then `eventService.expireHold(id)` per hold, through the bean.
- `Booking.status` (`BookingStatus`, `@Enumerated(STRING)`, not null). `writeBook` sets CONFIRMED.
- `OutboxPoller` 409 catch (`HoldDataExceedTimeException`): booking → CANCELLED + save, **then** row SENT. `DownstreamServiceException` stays PENDING (retry).
- Tests (coach wrote, Anton asked): `HoldExpiryJobTest` (two overdue holds → two `expireHold`; none → never). `OutboxPollerTest` fixed (did not compile) + new test: 409 → booking CANCELLED, row SENT. All pass.

**Checks:**
- Job calls `expireHold` per hold instead of `@Transactional` on the job: Anton — one failure must not roll back the 50 that worked (right). Plus: each row lock is held short.
- Row SENT before the cancel is saved, crash in between: Anton — row updated, ticket not cancelled (half). Other half: SENT is never polled again → ticket stays CONFIRMED forever with no seats. Fix: cancel first; a repeat cancel is harmless.

**Code slips:** `now().minusMinutes(10)` in the job (counts the 10 minutes twice — the deadline is already in `expiresAt`). Forgot CONFIRMED in `writeBook` (with `nullable = false` every Book would fail).

### Part 2 — Demo path

> The user logs in through Auth and gets a JWT. Every request goes through the gateway, which routes it, and the service's JWT filter validates the token. When the user taps Book, Booking calls Auth for the user and Event to reserve seats. Event locks the event row, decrements the seats, and saves a hold with a 10-minute deadline in one commit. Booking then saves the ticket plus two outbox rows (notify and confirm hold) in one commit. A poller delivers them: it sends the email and calls Event to confirm the hold, which is safe to repeat. If the confirm never arrives in time, a job on Event expires the hold and gives the seats back. A late confirm gets 409, and Booking cancels the ticket.

**Gaps in first try:** sounded like one app (no hops); said the gateway validates the JWT (it only routes — each service's `JwtAuthenticationFilter` does); "Event marked HELD" (the hold is a separate row); missed "one commit" on Booking; said the poller confirms (Event confirms, poller only calls).

### Part 2 — Architecture recap

| Box | Owns | Called by |
|---|---|---|
| Gateway | nothing — routes | browser |
| Auth | users, roles; issues the JWT | Gateway (login / register), Booking (who is the user) |
| Event | venues, events, seat holds | Gateway (browse, organizer), Booking (reserve seats; confirm hold) |
| Booking | tickets, outbox rows | Gateway (Book) |

Sync vs later: **reserve seats** is inside the user's request; **confirm hold** and email go later through the outbox poller. Only what the user needs for the answer runs while they wait.

**Calendar:** Part 2 **closed**. Part 3 **not run** (carried again). Spring pushed W8 Day 2 morning.

---

## Week 8 Day 2 (Tue 2026-10-06) — short day, job summit

Anton asked: CV drill instead of the normal day. Part 2 (weak-topic drill + one fix) and Part 3 (carried W7 items) **not run** — carry.

### CV drill

Full answers, examples and edits: [cv-prep.md](cv-prep.md).

- Tell me about yourself: first try was a word list. Shape: who → domain → one win → project → what you want.
- Two projects: Telefónica Hispam + customer migration after acquisition. No internal words ("Tef", "DFS").
- Migration story (strong): locked customers → retry table → second job reads the table, re-validates, reprocesses.
- Batch: described the internal batch framework (`init` / mapper / `run` / `finalActivities`, chunks of 100, rollback then one by one). Mapped to Spring Batch words. **Open:** check for `org.springframework.batch` imports; until then say "Spring-based batch framework".
- SQL tuning: tuned with the DBA (AWR report). First answer "add indexes" = the fix, not what you look for → full table scan / cost / skipped index. Second try good (index on the filtered column, PARALLEL for big batch scans).
- SOAP vs REST: the first answer ("SOAP is XML in WebLogic, REST is JSON") was half right. WebLogic is only the server. SOAP is a protocol with a WSDL contract and WS-Security; REST is a style over HTTP. Mixed up WSDL with encryption and with the token twice. Fixed with real XML and HTTP examples and the "menu vs ID card" picture.
- EJB: didn't know what an EJB is. Learned it, then mapped it to his own work: remote EJB APIs that CRM calls, with the client kit as the contract. His own good answer: EJB for internal Amdocs Java-to-Java calls, SOAP for outside partners like banks.
- Booking app: checked the code. It's one Spring Boot app plus a gateway (a modular monolith with service boundaries), not four deployed services. The CV line was reworded, and the list of what's missing to split it (Booking's `@ManyToOne` links to Event and User, one shared DB, one deployable, the gateway routing only Book) is in the notes.

**Carry:** Week 8 Day 2 Part 2 (weak-topic drill and one fix). Part 3's carried Week 7 items (notification-outbox critique, the 429 and queue board, the URL shortener). CV: one high-availability example with DevOps, one root-cause story.

---

## Week 8 Day 3 (Wed 2026-10-07)

Part 2 used the "one fix" + "leftover" slots for the first two items of the split list in [cv-prep.md](cv-prep.md) section 8. Part 3 **not run** (Anton closed after Part 2) — carry.

### Part 2 — Booking keeps ids, not links to Event and User

- `Booking`: `@ManyToOne Event` and `@ManyToOne User` replaced by plain `eventId` and `userId` columns (not null, same column names, so `idx_booking_user` still matches). New `eventTitle` column.
- `BookingWriter.writeBook(resUser, seats, eventRes)`: no more `UserRepository` / `EventRepository`. User id comes from Auth's answer; event id, title and hold id come from Event's `reserveSeats` answer.
- `myBookings()` reads the booking's own `eventTitle`. `@EntityGraph` on `findByUserId` removed (no `event` field left to join).

**Why:** after the split Booking has its own database. It cannot join Event's or User's tables, so a foreign key into them is impossible.

**Questions Anton asked (good ones):**
- *How does it know the relation without the FK?* It doesn't, on purpose. The id is trusted because Event answered `reserveSeats` for it (no event → 404 → no booking row). The check moves from the DB into the flow.
- *How does lazy fetch work now?* It doesn't apply. Lazy is only for a relationship field (a proxy loaded on first touch). A `Long` column loads with the row like `seats`.
- *Inject the clients into `BookingWriter` instead of the repos?* No. `book()` already called them; `writeBook` gets their answers as parameters. HTTP inside `writeBook` would hold a DB connection during the call again (the W7 Day 3 fix).

**Check:** title for "My tickets" — call Event over HTTP, or keep a copy? First answer: HTTP, "because the user might want something else" (missed the cost). After the cost was spelled out: the list shows the copied title, clicking a ticket calls Event for full details. Right.
- HTTP per ticket: 10 tickets = 10 calls, and Event down = "My tickets" down.
- Copy: no calls, works with Event down, but stale if the organizer renames the event.
- Interview line: *"The ticket keeps a snapshot of what I bought. The detail page asks the owner."*

**Code slips:** kept `@ManyToOne` + `@JoinColumn` on the new `Long` fields (startup would fail — those annotations mean "link to an entity"). Passed the same data twice to `writeBook` (`id` + `title` next to `eventRes`).

### Part 2 — Gateway routes every public endpoint

Coach wrote this one (Anton: "2 u do it").

- `GatewayRouteConfig` replaces `BookingRouteConfig`. Three groups, each with its own URI property:
  - Auth: `/api/auth/**`, `/api/users/**`
  - Event: `/api/events`, `/api/events/{id}`, `/api/venues/**` — exact paths, no `/api/events/**` wildcard
  - Booking: POST `/api/events/{eventId}/bookings`, `/api/bookings/**`
- Left off on purpose: Event's `seat-reservations` and `holds/{holdId}/confirm` (only Booking calls them).
- `auth.service.uri` / `event.service.uri` = `localhost:8080` today (one app). After the split only these values change. Compose passes `AUTH_SERVICE_URI` / `EVENT_SERVICE_URI` too (inside a container `localhost` is the gateway itself).
- The gateway did not compile before today: this Spring Cloud Gateway version dropped `http(uri)`. Now `http()` + `before(uri(...))`.

**Check (skipped by Anton — "I know why"):** why keep those two off the gateway? First answers were the label ("they're internal"). The real reason: a user could call `seat-reservations` 500 times with no ticket → holds make the concert look sold out, again every 10 minutes (denial of inventory). `confirm` is `permitAll` → anyone could make a ticketless hold permanent. Seats may only change through Book, which checks who you are and writes the ticket.

**Side questions:**
- *What happens when I open www.mysite.com?* DNS → IP of the load balancer / gateway → TLS → `GET /` returns the front-end (static host / CDN) → its JS calls `/api/...` through the gateway → the gateway routes by path → the service's JWT filter → controller.
- *What do I see today?* No front-end in the repo. `localhost:8081/` → 404. `/api/events` → raw JSON. `/api/bookings/me` → 401 (the address bar can't send a JWT).
- *So I need a default route?* Only when there's a front-end: everything not `/api/**` goes to the front-end server, checked **last**. Never point it at 8080 (would reopen the internal Event endpoints). Front-end to be built later (Anton asked).

### Tests

The same 7 of 11 fail on the W8 Day 1 commit too — not from today. Causes: tests don't set `JWT_SECRET`; `@WebMvcTest` / `@DataJpaTest` slices have no `CacheManager`; `ConcurrentBookingTest` now goes through the HTTP clients with no server running. With `JWT_SECRET` set the full context loads (new `Booking` mapping is fine). `OutboxPollerTest` and `HoldExpiryJobTest` pass.

### Left for the split

1. ~~Booking FK links~~ — done today.
2. ~~Gateway routes~~ — done today.
3. One database per service.
4. Three deployables (Auth, Event, Booking), each a Compose container.
5. JWT: Auth issues, Event and Booking only check.
6. Rate limiter on the gateway (Anton 2026-10-08). Not in the repo. Extra Book taps get **429** at the gate. Sold out stays **409**.

3–6 are Docker / deploy work and line up with Week 9, next to the per-service database. Leftovers: `holds/*/confirm` is `permitAll` in `SecurityConfig` (port 8080 is still open, so the gateway alone doesn't protect it); the 7 broken tests.

**Carry from Day 3:** Part 3 ran on Day 4. CV: add Docker at the end of the skills list, low-key, ATS friendly.

---

## Week 8 Day 4 (Thu 2026-10-08)

Part 2 (dry-run answers, polish) **not run**. Part 3 was the carried boards. No app code in this session. Rate limiter is scheduled, not built.

### Part 3 — Notification critique

Sketch: one `NotificationService` does every job. `channel` and `status` are Strings. `EmailNotification` and `SmsNotification` each extend a parent and each `send()`. No record of one delivery try.

Anton: one `Notification` with a type enum, no separate channel, status on it, the service checks the type and sends. The one class plus the enum is right. Channel was that same value. The service still must not contain every send. Each pipe is its own sender.

He would not save a failed try (try/catch, bad address → FAILED and stop, other errors retry, then FAILED or SUCCESS). The stop rule is right: a missing address is not retried. A timeout may be retried.

The count of tries has to be committed in the database before the send. He said stop after 2. A crash before that commit drops one try. The next commit still moves the count, so it does not send forever.

**Line:** the try count is committed before the send, and at 2 the notification is FAILED, so a restart does not send again.

### Part 3 — 429 and the queue

Nothing in the repo returns **429**. He asked to build it in Week 9 with the per-service database. Noted in [week-09.md](week-09.md).

A load balancer only picks a pod. It does not line people up. The gate refuses overflow with **429**. A queue is a different box: a few workers pull from it.

**Checks:**
- Sold out first called **429**. Corrected: phones that reach the seat row and find 0 get **409**.
- A phone that waited in the queue, then found 0 seats: **409**. Right, after a miss.
- Waiting does not take a seat. Right.
- 100,000 phones in the queue, the back of the line: timeout, if they stay connected. Right.
- Queue already full: the next phone gets **429**, not a hang. Right.

**Line:** a full queue returns **429**. A phone that reaches Book and finds no seats gets **409**, even after waiting. Waiting does not take a seat.

### Part 3 — URL shortener

Not in this app. He had not seen the product. Questions were too indirect. The picture, stated straight:

Save the long address with a short code nobody else has, and hand back the short link. If the code is taken, pick another. Never overwrite. Open looks up the code and sends the browser there. An unknown code is **404**. A million opens of one link are one lookup a million times, so cache that pair. If every open must be counted, the browser must not remember the jump, because later opens would never call you.

**Checks:** million opens → cache (right). Unknown code → **404** (right). Same code overwritten → the first person lands on the second page (right). Refuse to reuse the short code (right). The click count took a restatement: cache cannot see an open that never reached the server.

**Line:** the short code is unique. Cache the pair for reads. Count an open only when that open calls you.

**Calendar:** carried Part 3 **closed**. Part 2 Thu **not run**. Next is Friday's long mock. Spring split (Auth and Event as their own apps) is in the Spring repo, not from this session.
