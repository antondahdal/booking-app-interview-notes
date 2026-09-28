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

**Book is broken on master.** Every `POST` Book returns **500**.
`@TimeLimiter` on `EventClient.reserveSeats` needs a `CompletionStage` return type; the method returns `EventResponseDto` directly.
This is the same problem from W6 Day 3 (TimeLimiter was commented for that demo, then put back). Not fixed today.

For the N+1 demo, the venue, 3 events and 3 bookings were seeded from a throwaway SQL file in `target/` (not in the repo).

---

### Part 3 — skipped

Anton skipped Part 3 today.
Not run: food-delivery LLD critique (~15) and the HLD board "services split + who owns what data" (~30).
The critique design was shown but not answered: `Courier extends Customer`, `Order` holding `List<Dish>` with no line item, and `Order` doing pricing, card charge, courier assignment and SMS.

**Carry:** run both first next weekday, then the Day 2 slots (split-bill critique + read replica / index / shard board).

**Calendar:** Part 1 + Part 2 **closed**. Part 3 **open** (carry). **Next weekday:** Week 7 Day 2 — security pass (secrets / CORS) + README, then carried Part 3.

