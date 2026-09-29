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

