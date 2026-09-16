# Interview notes — Week 5

Gateway + resilience: Gateway, Resilience4j, Docker Compose. Git: [git.md](git.md).

**Packed:** **two** Spring topics each Mon–Thu. Friday = one HLD. Full calendar: [part2-map.md](part2-map.md).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Gateway routes | First service behind it |
| Tue | Resilience4j timeout | Retry (which calls, which not) |
| Wed | Circuit breaker on the hot call | Fallback status |
| Thu | Docker Compose for the set | One health check |
| Fri | HLD traffic through gateway | — |

---

## Week 5 Day 1 — Remove Nth + Reverse + cache talk

**Date:** 2026-09-14 (Mon)

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | [LC-Practice](https://github.com/antondahdal/LC-Practice) | **#19** he coded. **#206** he coded. **Done.** |
| **Part 1 Design (LC-SD)** | Course card talk. **Still Part 1.** | **Chapter 4** — Cache / TTL. **Done.** |
| **Part 2** | Spring | Gateway routes + first service behind it. **Done.** |
| **Part 3** | OOP + this-app design | equals / hashCode + Collections (longer) + JWT at gateway vs service. **Done.** |

Do **not** call Chapter 4 “Part 3.” Spring `@Cacheable` lab is **W7 Wed**, not this week.

---

### Part 1 — two LCs — passed + Design talk

#### LC 19 Remove Nth Node From End of List (Medium) — passed

`n` is from the **tail**. `1 → 2 → 3 → 4 → 5`, `n = 2` drops **4**. `1 → 2`, `n = 2` drops **1**, leftover `{2}` (not “node value 2”, not 2nd from head).

**Gate:** first said sliding window (wrong). Picture was two walkers with gap **n**. Name: two pointers. O(n) / O(1).

**Memorize this:** Front walks `n`. Then both until front has no next. Back sits on the node **before** the victim. `back.next = back.next.next`. If after the `for` front is already `null`, `n` = length → victim is **head** → `return head.next` (do not rewire while sitting on head). Dummy before head = one path, no `if`.

**Weak:** mid-list skip used for the head case. Single node `n = 1` → NPE on `back.next.next`. Long list with `n = length` is the same head case.

**Cousin:** dummy; two-pass count length.

#### LC 206 Reverse Linked List (Easy) — passed

Flip every `next`. New head = old tail. Empty → `null`.

**Gate:** two pointers, but neighbor swap is **#24**, not reverse.

**Memorize this:** Behind starts **null**, current = head. Save `current.next`, point current at behind, behind = current, current = saved. Loop `while (current != null)`. `|| current.next` NPEs. Both on head → `1.next = 1` cycle.

**Cousin:** recursion; #92 reverse a slice.

---

### Part 1 Design — Chapter 4 How caching improves performance — Cache / TTL

Talk. No Java. Still Part 1.

Anton: cache on **Event** because the write is there. Gospel = cache is not the real DB. Did not name a key. Did not get **10×**.

#### What it is

A cache is a **copy** of a **GET** answer so browse does not hit Event DB every time.

- **Miss:** locker empty → call Event → store under a key.
- **Hit:** return the copy.
- **TTL:** how long the copy may live. After that, miss again. Stale is **bounded**, not forever.

We cache **browse** (GET event 7). We do **not** cache Book / “you got a seat.”

#### Where it sits

Event **DB** is truth. Best: cache **in front of Event’s GET** (one owner). Booking cache only skips HTTP on browse. **`book()` skips the cache** either way.

Key: `event:{id}`. Shared Redis, not a `HashMap` on one pod.

After a successful take: **delete / refresh** `event:7` so the next GET is not wildly wrong. That updates the **browse copy**. It is not “the cache is now the booking.”

#### 10×

10× more **people browsing**, not 10× Book. Extra GETs should be hits. Book volume stays small and still hits the row.

#### Trap

Redis “1 seat left” → **201**. Never. Sold out is Event **409**. Stale **browse** is OK. Stale **take** is a fake ticket.

#### Interview sentence

> Cache GET with a TTL. Book is Event’s row. Never treat “1 left” in cache as a ticket.

### 60-sec (Part 1)

> #19: gap of n, dummy or `return head.next` when front falls off. #206: behind = null, save next, point back. Ch 4: cache browse; Book skips it; TTL; 10× = more GET; never cache → 201.

**Weak:** sliding window on a list. Neighbor swap as reverse. Cache on Booking as the ticket. 10× mixed with more Book. Event cache = Event DB.

**Calendar:** Part 1 **closed**. Part 2 **done** (see below). Part 3 **done** (see below).

---

### Part 2 — Gateway routes + first service behind it

**Date:** 2026-09-14 (Mon, after Part 1)

**Goal:** Fourth box. Phone’s one door. One Book route to this app. No Resilience4j, no JWT-on-gateway code, no cache lab.

Anton wrote a **second process** under `gateway/` (own `src/` + `pom.xml`). Not a class inside Booking’s `src/`.

| Piece | What |
|---|---|
| `GatewayApplication` | Boot class. Same job as `EventBookingPlatformApplication`, other JVM |
| `server.port=8081` | Gateway door. Booking stays **8080** |
| `booking.service.uri=http://localhost:8080` | Where to forward |
| `BookingRouteConfig` | `POST /api/events/{eventId}/bookings` → `http(bookingUri)` |

`route("booking")` = id. `path` + `POST` = match Book. `http(uri)` = copy method/path/body/**headers** to Booking. `.build()` = freeze the rule.

**First service:** Booking only. Gateway does **not** call `EventClient`. GET `/api/events/7` on 8081 → gateway **404** (no route). Browse still hits **8080** directly. `www.example.com` in prod = the gateway.

**Headers:** `http()` **copies** `Authorization`. Gateway does **not** parse JWT today. Drop the Bearer → Booking **401**. `reserveSeats` stays in `book()`.

**Trap:** Gateway starter on Booking’s pom (same JVM = not a box). Path ≠ controller → 404. `EventClient` is a box.

### 60-sec (Part 2)

> Phone → gateway (8081). One route: POST Book → Booking (8080). Same path as `BookingController`. GET Event is not behind it yet. Headers copied, not checked. Seats still Event via Booking.

**Weak:** Gateway = Booking class. GET magically uses the Book route. Gateway checks JWT today.

**Calendar:** Part 2 **closed**. Part 3 **done** (see below).

---

### Part 3 — equals/hashCode + Collections + JWT at gateway vs service

**Date:** 2026-09-14 (Mon, after Part 2)

**`equals` / `hashCode`:** On the **entity** `Booking` (`HashSet<Booking>`), not the DTO. Both on `id`, stay in sync. Today’s class has neither → two instances, same id → set size **2**. With both → **1**.

**Collections:** 50th item → `ArrayList` (`get(50)`). Slide a new item **between** two existing → `LinkedList` (change links). `ArrayList.add(index, x)` **can** insert in the middle; it **shifts** later items. `add(x)` with no index = end. This app: `ArrayList` / DB `Page`, not a splice chain.

**JWT at gateway vs Booking:** Gateway **may** check first (junk never forwards). Booking **still** checks. `:8080` is still open; skip the door → still need Bearer. Token is **who**; role is **may**. Bearer is not a second envelope — it **is** the JWT inside `Authorization: Bearer …`. Correlation id is a **header**, not a claim.

### 60-sec (Part 3)

> Entity `equals`+`hashCode` on id. HashSet needs both. ArrayList = jump to index. LinkedList = insert between. Gateway check ≠ skip Booking’s filter.

**Weak:** `equals` on the DTO. ArrayList cannot insert except at the end. Check JWT only on the gateway because “once is enough.”

**Calendar:** Day 1 **closed**. Next was Week 5 Day 2 (below). `@Cacheable` **code** was pulled to Day 2 (Anton asked to code it).

---

## Week 5 Day 2 — timeout + retry + cache

**Date:** 2026-09-15 (Tue)

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | [LC-Practice](https://github.com/antondahdal/LC-Practice) | Linked list bank. **Not this chat.** |
| **Part 2** | Spring | Resilience4j timeout + retry + **cache lab** (Anton asked to code it). **Done.** |
| **Part 3** | OOP + this-app design | Immutability + `Optional` + click id to retry Book. **Done.** |

---

### Part 2 — Resilience4j timeout + retry + cache

**Date:** 2026-09-15 (Tue)

**Goal:** Named timeout on the Event call. Retry GET, not Book. Cache browse GET; evict on take. Redis talk (how two servers stay in sync).

#### Timeout

Booking **stops waiting**. Event is another process — timeout does **not** roll back Event’s row. Hang-up ≠ “no seat taken.” Phone gets **502** (`DownstreamServiceException`), not **201**.

Repo `lock.timeout` = wait to **get the row lock** on that query. Resilience4j / WebClient 3s = Booking waits for the **HTTP answer**.

WebClient 3s is one shared client → `AuthClient` and `EventClient` can both time out. Resilience4j instance name `event` is the Event call (`@TimeLimiter(name = "event")` on `reserveSeats`). Properties line alone does nothing until the method uses the same name.

Jars on **Booking’s** pom (not gateway). Gateway never calls Event. EventController must **not** time out its own take.

#### Retry

`@Retry(name = "eventGet")` on `AuthClient.checkIfExist` (GET). `maxAttempts=3`.

GET = read. Asking twice does not create a user. Book / `reserveSeats` = write. No click id today → retry can take a **second** seat. GET yes because it does not duplicate. Book no because we have no duplicate guard.

#### Cache (this app + Redis)

**What we coded**

| Piece | Where |
|---|---|
| `spring-boot-starter-cache` | Booking pom |
| `@EnableCaching` | `EventBookingPlatformApplication` |
| `@Cacheable("events")` | `EventServiceImpl.getEvent` |
| `@CacheEvict(value = "events", key = "#id")` | `EventServiceImpl.reserveSeats` |

Spring sees `@Cacheable("events")` on `getEvent` — not “getById” by magic. Call `getEvent(7)` → look in box `events`, key `7`. Miss → DB, store DTO. Hit → skip DB.

`@CacheEvict` same name `events`, `#id` = that method’s `id`. It **deletes** the copy. It does not reload GET immediately. Next `getEvent(7)` is a miss, then DB. Wrong name → browse stays stale.

**Not on** `BookingServiceImpl.book()`. Take still hits the row. Cache “1 left” is not a ticket.

**Today’s store:** a map **inside this JVM**. Two servers = two maps. They do **not** stay in sync.

#### How we keep cache in sync — Redis

Yes. **Redis is how you keep the cache in sync** when you have two servers.

Each server does **not** keep its own copy of event 7. Both servers use **Redis as the cache**. Redis is one box on the network. Server A and server B both talk to that same box.

**How Redis is used in general:** it is a key–value store. You put a key, you get a value, you delete a key.

For us:

1. Someone browses event 7. The server asks Redis: do you have `events` / `7`?  
   - No → load from the Event database, then **SET** that DTO in Redis under that key.  
   - Yes → **GET** it from Redis. Do not hit the database.

2. Someone books. `reserveSeats` runs, then **DEL** that same key in Redis (`@CacheEvict`). Redis no longer has event 7. The next browse on **either** server misses and loads from the database again.

3. You can also set a **TTL** (e.g. 30 seconds). Redis deletes the key by itself when time is up.

Nothing is “synced between A and B.” A and B simply **share one Redis**. If A deletes the key, B cannot see a stale copy, because B was never storing it locally.

Java stays `@Cacheable` / `@CacheEvict`. You swap the store to Redis. Book still ignores Redis and hits the row. Two `reserveSeats` still lock the **database**, not Redis.

**10×** = more **browse** (GET hits). Not 10× Book.

**Trap:** Redis “1 seat left” → **201**. Never. Sold out is Event **409**. Stale browse is OK. Stale take is a fake ticket.

### 60-sec (Part 2)

> Timeout: Booking hung up; Event may still have taken a seat; 502 not 201. Retry GET (read). Do not retry Book without a click id. Cache `getEvent`; evict on take; same name `events`. Two servers: Redis is the shared store (GET/SET/DEL + TTL), not a HashMap per JVM. Book still hits the row.

**Weak:** Timeout = Event rolled back. Retry Book because GET has no guard. Cache on `book()`. Two servers sync their HashMaps. Redis “1 left” = ticket.

**Calendar:** Part 2 **closed**. Part 3 **closed** (see below).

---

### Part 3 — immutability + Optional + click id

**Date:** 2026-09-15 (Tue, after Part 2)

Each topic closed on its own. Do not mix them.

#### Immutability — closed

The question in human words: after `book()` has started, should we still treat `dto.seats` as a number we can edit?

**No.** That number is the user’s order (“I want 2 seats”). `book()` sends it to Event and saves it on the ticket. If it changes mid-method, Event can take 3 and the ticket can say 2.

`Event.availableSeats` **may** change. That is the warehouse, not the order.

DTO has a setter today so Java **can** change it. “Is it OK?” means **should you**. You should not.

**Trap:** thinking the question is “does `@Setter` compile?”

#### Optional — closed

`findByIdForUpdate` returns `Optional<Event>`. Empty **box** = no row with that id. Not an Event with 0 seats.

This app: `orElseThrow(() -> new ResourceNotFoundException(...))` → handler **404**. Not a fake Event.

#### Click id — closed (interview words; **not in the app**)

If Book times out, we do not know if Event already took a seat. Retry without a guard can take a **second** seat.

**Click id** = one UUID for **this one tap**. Event **stores** it. Same id again → same take, do not subtract again.

- **Who mints:** the **phone**, once per tap. Retry of that tap sends the **same** UUID. If Booking mints a **new** UUID on every HTTP hit, two hits = two seats.
- **Where:** Event (the take), not Booking. Timeout means Booking may never have saved.
- **Not** the JWT (who you are for the whole login — one token, many taps).
- **Not** the correlation id (log sticker; we mint if missing; new request can get a new sticker).

GET retry was already closed in Part 2. Do not re-ask it.

**Trap:** click id = token. Click id = correlation id. Mint a new UUID per HTTP retry.

### 60-sec (Part 3)

> DTO seats stay frozen; Event seats may change. Empty Optional = no row → 404. Click id = phone UUID for this tap, stored on Event; retry sends the same one. Not JWT, not correlation id. Not built.

**Weak:** JWT as click id. Correlation id as click id. Server mints a new UUID on every retry.

**Calendar:** Day 2 **closed**. **Next weekday:** Week 5 Day 3 — LC first, then circuit breaker + fallback status. OOP: LSP (fallback ≠ 201).

---

## Week 5 Day 3 — Add Two Numbers + Merge Two Lists + hot key

**Date:** 2026-09-16 (Wed)

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | [LC-Practice](https://github.com/antondahdal/LC-Practice) | **#2** coach wrote. **#21** he coded. **Done.** |
| **Part 1 Design (LC-SD)** | Course card talk. **Still Part 1.** | **Chapter 10** — Hot key / partition lite. **Done.** |
| **Part 2** | Spring | Circuit breaker on the hot call + fallback status. **Done.** |
| **Part 3** | OOP + this-app design | LSP **skip** (repeat). Leftover: checked vs unchecked. Design: circuit open board. **Done.** |

Cover from today: always paste the LeetCode URL.

Week 5 printed bank leftover was **#21**. Mon/Tue already used the Mediums on that line, so first LC was Top Interview 150 **#2** (not on the printed Week 5 line), then Easy **#21**.

---

### Part 1 — two LCs — passed + Design talk

#### LC 2 Add Two Numbers (Medium) — passed, coach wrote it

Lists store digits least-significant first. `2 → 4 → 3` plus `5 → 6 → 4` is `342 + 465` → `7 → 0 → 8`.

**Gate:** “regular loop”; then strings + `Integer` (overflow on long lists). Stack is **#445** (ones at the tail), not this problem. Name: two walkers. Keep **carry**. O(max(m,n)) / O(1) extra besides the new list.

**Memorize this:** Dummy + tail. Loop while either list **or carry**. Digit = (l1 or 0) + (l2 or 0) + carry. Write `% 10`, carry = `/ 10`. Missing node is 0, not stop.

**Weak:** splice leftover list when one walker falls off. Carry can still change those digits. `9 → 9` plus `1` is `0 → 0 → 1`, not `1 → 9`.

**Cousin:** dummy; #445 stack because MSD is at the head.

#### LC 21 Merge Two Sorted Lists (Easy) — passed

`1 → 2 → 4` and `1 → 3 → 4` → `1 → 1 → 2 → 3 → 4 → 4`.

**Gate:** two walkers, O(n+m). First said extra space = new list. Extra is **O(1)** if you rewire existing nodes.

**Memorize this:** Dummy in front. `tail` is last kept. While both alive, hook the **smaller** head (one node per step; equal 1s are two steps). After the while, leftover is one assignment: `tail.next = list1 or list2`. Return `dummy.next`.

**Weak:** took the **larger** head. Copied `new ListNode` instead of hooking the live node. Leftover ifs inside `while (both)` never run. Returned dummy, then fixed to `dummy.next`.

**Cousin:** same dummy as #2; hook leftover in one shot.

---

### Part 1 Design — Chapter 10 How to deliver data at scale — Hot key / partition lite

Talk. No Java. Still Part 1. Do not redesign Kafka.

Anton: hot key is **7**. Client hits GET and Book. Nothing “breaks.” **409** = lots of Book on 7, seats run out. **502** = traffic, lock wait on that row. He had both. Coach mashed 409 with traffic, then restated 502 as if it was new.

#### What it is

A hot key is **one** popular id. Most traffic hits that slot. Other ids stay quiet.

This app: `event:7` — that Event **row**, cache key, lock. Not the whole catalog.

- **Browse:** GET event 7. Can be cache hits (Mon/Tue).
- **Book:** POST take on 7. Still hits Event’s **one row**. Extra pods do not split id 7. All Book-7 still serialize on that lock.

10× means 10× **that event**, not 10× the catalog. Other events stay fine.

#### Status

- **409** = Event sold out. Traffic emptied 7. The row worked. Not “the box died.”
- **502** = Booking hung up waiting on Event (lock wait / timeout on that take). Not a ticket.

#### Trap

10× servers fixes a hot event. Treating 409 as overload. Stale cache “1 left” → **201** (Ch 4; do not re-teach).

#### Interview sentence

> Hot key is event 7. Extra browse can hit cache. Extra Book still fights one Event row. More pods do not split that id.

### 60-sec (Part 1)

> #2: dummy, carry, missing digit is 0, loop while list or carry. #21: smaller head each step, leftover in one hook, dummy.next. Ch 10: hot key = event 7; GET can cache; Book still one row; 409 = sold out; 502 = lock wait; pods do not split that id.

**Weak:** Integer/string add. Stack on LSD-first lists. Merge took the larger node. Coach misheard 409 vs 502.

**Calendar:** Part 1 **closed**. Part 2 **done** (see below). Part 3 **done** (see below).

---

### Part 2 — circuit breaker + fallback status

**Date:** 2026-09-16 (Wed, after Part 1)

**Goal:** Named circuit on the Event take. Open = do not call Event. Fallback is not a ticket.

#### Circuit breaker on the hot call

Jars already on Booking’s pom (Day 2). Gateway never calls Event. Annotation on `EventClient.reserveSeats`, same name `event` as `@TimeLimiter`.

| Piece | Where |
|---|---|
| `@CircuitBreaker(name = "event", fallbackMethod = "reserveSeatsFallback")` | `reserveSeats` |
| `slidingWindowSize=10` | last 10 takes |
| `failureRateThreshold=50` | half fail → open |
| `waitDurationInOpenState=10s` | then one trial (half-open) |
| `ignoreExceptions` | `InsufficientSeatsException`, `ResourceNotFoundException` (full class names, **one** property line, no `.java`) |

Closed = keep calling Event. Open = Booking does **not** enter `reserveSeats`. Resilience4j throws `CallNotPermittedException`. Timeout still waits 3s then hangs up; open is fail fast.

**409** is Event working (sold out). Those exceptions must not count toward the 50%, or later callers get **503** while Event is fine.

Two `ignoreExceptions=` lines: the second **replaces** the first. Simple name / `.java` suffix: `Class.forName` cannot load them.

#### Fallback status

`reserveSeatsFallback` lives on **`EventClient`** (same class as the annotation). Same return type, same args, last `Throwable`. **Throw.** Do not return a DTO. Do not return `null`. A return means seats were taken → `book()` saves → phone **201** with no take.

The fallback also runs when `reserveSeats` **throws**, not only when the circuit is open. Sold out is already `InsufficientSeatsException`. If the fallback always throws `DownstreamServiceException`, Maria gets **502** instead of **409**. Rethrow `InsufficientSeatsException`, `ResourceNotFoundException`, and `CallNotPermittedException`. The rest stay `DownstreamServiceException` (**502** = we called Event and the hop died).

`CallNotPermittedException` → handler **503** (`SERVICE_UNAVAILABLE`). Not **500** (unhandled). Not **502**. Title “Event unavailable,” not “Server Error.”

Anton first said open already throws `DownstreamServiceException` (internal error). Unhandled open is **500**. After the handler: **503**. Status check named **409** and **502**; missed **201** until asked. Then: fallback runs on error / open, so a returned DTO is fake.

### 60-sec (Part 2)

> Circuit on `reserveSeats`, name `event`. Open = do not call Event. Phone **503**, not **201**. Sold out still **409** (rethrow; ignore on the circuit). Fallback throws; a returned DTO is a fake ticket. **502** = we tried the hop. **503** = we did not.

**Weak:** Open = `DownstreamServiceException` / **500**. Fallback `if` on `WebClientResponseException` (already mapped). Two `ignoreExceptions` keys. Simple names / `.java`. Status list skipped **201**.

**Calendar:** Part 2 **closed**. Part 3 **done** (see below).

---

### Part 3 — checked vs unchecked + circuit open board

**Date:** 2026-09-16 (Wed, after Part 2)

LSP **skip** (W4 Thu + Part 2 today). Anton: do not re-ask; still run the OOP **slot** with leftover.

#### Checked vs unchecked — closed

`InsufficientSeatsException` is a `RuntimeException`. Unchecked: no `throws` on `reserveSeats`. Checked (`extends Exception`) would force `throws` or `try/catch` on client, `book()`, controller. Empty catch so it compiles → `book()` continues → phone **201**. Domain failures stay unchecked so they reach `GlobalExceptionHandler` (**409** / **502** / **503**).

#### Circuit open board — closed

Phone → Booking controller → `book()` → Java call `eventClient.reserveSeats(...)`. Annotation is on `reserveSeats`, not `book()`. Open: that Java call throws `CallNotPermittedException` (fallback rethrows). `book()` has no catch → handler **503**. **Zero** HTTP to Event. Coach misread “call EventClient” as “Event was POSTed”; Anton’s sequence was right.

After `waitDurationInOpenState`: **not open** (half-open). One trial **does** run `reserveSeats` and HTTP. Event still down → `DownstreamServiceException` → **502**, then open again. Trial works → real DTO, **201**, circuit closes. Not a fake ticket.

### 60-sec (Part 3)

> Domain = unchecked; empty catch → **201**. Open: `book()` → `reserveSeats` Java call throws `CallNotPermittedException` → **503**; no Event HTTP. Half-open trial is not open: hop runs; still down → **502**.

**Weak:** Open mixed with “HTTP to EventClient.” Coach over-corrected Java vs HTTP.

**Calendar:** Day 3 **closed**. **Next weekday:** Week 5 Day 4 — LC first, then Docker Compose + one health check. OOP: ISP (health ≠ book). Design: live vs ready; kill a pod holding `FOR UPDATE`.


