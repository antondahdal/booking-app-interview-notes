# Interview notes — Week 7

Polish + performance: security, N+1, indexes, README.

**Packed:** **two** Part 1 topics each Mon–Thu. Friday = one HLD. Full calendar: [part1-map.md](part1-map.md).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | N+1 fix | Index on hot FK |
| Tue | Security pass (secrets/CORS) | README |
| Wed | One perf check **(+ Spring Cache lab if not done)** | Leftover polish |
| Thu | Seat hold + confirm on Event (hold row `HELD` + `expiresAt`, Booking confirms after commit, Event `@Scheduled` expires → seats back). Anton asked W6 Fri. | Test (expiry job) |
| Fri | HLD/recap | — |

**Part 3:** continue LLD class-design (started Week 6). **W7 Wed longer:** this-app cache **if still weak**. `@Cacheable` **code** already **W5 Day 2**.

**Part 3 format change (Anton, 2026-09-24):** LLD = **critique**, not a fresh sketch. Coach shows a flawed class design (missing entity, god-class, wrong is-a, method on the wrong owner). He finds and fixes. ~15 min. Rest of Design = **HLD board**: more services, DB, traffic (~30 min). Do not run the same actors → classes → fields frame again in W7.

| Day | LLD critique (~15) | HLD board (~30) |
|---|---|---|
| Mon | Food-delivery order | Services: split + who owns what data |
| Tue | Split-bill | DB: read replica, index, when to shard |
| Wed (longer) | Chat (User / Message / Room) | Traffic: LB + cache (plus cache talk above) |
| Thu | Notification outbox | Traffic: rate limit (429), queue for spikes |
| Fri | — | Classic HLD URL shortener (longer) |

**Day 1 closed** (Part 3 carried). See below.

---

## Spring Cache lab (coded W5 Day 2)

Same idea as Chapter 4 talk, with Spring names. **Coded** on Week 5 Tuesday (`@Cacheable` / `@CacheEvict`). Store today is in-memory. Redis is still the interview store (not wired).

**Lab status:** Java **done W5 Day 2**. Redis / TTL in Redis **not coded**. W7 Wed = talk if still weak, not a second copy of the same lab unless Redis is the leftover.

### What Spring would do

| Piece | Here |
|---|---|
| `@Cacheable("events")` on Event **GET by id** | Miss → DB, then store. Hit → skip DB. |
| TTL on that cache | e.g. 30s. Copy dies; next GET is a miss. |
| `@CacheEvict` on **reserve / take seat** | After a real write, drop `event:{id}`. |
| Store | **Redis** (one box for all Event pods). Not `ConcurrentHashMap` in the JVM. |

`BookingService.book()` does **not** `@Cacheable`. It calls Event HTTP. Event’s **take** method does not read the GET cache.

Title-only GET may be cached. Remaining seats on the **browse** page may be slightly stale. Remaining seats for **Book** = row lock.

### Trap

`@Cacheable` on `reserveSeats` / “return the last Book DTO.” That is gospel. Also: cache on Booking’s `EventClient` **and** then `book()` uses that number.

### Interview sentence

> `@Cacheable` on Event GET, `@CacheEvict` on take, Redis + TTL. Book still hits the row.

---

## Week 7 Day 1 — N+1 fix + index on the hot foreign key

**Date:** 2026-09-28 (Mon)

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LC-Practice | Kth Largest, Top K Frequent, Search Insert Position. Details: [LC-Practice `notes/week-07-day-01.md`](https://github.com/antondahdal/LC-Practice/blob/master/notes/week-07-day-01.md). **Done.** |
| **Part 1 Design (LC-SD)** | Course card talk | **Chapter 10** consistent hashing. **Done.** |
| **Part 2** | Spring | N+1 fix + index on `bookings.user_id`. **Done.** |
| **Part 3** | LLD critique + HLD board | **Skipped** (Anton). Carried to Day 2. |

---

### Part 2 — The "My tickets" screen (setup for N+1)

**Why it exists now**

Until today Booking could only create a ticket (`POST` Book).
Nothing read a user's tickets back.
A list read is where N+1 shows up, and it is also a screen every real booking app has.

**What was built**

`GET /api/bookings/me` in a new `MyBookingsController`.
The service method `myBookings()` finds the current user through `AuthClient` (same as `book()`), then calls `findByUserId` on `BookingRepository`.
Each booking is mapped to `MyBookingResponseDto`: booking id, event title, seats.

The event title is on purpose.
Reading only `getEvent().getId()` would not hit the database, because the lazy placeholder already knows its id.
Reading the title forces Hibernate to load the event.

---

### Part 2 — N+1

**What it is**

`Booking.event` is `LAZY`.
Loading a list of bookings runs **one** SELECT on `bookings`.
Each booking's event is only an empty placeholder at that point.
When the mapping calls `getEvent().getTitle()` on each row, Hibernate runs **one more SELECT on `events` per booking**.
So N bookings cost 1 + N queries. That is the name.

**What the log showed (3 bookings on 3 events)**

1 query on `users` — this is the Auth hop (`/api/users/me`), not part of the N+1.
1 query on `bookings where user_id = ?` — the list.
3 queries on `events where id = ?` — one per booking. These are the N.

**What the user feels**

Nothing on 3 tickets.
With 50 tickets it is 51 round trips to the database for one screen, so the screen gets slower as the user books more.

**The fix**

`@EntityGraph(attributePaths = "event")` on `findByUserId`.
Spring still builds the query from the method name, but adds a join to `events` and fills each booking's event in the same row.
After the fix the log showed **one** query: `bookings join events where user_id = ?`.

`attributePaths` is the **field name on `Booking`** (`event`, lowercase).
It is not the class name, and it is case-sensitive.

The other tool is a `@Query` with `JOIN FETCH` (like `findAllWithVenue` in `EventRepository`).
Use one or the other, not both. When a method has `@Query`, Spring ignores the method name.

**Why not `EAGER` instead**

`EAGER` goes on the entity, so every place that loads a `Booking` also loads its event, forever.
`book()`, the outbox code, and any future query all pay for it, even when they never read the event.
`@EntityGraph` sits on **one finder**, so only the screen that needs the event pays for it.
Also, `EAGER` on a list query often does not join at all. Hibernate runs the list, then one SELECT per event anyway, so you pay everywhere and still have N+1.

**60-sec:** "My tickets" loaded bookings, then one SELECT per event because `event` is lazy. That is N+1. I fixed it with `@EntityGraph` on that one finder, so it is one join query. I did not switch to `EAGER`, because that would load the event on every Booking query in the app.

**Weak:**
Said the finder alone is "1 SQL" and missed that the loop fires the rest (right after a hint).
First answer on `EAGER` vs `@EntityGraph` said the graph "does not load all at once". It does, in one join. The difference is that it applies only to that finder.
Code slips: `attributePaths = "Booking"` / `"Event"` instead of `"event"`. `@Query` with an undeclared alias and no `WHERE`.

---

### Part 2 — Index on the hot foreign key

**Why it exists now**

The fixed query filters `where user_id = ?`.
With millions of bookings, the database would read the whole `bookings` table to find one user's rows. That is a full scan on every "My tickets" call.

**What an index is**

A separate sorted structure the database keeps next to the table, like the index at the back of a book.
It maps each `user_id` to its rows, so the lookup jumps straight to them instead of scanning.

**The trap**

Postgres does **not** create an index on a foreign key column automatically. It only creates the constraint.
H2 and MySQL do add one, which is why people assume it happens everywhere.

**What was built**

On `Booking`, the `@Table` annotation got an `indexes` attribute with one `@Index`: name `idx_booking_user`, column list `user_id`.
The column list uses the **database column name**, not the Java field name.
`@Index` only works inside `@Table`. It cannot go on a field.
Startup log confirmed it: `create index idx_booking_user on bookings (user_id)`.

**Why not index every column**

Every index is updated on every write.
Each Book inserts a row into `bookings`, so each extra index is one more write inside the hottest write path in the app.
Indexes also take disk space and memory.
An index on a column nobody filters by speeds up nothing.

**60-sec:** "My tickets" filters by `user_id`, so I indexed that column. Postgres does not index foreign keys for you. I do not index every column, because each index slows down every insert, and here every insert is a Book.

**Weak:**
First answer on the cost was "complicated at Hibernate launch". The index is built once; the cost is on every write.
Second answer was memory and disk space, which is true but secondary. Needed the hint to land on the write path (Book).
Code slips: `@Index` first put on the `id` field with no column list.

---

### Part 2 — Lab notes (not topics)

**Book was broken, now fixed.** Every `POST` Book returned **500**.
`@TimeLimiter` only works on methods that return a `CompletableFuture` (async). `reserveSeats` returns `EventResponseDto` directly, so Resilience4j threw before the method even ran.
This is the same problem from W6 Day 3 (TimeLimiter was commented for that demo, then put back).

**Fix:** `@TimeLimiter` and its `timeoutDuration` property are commented out.
The timeout still exists: `WebClientConfig` sets `responseTimeout` to 3 seconds on the HTTP client.
A slow Event throws `WebClientRequestException`, the existing catch turns it into `DownstreamServiceException`, and the phone gets **502**.
The circuit breaker still counts those timeouts as failures.

**Why not make `reserveSeats` async instead:** the call would run on another thread, where `RequestContextHolder` is null (lost `Authorization` + correlation id). `book()` needs the result before it can save, so it would block anyway. Errors come back wrapped in `CompletionException`, which breaks the 409 mapping.

**Verified:** event with 2 seats. Book 2 → **201**. Book again → **409**. "My tickets" shows the one ticket.

**Leftover:** the commented lines and the unused `TimeLimiter` import in `EventClient` should be deleted, not left commented, so nobody puts them back.

For the N+1 demo, the venue, 3 events and 3 bookings were seeded from a throwaway SQL file in `target/` (not in the repo).

---

### Part 3 — skipped

Anton skipped Part 3 today.
Not run: food-delivery LLD critique (~15) and the HLD board "services split + who owns what data" (~30).
The critique design was shown but not answered: `Courier extends Customer`, `Order` holding `List<Dish>` with no line item, and `Order` doing pricing, card charge, courier assignment and SMS.

**Carry:** run both first next weekday, then the Day 2 slots (split-bill critique + read replica / index / shard board).

**Calendar:** Part 1 + Part 2 **closed**. Part 3 **open** (carry). **Next weekday:** Week 7 Day 2 — security pass (secrets / CORS) + README, then carried Part 3.

---

## Week 7 Day 2 — Security pass (secrets + CORS) + README

**Date:** 2026-09-29 (Tue)

| Name | What it is | Today |
|---|---|---|
| **Part 1** | LC-Practice | #56, #228, Ch 11 fixed window vs token bucket. Details: [LC-Practice `notes/week-07-day-02.md`](https://github.com/antondahdal/LC-Practice/blob/master/notes/week-07-day-02.md). **Done.** |
| **Part 2** | Spring | JWT secret out of the repo, CORS, README. **Done.** Anton asked the coach to write the config, CORS bean and README. |
| **Part 3** | Carried Mon + Tue | Food-delivery critique **done**. Services split board **done**. Split-bill critique **skipped** (Anton). DB board **done**. |

---

### Part 2 — JWT secret out of the repo

**Why it exists now**

`app.jwt.secret` was a literal string in `application-dev.properties`, pushed to a public GitHub repo.
Anyone could read it and sign their own token for any user and any role.

**What was built**

The property is now a placeholder that reads the `JWT_SECRET` environment variable, with **no default** (a default would put the secret back in the file).
The app does not start without it.
`docker-compose.yml` passes `JWT_SECRET` from the shell into the booking container and stops with a clear message if it is not set.
Anton runs from the terminal, not IntelliJ: set `$env:JWT_SECRET` in PowerShell before starting.

**Why a new value, not the old one**

The old secret is still in git history.
With it, anyone can **sign** a fresh token for any user, not just replay one.
A new secret makes every token signed with the old one fail the signature check (401).

**60-sec:** The JWT secret comes from an environment variable, never the repo, with no default. Because the old value is in git history, I rotated it; tokens signed with the old secret now fail.

**Weak:**
First answer was why env vars beat hard-coding (true, but not the question).
Second answer: "he can send requests with the token." Missed that he can **forge** new tokens, and that a new value is what kills them.

---

### Part 2 — CORS

**What it is**

The browser's rule, not the API's lock.
Before a cross-origin call with an `Authorization` header, the browser sends an `OPTIONS` preflight asking "may this origin call you?". If the answer does not name the origin, the browser blocks the call.
Curl and Postman never ask.

**Where it goes**

In this app's `SecurityFilterChain`, not the gateway: the browser calls `/api/auth` and `/api/events` straight on 8080, and the gateway only routes Book.
If all traffic later goes through the gateway, CORS moves there, and only there (two places = duplicate headers, the browser rejects).

**What was built**

`.cors(Customizer.withDefaults())` in the chain + a `CorsConfigurationSource` bean.
Origin only `http://localhost:3000` (not `*`). Methods GET, POST, PATCH, OPTIONS. Headers `Authorization`, `Content-Type`, `X-Correlation-Id`. `X-Correlation-Id` exposed so the front end can read it. Registered on `/api/**`.
CORS has to be in the security chain because the preflight carries no JWT, so `anyRequest().authenticated()` would 401 it.

**Verified:** preflight from `localhost:3000` → **200** with the allow-origin header. From `evil.com` → **403**.

**The picture that landed (Dana)**

Dana is logged in, in Chrome. A script on `evil.com` in another tab tries to read her tickets through her browser. Chrome asks the API, the API says no, Chrome blocks. CORS protects **Dana's browser**.
An attacker's curl on his own server: no browser, no question. Only the JWT filter and the URL rules stop him.

**60-sec:** CORS lets only my front-end origin call the API from a browser. It protects the user from other sites in her own browser; it is not auth. A server-side caller is stopped by the token and the role rules.

**Weak:**
Said curl from another server gets 403 "because it's not the defined server." It gets 200 if the token is valid; curl sends no `Origin` and Spring skips CORS.
Needed the Dana picture; the preflight explanation alone did not land.

---

### Part 2 — README

Rewritten (coach draft, Anton asked).
Fixed: "Booking owns inventory" → **Event owns seats**, Booking reserves over HTTP. Diagram now shows gateway → Booking → Event / Auth over HTTP → outbox → poller.
Added: Design decisions table (row lock, `@Version`, correlation id, circuit breaker 503 / 502 / 409, retry only on the Auth GET, outbox, probes, metrics, My tickets N+1 + index), Security section (env secret, roles, CORS, CSRF off), Run with `JWT_SECRET` + `docker compose up --build`.
Compose itself not run today.

---

### Part 3 — Food-delivery LLD critique (carried from Mon)

**Design shown:** `Courier extends Customer`; `Order` has `List<Dish>`; `Order` does `calculateTotal()`, `chargeCard()`, `assignNearestCourier()`, `sendSms()`; `Dish` has the price.

**Found:**
`Courier extends Customer` is a wrong is-a. Share a `Person` parent or a contact-details object; they never extend each other. (Right, first try.)
`chargeCard`, `assignNearestCourier`, `sendSms` move out: payment, dispatch, notification services. (Right.)

**Corrected:**
`calculateTotal()` can **stay** on `Order`: the order holds the lines, adding them up is its own job. Moves only if pricing grows rules (promos, fees, tax).
`List<Dish>` has no quantity → `OrderItem` (dish, quantity, **unitPrice**).
Price copied at checkout into `unitPrice`. Otherwise a menu price change rewrites the total of yesterday's order.
Anton's own version landed: store each order in the DB with the total and the price at that time. Plus: keep the price per line too (receipt, refund one item).

**Interview sentence:** Order items copy the price at checkout; an order is a record of what was paid, not a live view of the menu.

**Weak:**
Did not see the quantity problem until restated ("how does the list say 3?").
Asked why yesterday's total would change: did not see that recalculating from `Dish` reads today's price.
First fix offered: `final`. That freezes the menu or only the reference, not the price the customer paid.

---

### Part 3 — HLD board: services split + who owns what data (carried from Mon)

Food delivery, services from the critique: Order, Payment, Dispatch, Notification, Restaurant.

**Data per service:** Order = orders, items with copied price, status. Payment = charges. Dispatch = couriers, availability, assignment. Notification = what was sent. Restaurant = menus, dishes, **current** prices. (Right.)

**Rule:** no service reads another's tables. Order gets prices by **calling Restaurant over HTTP** (right). Never trust a price the phone sends.

**Sequence:**
Charge = **wait** (right: kitchen must not cook an unpaid order).
SMS = **announce** (right).
Find courier = he said wait. **Announce**: finding a courier can take minutes; the checkout request would hang. Order announces "paid", Dispatch keeps looking, announces "courier assigned".

**Statuses:** his PAID → ASSIGNING_COURIER → COURIER_ASSIGNED → ON_THE_WAY (right). Added CREATED, PAYMENT_FAILED, DELIVERED, CANCELLED.

**One change: no courier after 20 minutes, money already taken.**
He said Payment rolls back the charge. Right idea, wrong word: **refund**, a compensating action (saga). Nothing to roll back; the charge committed long ago in another service.
Dispatch announces "no courier" → Order CANCELLED → Payment refunds → Notification tells the customer.
Refund message arrives twice: he said check status, skip if already REFUNDED (right, idempotent). Trap added: two copies at the same moment both read PAID. Make check-and-change one step (PAID → REFUNDING only if still PAID), and pass the order id as the provider's idempotency key.

**Interview sentence:** Each service owns its data; Order gets prices from Restaurant over HTTP, waits only for the charge, and the rest is events; when no courier is found, a saga refunds, and the refund handler is idempotent.

**Weak:**
"Find courier" as a wait. "Rollback" across services.

---

### Part 3 — Split-bill critique — skipped

Anton skipped it. Design shown, not answered: `double` for money, `balance` on `User` (balances are per group), `Settlement extends Expense`, `Group.addExpense()` saving + balances + currency + email.

---

### Part 3 — HLD board: read replica, index, when to shard

Booking app, festival Saturday, 10× traffic, 95% reads, one Postgres at 90% CPU, Book timing out behind reads.

**First answer:** cache for the hot GETs. Valid (W7 Wed topic), but "My tickets" is per user and changes on Book.
**Read replica:** did not know it. Taught: primary takes every write, replica is a streamed copy slightly behind, read-only queries go there (`readOnly = true` can route).

**Can Book's seat check read the replica?** No, outdated data (right). Plus: `FOR UPDATE` exists only on the primary. Book stays on the primary.

**Dana books, "My tickets" on the replica misses the ticket:** he said block a second booking at Book time. A guard, but people may want two tickets and the screen is still wrong. Fix = **read-your-own-writes**: that user's reads go to the primary for a few seconds (or "My tickets" always on the primary). The 201 already carries the ticket.

**Primary full on writes, 2 TB table:** he proposed old data on one replica, active on others. Replicas are **full copies**, so that is an archive DB, not a replica. Archiving / date partitioning fixes size, not write throughput. Writes need **sharding**.
**Shard key:** did not know. Answer: `event_id`. Book locks the event row and inserts that event's bookings in one transaction; same shard keeps it one normal transaction. By `user_id`, the seat count would be across shards.
Cost (coach gave it, no check): "My tickets" spans every shard → per-user copy fed by events.

**Interview sentence:** Reads go to replicas, but Book and anything that must see its own write stay on the primary; when writes outgrow one primary I shard by the key the transaction locks, here `event_id`.

**Weak:**
Did not know read replica or shard key.
Thought replicas can hold different data.
Coach said "last question" and then asked one more; Anton was annoyed. When saying last, mean it.

---

**Calendar:** Day 2 **closed**. Spring changes (secret, CORS, README, Compose) **not pushed**. **Next weekday:** Week 7 Day 3 — one perf check + leftover polish; Part 3 chat critique + traffic board (LB + cache, longer). Leftover: delete the commented `@TimeLimiter` lines and import in `EventClient`.

---

## Week 7 Day 3 — Perf check (connection held during HTTP) + leftover polish

**Date:** 2026-09-30 (Wed)

---

### Part 2 — Perf check: `book()` held a DB connection while waiting on HTTP

**What a connection is**

An open link between the Java app and the database. Every SQL query goes over one.
Opening one is slow, so Spring Boot opens a fixed set at startup (10 by default) and reuses them. That set is the **pool**; **HikariCP** manages it.
A transaction borrows one connection when it starts and gives it back on commit.

**The problem**

`@Transactional` was on the whole `book()`. So the connection was borrowed **the moment `book()` started**, not at `save`.
Then `book()` called Auth over HTTP and Event over HTTP (up to 3 s). The connection sat idle the whole time.
10 slow Books = all 10 connections idle, and request 11 (even "My tickets") waits and fails.

**Proved it**

Pool set to 1 (`spring.datasource.hikari.maximum-pool-size=1`, `connection-timeout=5000`). One Book → **500**.
Log: `HikariPool-1 - Connection is not available, request timed out after 5004ms (total=1, active=1, idle=0, waiting=2)`.
`active=1` = `book()` holds the only connection. `waiting=2` = the Auth call (same app, needs a connection to look up the user) and its retry, stuck in line. The app blocked itself.
500, not 502: `AuthClient` does not map a timeout to `DownstreamServiceException` (gap, not fixed today).

**The fix (not async)**

HTTP hops stay in `book()`, which is no longer `@Transactional`. The DB part (find user, save ticket, save outbox row, publish) moves to a `@Transactional` method on a **separate bean** (`BookingWriter`).
Same steps, same order, the user waits the same time. Only the connection is borrowed later: just for the save.
Publish stays inside the transactional method, because the listener is `AFTER_COMMIT` and needs a commit.
`myBookings()` had the same bug (`@Transactional` around the Auth call). `@Transactional` removed; the `@EntityGraph` finder already loads the events in one query.
`BookingWriter` is one plain class, no interface + `Impl`: Spring Boot proxies classes directly, and only `BookingServiceImpl` uses it. The transactional method must not be `private` or `final` (the proxy cannot intercept those).

**Verified after the fix:** same pool of 1, same single Book → **201**. No Hikari timeout.

**Why a separate bean, not another method in the same class (self-invocation)**

`@Transactional` works only when the call comes from **another** bean.
Spring wraps each bean in a proxy, and the proxy is what starts the transaction.
A call inside the same class (`this.method()`) skips the proxy, so `@Transactional` on it is silently ignored. No transaction, no error.

**60-sec:** `@Transactional` on `book()` borrowed a DB connection before the Auth and Event HTTP calls, so a slow Event held connections doing nothing. With the pool at 1 one Book blocked itself. I kept the HTTP calls outside and put only the writes in a `@Transactional` method on a separate bean, because a call inside the same class skips Spring's proxy and gets no transaction.

**Check (right, first try):** pool 10, Event slow, 10 Books at once. Can an 11th user open "My tickets"? Yes: the 10 wait on Event without holding a Booking connection; each borrows one only for the short save.

**Weak:**
Did not see at first what HTTP has to do with the connection: thought the connection was taken at `save`, and that moving HTTP out of the transaction makes it async.

---

### Part 2 — Leftover polish

Hikari test lines removed (pool back to default 10). Commented `@TimeLimiter`, its property and the unused import removed from `EventClient` / `application.properties`.

**Timeout today:** 3 s per HTTP call, from `responseTimeout` on the shared `WebClient` bean. Event: 3 s → `DownstreamServiceException` → **502**, no retry. Auth: 3 s per try × `@Retry` 3 attempts ≈ 10 s, then **500** (Auth timeout not mapped — gap).

---

### Part 3 — Chat LLD critique

**Design shown:** `User` (rooms, `unreadCount`); `GroupAdmin extends User` (`banUser`, `renameRoom`); `Room` (members, list of **every** message, `sendMessage` = add + save to DB + push to every member); `Message` (text, sender, `sentAt`).

**Found:**
All messages in `Room` is bad: millions in memory. (Right.) Messages live in the DB, loaded a page at a time by room + time. No `HashMap` needed; the query does it.
`GroupAdmin extends User` → after the hint (Dana admin in Family, member in Work) said admin belongs to the room. Right direction.
`sendMessage` → async worker picks it up after commit (right: outbox, like W6).

**Corrected:**
Admin is a **role in one room**, not a kind of user (wrong is-a). The pair "Dana in Family" is the **missing entity**: `Membership` (user, room, role, joinedAt, `lastReadMessageId`).
Read flag on `Message` fails in a group (Dana read, Anton did not). Unread list on `User` has no room and grows forever. Unread = messages in that room newer than `Membership.lastReadMessageId`; opening the room moves it.
`Room` is data. `MessageService` = save + outbox row. `NotificationService` = pushes. (Same move as `chargeCard` / `sendSms` leaving `Order`.)

**Interview sentence:** Admin is a role in a room, not a kind of user, so user and room meet in a `Membership` that holds the role and `lastReadMessageId`; messages are paged from the DB, and notifications go out after commit through an outbox.

**Weak:**
First answer listed new features (admin adds users, user creates room, read flag) instead of flaws.
Did not see the wrong is-a until the Dana picture.
"Unread comes from the DB" without the field to count against.

---

### Part 3 — HLD board: traffic, load balancer + cache (longer)

Festival lineup at 10:00. 200,000 phones on `GET /api/events/5`. 3 Event pods. Book is small but must be correct.

**Load balancer:** named it (right). How it picks: "who is less busy" = **least connections** (right). Also **round robin** (in turn; default, fine when requests cost the same). Only sends to pods whose **readiness** passes (W5).
**Dana's next request on another pod:** "they share the same DB" (right) + **stateless**: identity is in the JWT, every pod has the same secret, no session in memory → no sticky sessions.

**Cache:** separate shared pod (right; Redis, not per-pod memory, or pods disagree). Key `event:5`. **Cache-aside:** check → hit returns → miss reads DB, stores with **TTL**.
**1 seat left in the cache, 3 tap Book:** Book reads the DB, first 201, others 409 (right). From the cache all three would get a ticket for one seat.
**Stale "1 left" for 30 s after the last seat:** not a business problem, Book gets a clean 409 (right). His fix "update cache on insert" → **evict**, not update (two updates can land out of order; delete can't be wrong). Evict in **Event** (owns seats), not Booking's insert. `@CacheEvict` coded W5.

**One change: key expires at 5,000 req/s.**
He said "first reads, rest from cache." Wrong: nothing makes the others wait. Every request in the ~20 ms gap misses → ~100 same SELECTs → pool full → Book waits behind page views. Name: **cache stampede** / thundering herd.
Fix: "lock" (right). Which lock: first said the row lock (`FOR UPDATE`). Wrong: the 100 are already in the DB holding connections, and Book locks the same row (W3: never lock on GET). Then: "the lock on the cache" (right). Name: **distributed lock** in Redis (`SET lock:event:5 NX` + short expiry so a crashed pod frees it). One DB read total. Per-pod `synchronized` = 3 reads, also acceptable.

**Change: Redis down at 10:10.**
Must not take the system down; cache is a speed-up, not data (right). Page views fall back to the DB (right). Book works the same, never used the cache (right, after the question was restated).
With high traffic, Book gets slower: all reads now hit the DB Book uses (right, after restating as "with high traffic what happens to Book"). Slow enough → 502 / lock wait timeout.
Protect Book: **read replica** for page views, primary for Book (right, from W7 Day 2). Correction: the DB streams to the replica itself; the app does not "update it frequently". Also: short timeout on Redis calls; tiny in-pod cache (seconds) as a second layer.

**Interview sentence:** A load balancer spreads requests across stateless pods; event pages come from Redis with cache-aside and a TTL, the take-seat path evicts the key, and a Redis lock stops a stampede on a miss. Book never reads the cache; it locks the row on the primary. If Redis dies, reads fall back to a replica so they don't slow Book.

**Weak:**
Stampede: assumed the first miss makes the rest wait.
Reached for the row lock to protect the cache.
Needed the "what happens to Book" question in plain words (Anton: phrase it as "with high traffic, what happens to Book").

---

**Calendar:** Day 3 **closed**. Spring change (`BookingWriter`, `book()` / `myBookings()` without `@Transactional`, TimeLimiter leftovers removed) **not pushed**. Notes **not pushed**. **Next weekday:** Week 7 Day 4 — seat hold + confirm on Event (TTL expiry) + expiry job test; Part 3 notification-outbox critique + traffic board (rate limit 429, queue for spikes).

