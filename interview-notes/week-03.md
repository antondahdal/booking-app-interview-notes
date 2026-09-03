# Interview notes — Week 3

Concurrency + tests: double-booking, `@Transactional`, locking.

**Packed:** two Part 1 topics per weekday (Fri = long board). Full calendar: [part1-map.md](part1-map.md).

Wed 2026-09-02: test + wait on `book()` + title PATCH. **Thu 2026-09-03:** two **new** code topics (`WebClient` bean + Booking calls Event over HTTP). Do not empty a weekday. **Fri:** long HLD.

You can reread a day without the chat. These notes are written in plain English.

---

## Week 3 Day 1 — Event-row lock on book

**Date:** 2026-08-31  
**Goal:** Two book calls on the same concert cannot both pass the seat check on a stale `1`. Lock that event row for the whole `book()` transaction. Cap how long the waiter blocks.

### Quick recall (Spring — Part 1)

**Transaction vs lock**

`@Transactional` is one bucket for this request — all writes commit or all roll back. It does **not** make another request wait. A **lock** is the database holding a row so a second transaction cannot change it until the first commits.

Here, `book()` already had a transaction. The hole was `findById` (plain read). `findByIdForUpdate` plus `PESSIMISTIC_WRITE` is the wait.

The mix-up: `@Transactional` is not a lock.

**Pessimistic vs optimistic (when you lock, not how much)**

**Pessimistic** = lock **now**, others **wait**, then you change. **Optimistic** = no wait now; remember a stamp (version); on save, if the stamp changed, fail and retry. Table vs **row** is a different question (how much). Pessimistic can still be one row.

Here, we used pessimistic on **event 5’s row**. No `@Version` today.

The mix-up: pessimistic does not mean “lock the whole table.” Optimistic does not mean “lock one row.”

**Pessimistic write / `FOR UPDATE`**

Hibernate turns `PESSIMISTIC_WRITE` into `SELECT … FOR UPDATE`. The database holds that row until **this** transaction commits. Not a Java `synchronized`.

Here, `@Lock(PESSIMISTIC_WRITE)` sits on `findByIdForUpdate`. Only `book()` calls it.

The mix-up: putting `@Lock` on inherited `findById` puts **GET** in the same line. Browse waits; it should not.

**Which row**

Lock the row that **is** the inventory, not the whole table (everyone queues), not a related parent.

Here, that is event 5 (`availableSeats`). Concert 6 is another row. Venue is the hall — seats are not stored there.

The mix-up: locking the venue would block other shows in the same hall.

**Same transaction**

The lock lives until **this** transaction commits. Split “lock then return” and “subtract later” = two buckets. First commit **releases**. Second caller can still win. Split is not a lock that sits forever.

Here, lock, check seats, save booking, decrement — all in `book()`.

The mix-up: an unknown-long lock is slow work **inside** the same method (HTTP call while holding the row). Keep `book()` short.

**`synchronized` vs DB lock**

`synchronized` on a Spring service lives in **this JVM**. Two app instances, one database → two Java locks, still oversell. Also one `synchronized` on `book()` makes every concert wait behind one concert on that machine.

Here, the seat number is in the DB, so the lock is the DB row.

The mix-up: a `HashSet` of event ids has the same “one process” hole.

**Lock wait timeout**

A lock wait can be capped. Timeout = never got the row (seat check never ran). Sold out = got the row, current state says no. Unhandled lock exception → **500**. Mapping timeout to **409** is wrong.

Here, `@QueryHints` plus `@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")` sit on `findByIdForUpdate` only.

The mix-up: `@QueryHint` cannot sit on the method alone (“disallowed for this location”). Wrap with Spring Data `@QueryHints`. H2 may ignore the hint; the interview answer is the picture.

**After the first buyer commits**

The waiter was blocked on the locking select **inside the same request**. Wait ends, they read the new number, then 409 if nothing is left. No second HTTP. UI can still show an old GET.

Here, Maria stays in `book()`, reads **0**, **409**. No refresh required.

The mix-up: 409 after wait is not timeout 500.

### What I built

- `EventRepository.findByIdForUpdate` — `@Query` + `:id` + `@Param` + `@Lock(PESSIMISTIC_WRITE)` + lock-wait `@QueryHints`
- `BookingServiceImpl.book()` loads with that method, not `findById`
- GET / `EventService` still use `findById`

### Gate (weak spots)

- **Q2:** Swapped pessimistic/optimistic with table/row. Pessimistic = wait **now** (we did this, on one row). Optimistic = version **later** (not today).
- **Q4:** Did not know `synchronized`. Java lock = this process. Two servers still oversell. DB row lock is shared.

### 60-sec

> Transaction is one commit, not a queue. I lock the event row on book (`FOR UPDATE`). Browse does not take that lock. Lock, check, and subtract stay in one transaction. Timeout is “never got the row” (500 if unhandled), not sold out (409). `synchronized` is not the interview lock.

---

### Part 2 — two LCs (from today: 2/weekday through Week 5)

Cadence: Weeks 3–5 = **two** LCs/weekday. Weeks 6–8 = **three**. Sat/Sun off. Most-asked only (Blind 75 / Grind 75 / NeetCode 150).

---

#### Two pointers vs “index + worker” (learn this — it is the week’s hard part)

Two pointers means two indexes, and **neither resets**. Each step **throws one side away** because one comparison tells you that side can never be in the answer. Usual setup: array **sorted**, or you start at **both ends** (palindrome). Time **O(n)**.

Index + worker is different. Stand on `i`. Worker walks `i+1, i+2, …`. Then `i++` and worker **goes back** to the new `i+1`. That is a **nested loop**, **O(n²)**. Two integer variables do not make it two pointers.

Here, #167 and #125 are both **ends**. #739 Daily Temperatures was the worker (each day has its own look-ahead, no throw-away rule) → stack, not two pointers.

The mix-up: calling the worker “two pointers.” Interview two pointers **never** send the second index back to `i+1`.

| | Two pointers | Index + worker |
|---|---|---|
| Second index | Other **end** (or stays ahead, never resets) | Starts at `i+1` **every** `i` |
| After a miss | Move **one** end in | Keep walking, then **restart** |
| Time | **O(n)** | **O(n²)** |

---

### LC 167 Two Sum II — Input Array Is Sorted (Medium) — passed

Array is **already sorted**. Find two **different** indexes whose values sum to `target`. Exactly one pair. Return **1-based** indexes, smaller first.

`[2,7,11,15]`, target `9` → `[1,2]` (`2+7`). Extra space must be **O(1)** (no map).

---

#### Why two pointers

Sorted pair plus “throw a side away.”

Here, `left = 0`, `right = last`. Sum too **small** → left is too small even with the current largest → `left++`. Sum too **big** → right is too big even with the current smallest → `right--`. Equal → return `left+1`, `right+1`.

The mix-up: HashMap like **#1 Two Sum**. That is the unsorted cousin (O(n) extra). This problem forbids it and does not need it. Also: two separate `if`s (not `else if`) can move **both** indexes in one step and skip the pair.

---

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **#1 HashMap** | Unsorted array. Store value→index. Extra space O(n). Here the array is sorted and space must be O(1). |
| **Index + worker** | For each `i`, scan `j = i+1…`. Correct, **O(n²)**. Not two pointers. |

**#1 vs #167 in one line:** unsorted + map vs sorted + two ends.

---

#### The walk (learn this)

`[2, 7, 11, 15]`, target `9`

- `2+15=17` too big → throw 15 (`right--`)
- `2+11=13` too big → throw 11
- `2+7=9` → indexes **1 and 2** (1-based)

Keep going while `left < right` (or `!=` if they never cross before a hit). **O(n)** time, **O(1)** extra.

---

#### Cousin

Drop “already sorted” / allow extra space → **#1 Two Sum**. Three numbers that sum to 0 → [#15 3Sum](https://leetcode.com/problems/3sum/) (sort, then this scan for each left).

---

#### Interview sentence

> The array is sorted, so I start at both ends. Too small, I throw the left away; too big, I throw the right away. One pass, O(1) extra space — not a worker that resets, not a HashMap.

---

### LC 125 Valid Palindrome (Easy) — passed

Same **two ends**. After ignoring case and skipping anything that is not a letter or digit, does the string read the same forward and backward?

`"A man, a plan, a canal: Panama"` → true. `"race a car"` → false. `" "` → true. `"0P"` → false.

You **cannot** edit a Java `String`. Do not build a cleaned copy (that is O(n) extra). Skip junk **in place**.

---

#### Why two pointers

Both ends; each step either skips junk on **one** side or compares.

Here, if left is junk → only `left++`. Else if right is junk → only `right--`. Else both are real → if they differ (ignore case) → false; if they match → move **both**.

The mix-up: `else` glued to the **nearest** `if`. Old code: skip left, then `else` still moved **both** — skipped a real letter. Panama failed. **`else if`**: one branch per step.

Second mix-up: `while (left > right)` never runs (loop is backwards). Need `left < right`. `!=` does not stop if they **cross**.

`Character.isLetterOrDigit` — not `isWhitespace` alone (`:` and `,` must skip too). Compare with `toLowerCase` on **those two chars**. `s.toLowerCase()` on the whole string is an extra copy.

---

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **HashMap** | Not “have I seen this key?” |
| **Stack** | Not unfinished openers. Parentheses was push/pop. This is compare ends. |

---

#### Cousin

Cleaned string, no junk → same two ends, no skip. **Longest palindrome substring** → expand around center, not this yes/no scan.

---

#### Interview sentence

> Left and right move in. If a side is not a letter or digit I skip only that side. If both are real and they differ, it is not a palindrome. One pass. O(1) extra if I lower-case the two chars, not the whole string.

---

### Part 3 — OOP + design (interview questions)

These are **real mid-level prompts**: last-seat race, lock vs `synchronized`, two app instances, don’t lock reads, don’t hold a lock across payment, idempotency ≠ auth token, row lock vs connection pool. Same pictures as a ticket/inventory screen. Not trivia. Weekday design ~45 min from today.

Each: they ask … you say … mix-up …

---

**They ask — Who owns `book()`?**
Seats are a field on `Event`. Why isn’t booking a method on `EventService` (or on the entity)?

**You say:** Two jobs. Event service = the concert (title, venue, browse). Booking service = the ticket (who, how many, lock, subtract). Seats living on the event row does not move the use case. URL under `/events` does not either. One class doing both breaks SRP.

**Mix-up:** “The number is on Event so EventService.book().”

---

**They ask — Last seat, two users**
Two attendees book the last seat at once. What goes wrong without a lock? What does each caller get **with** your lock if `book()` is fast?

**You say:** Without a lock both read 1, both **201**, oversell. `@Transactional` is not a wait. With a **row** lock on that event: one **201**, the rest wait, then see 0 → **409**. Timeout is a **cap** for a stuck holder, not what the 99 normally get.

**Mix-up:** “They all get 500 / timeout.” GET is not in that line.

---

**They ask — 409 vs lock timeout**
Would you map a lock wait timeout to **409** like sold out?

**You say:** No. **409** = I saw the row, current state says no → client **stops**. Timeout = I never got the row → seats may still exist → client may **retry**. Same code and they give up on a fake sold-out. Unhandled timeout → **500** today.

**Mix-up:** 409 means “duplicate email” only — no; 409 is conflict with **state**.

---

**They ask — `synchronized` vs two instances**
Why not `synchronized` on `book()`? Two app servers, one database — does the **row** lock still work?

**You say:** `synchronized` lives in **one JVM**. Two servers = two locks = two **201**s. The seat is in **one** DB, so `SELECT … FOR UPDATE` on that row is shared. That is why the lock is in the database, not in Java.

**Mix-up:** “I’ll put a HashSet of event ids on the service.” Same one-process hole.

---

**They ask — Stale UI / lock GET?**
The page still shows 1 seat after someone else booked. User hits Book. Do you lock **GET** so the page cannot be stale?

**You say:** Book returns **409**. The page was an old snapshot; **POST book** is the source of truth. Do not `FOR UPDATE` on GET — browse would stand in the buy line. A GET transaction is milliseconds, not a 30-minute open tab. “I was looking so I own the seat” is a **reservation/hold**, a different design.

**Mix-up:** locking GET to keep the UI honest.

---

**They ask — Payment inside the lock?**
Charge the card (Stripe) while you hold `FOR UPDATE`, then decrement seats?

**You say:** No. The bank call is slow. Everyone else on that concert waits (pool + timeout). Lock = check + write + commit, then pay. If pay fails, put seats **back** and do not leave a successful booking row. That undo is extra work; still better than holding the row across HTTP.

**Mix-up:** “One big transaction including Stripe.”

---

**They ask — Double-click vs book again later**
Does the row lock stop a double-click from creating two tickets? If you add a duplicate check, does the same user get blocked hours later?

**You say:** Lock only **orders** the two POSTs. If seats remain, both **201**. Double-click = two HTTP calls, **one intent**. Fix: a **click id** (idempotency key) — new on each Book press. Same id twice → one booking (**200** with the first, or **409**), never silent drop. Hours later = new click = new id = **201** if seats left. We did **not** add “one booking per user per event” (that would block a real second ticket).

**Mix-up:** “If the JWT is sent twice, reject.” JWT is **who**, on every call (GET, other events, later tonight). Token ≠ click.

Here, this is not implemented yet — interview answer only.

---

**They ask — Connection pool vs row lock**
100 books on event 5. Pool size 10. Is GET event 6 blocked?

**You say:** Two waits. **Row:** only event 5. **Pool:** doors to the **whole** DB. Waiters on `FOR UPDATE` still **hold** a door. ~1 holds the row, ~9 wait on that row with a connection, ~90 wait for a connection. GET 6 does **not** need event 5’s lock. **Free door → GET 6 runs.** All 10 doors full → GET 6 waits at the door. Not a rate limiter we wrote. Not “100 POSTs lock concert 6’s row.”

**Mix-up:** “Other events are always fine” / “the whole app is locked by FOR UPDATE on event 5.”

---

### 60-sec (design)

> I lock the **event row** on book, not the table, not GET, not `synchronized`. Fast last-seat: 201 then 409s. Timeout ≠ 409. Pay after the lock. Double-click needs a **click id**, not the JWT; later tickets still allowed. Hot row can fill the **pool** so GET 6 waits for a door, not for event 6’s lock.

---

## Week 3 Day 2 — Optimistic `@Version`

**Date:** 2026-09-01  
**Goal:** Learn the other **when**. Readers do not wait. The stamp is checked on write. A miss is a real failure (not silent, not sold-out by default).

### Quick recall (Spring — Part 1)

**Optimistic locking (what it is)**

Nobody holds the row while you think. You remember a stamp (version). On write the database runs `UPDATE … WHERE id = ? AND version = old`. If that matches zero rows, someone else already saved. Fail. Retry or tell the client. This is not `SELECT … FOR UPDATE`. This is not `synchronized`.

Here, `@Version` sits on `Event`. Lab `book()` loads with `findById`. Two books can both see seats `1` and version `0`.

The mix-up: calling this “lock one row.” Table vs row is **how much**. Pessimistic vs optimistic is **when**.

**Pessimistic vs optimistic (when — same as Day 1 Q2)**

**Pessimistic** = lock **now**, others **wait**, then you change. **Optimistic** = no wait now; check the stamp **later** on save.

Here, Day 1 was wait on the event row. Day 2 is the stamp on that same row.

The mix-up: pessimistic = whole table. Optimistic = one row. Wrong split.

**Read does not check the stamp**

`findById` / GET is a normal load. Both callers can get the same entity and the same version. Nothing fails yet.

Here, GET and lab `book()` both use `findById`. Both succeed.

The mix-up: “the second caller will not find the row.” That is 404 thinking. The row is still there.

**Write is where it fails**

Hibernate puts the old stamp on the `UPDATE`. Zero rows updated → it throws. That can run at `save` or at commit. Same story: the write, not the read.

Here, `eventRepository.save(event)` (or the commit at the end of `book()`). No extra repository method. `@Version` on the entity is enough.

The mix-up: you must write a special `save` that “checks version.” You do not.

**Two different failures in one method (do not mix)**

Door A — **state you already see** (sold out / not enough stock):

You loaded the row. The field you care about is already too small. **You** throw. The stamp is not involved.

Here, `availableSeats` is `0`, they want `1` → `InsufficientSeatsException`.

The mix-up: using this exception for a stamp miss.

Door B — **stale stamp on write**:

You loaded when the number still looked OK. You passed your own check. Someone else committed first. Your `UPDATE` still has the old version → zero rows → **JPA** throws.

Here, both pass `checkSeats` on `1`. Anton’s save wins (version `0` → `1`). Maria’s save still says version `0`.

The mix-up: “zero rows on `UPDATE`” = `InsufficientSeatsException`. No. She never ran that `if` on a `0`.

**Unhandled vs mapped**

If nothing catches the JPA throw, the client gets **500**. `@RestControllerAdvice` can map that type to a JSON body and a status you choose. The service method does not need a `catch`. The transaction rolls back the whole bucket (including rows you already `save`d in that method).

Here, `ObjectOptimisticLockingFailureException` on `GlobalExceptionHandler` → **409**, title “Stale event version,” not “no seats.” `book()` has no `catch`.

The mix-up: zero-row `UPDATE` is a quiet success. It is not.

**Same 409, two client stories**

**409** means the request is allowed but conflicts with **current server state**. What the client does next depends on **which** state: gone forever vs “try again, it may work.”

Here, Door A 409 (no seats) → **stop**. Door B 409 (stale stamp) → **retry** (stock may still exist). Unhandled Door B → **500**.

The mix-up: every 409 means stop. Same lie as mapping a lock-wait timeout to sold-out 409.

**Which tool for a hot last-item write**

If many people fight over one counter (last ticket, last item), **wait now** (pessimistic) is the usual pick. The loser then reads the real number. Optimistic fits quiet updates (title, profile) where clashes are rare.

Here, last-seat `book()` → still Day 1’s `FOR UPDATE`. The lab used `findById` only so the stamp could clash. `findByIdForUpdate` stays on the repository. Keep `@Version` on `Event` even if you put the wait back.

The mix-up: “optimistic replaced pessimistic.” Two tools, one job.

**Crowd size is not the reason**

A short write stays short at 2 callers or at 100. You pick wait because one writer must see the real number — not because the line is short.

Here, “only two people, little delay” is not the interview why.

The mix-up: switching to optimistic when the crowd grows. The tool follows the job (hot counter vs quiet update), not the headcount.

### What I built

- `Event.version` — `@Version`
- `BookingServiceImpl.book()` — `findById` (lab only)
- `GlobalExceptionHandler` — stamp miss → 409, different title from no seats

### Gate (weak spots)

- **Q1:** Mixed Door B with Door A. Stamp miss = JPA throw, not `InsufficientSeatsException`.
- **Q3:** Picked wait (right). Why was delay / “only two.” Why is: one writer at a time so the other sees the real number.
- **Q4:** 500 unhandled and 409 with handler — correct. Then every 409 = stop. Only Door A is stop; Door B is retry.

### 60-sec

> Optimistic: no wait on read. Stamp checked on write (`AND version = old`). Zero rows → JPA throws, not my stock `if`. Unhandled 500. Advice can map to 409 stale. Sold-out 409 → stop. Stale 409 → retry. Last-item write I still pick wait.

### Part 2 — LC (this day)

**#11 Container With Most Water** — started, then **parked**. Come back this week. Not prefix. Two ends: water = shorter wall × distance; throw the shorter wall away.

**#238 Product of Array Except Self** — passed.

Each slot = everyone else multiplied. No division. Walk once storing “product of people already passed,” walk back multiplying the other side.

Here, `[1,2,3,4]` → `[24,12,8,6]`.

The mix-up: calling this two pointers. It is prefix / running product. **#11** is the ends picture.

**#26 Remove Duplicates from Sorted Array** — passed.

Sorted, so a duplicate sits next to the last unique you kept. One index writes uniques to the front; one scans. Return how many uniques. In place.

Here, `[1,1,2]` → `k = 2`, first slots `[1,2]`. Same write+read shape as #283.

The mix-up: worker that resets. Unsorted → HashSet (#217), not this.

### Part 3 — OOP + design

Each: they ask … you say … mix-up … One idea per question.

---

**They ask — Who owns the stamp? (OOP)**
Why is `version` on `Event`, not on `Booking`? Why is it not on the GET JSON?

**You say:** The class that owns the changing counter owns the stamp. `Booking` is who/how many. GET JSON is the API, not the entity. Client does not send a database column. Server loads in this request.

**Mix-up:** “Event shapes the table” as the OOP answer. Mapping is JPA. OOP is ownership + hide the insides.

---

**They ask — Last seat, first try + retry (stamp)**
1 seat, two POSTs, no wait.

**You say:** First try: one **201**, one **409 stale** (both passed `checkSeats` on `1`). Retry Book: load `0` → Door A **409**, stop. Refresh GET can show `0`. That is browse, not a second Book.

**Mix-up:** wait “then it works.” Last seat still ends **409**. Wait = one HTTP. Stamp = fail fast, then maybe a second HTTP.

---

**They ask — Two seats, one each**
Same stamp, no wait.

**You say:** First try: one **201**, one **409 stale** (not two 201s). Retry: load **1** → pass `checkSeats` → **201**.

**Mix-up:** both 201 on the first try (Friday hole). Retry = `InsufficientSeatsException` (that is the 1-seat retry).

---

**They ask — Two app servers, one DB**
Only the stamp. Same 2-seat story.

**You say:** Still one **201**, one **409 stale**. The `WHERE version = old` runs in **one** database. Which server took the HTTP does not matter.

**Mix-up:** `@Version` is like `synchronized` (per process). `synchronized` is the two-server hole. The stamp is not.

---

**They ask — Organizer changes title**
Same event row, seats still there. Book started on the old stamp.

**You say:** **409 stale**. Retry → **201**. The stamp is on the **whole row**, not only seats. This 409 is not sold out.

**Mix-up:** any version change means no tickets left.

---

**They ask — Who retries**
Server loop inside `book()` vs second HTTP vs refresh.

**You say:** Write can still succeed (title bump, seats left) → a few retries **inside** `book()`, one HTTP, **201**. Write cannot succeed (count is `0`) → Door A **409**, client **stops**. Refresh GET to **see** `0` is fine. Last-seat as the normal path → wait (Day 1), not retries.

**Mix-up:** last seat = ask them to Book again. That is a second Door A. Refresh ≠ retry POST.

---

**They ask — 100 last-seat POSTs, stamp only**
Then the 99 Book again.

**You say:** First wave: one **201**, 99 **409 stale** (99 failed writes). Second wave: 99 Door A **409**. Wait: 99 stay in the first HTTP, see `0`, no second round, far fewer wasted `UPDATE`s.

**Mix-up:** stamp is faster because it locks less of the row. Same row either way. Faster = loser does not wait in a lock line.

---

**Weave-in — booking `save` then stamp miss**

**You say:** `@Transactional` is one bucket. Event write throws → booking insert **rolls back**. No ticket row without a seat change. Retry is a new request, clean.

**Mix-up:** `save` on booking already committed because it ran first.

---

### 60-sec (design)

> Stamp lives on the inventory object, not the API. Last seat, no wait: 201 and 409 stale; retry Book is sold out; refresh GET can show 0. Two seats: retry can be 201. Two servers: still one DB. Title bump: stale 409, seats remain. Retry on the server only if the write can still succeed. 100 last-seat Books: wait, or you pay 99 clashes plus 99 more POSTs. Failed event write rolls the booking back.

---

## Week 3 Day 3 — Two-thread test + wait back + title PATCH

**Date:** 2026-09-02  
**Goal:** Prove two overlapping `book()` calls cannot both take the last seat. Put wait back on `book()`. Quiet title change uses the stamp, not the seat lock.

### Quick recall (Spring — Part 1)

**Which test slice**

`@WebMvcTest` = HTTP layer + mocked service. `@DataJpaTest` = repositories only. `@SpringBootTest` = full app, real beans, real DB. A mock returns what you programmed. It cannot oversell.

Here, `ConcurrentBookingTest` is `@SpringBootTest`. Real `BookingServiceImpl`, real H2, two threads on one event row. `BookingControllerTest` cannot prove this.

The mix-up: calling `book` twice in a controller test with a mocked service.

**Constructor vs field `@Autowired` (when)**

Both are DI. Production bean (`@Service`, `@RestController`) → constructor (one constructor, annotation optional). Spring test class → field `@Autowired` is the usual short form. `@Autowired` on a constructor only if there are two constructors and you mark which one.

Here, `BookingServiceImpl` = constructor. `ConcurrentBookingTest.bookingService` = field `@Autowired`.

The mix-up: treating `@Autowired` vs constructor as opposites. `@Autowired` is the marker; constructor vs field is where the bean goes in.

**`@Transactional` on the test method**

A transactional test wraps the **whole method** in one bucket on the **test thread**. Inserts are not committed until that method ends (then tests usually roll back). Other threads start their own transactions and only see committed rows.

Here, `save` and the two `book()` calls are **inside** `concTest()`. No `@Transactional` on the test. Each `save()` commits; then workers can load the event.

The mix-up: thinking `concTest` finishes, then a later test calls `book()`. The race is in the same method, after the saves, before the method returns.

**Latch vs `join`**

Latch = workers wait for a start gun. `join` = the caller waits until a worker thread has finished.

Here, `CountDownLatch(1)` so both enter `book()` together. `t1.join()` / `t2.join()` so `concTest` does not check seats until both `book()` calls are done.

The mix-up: latch locks the event row. It only starts Java threads together. `join` is not t1 waiting for t2.

**Who is logged in (per thread)**

“Current user” is stored **per thread**. Setting it on the test thread does not copy to workers.

Here, `book()` does `SecurityContextHolder` → email → `findByEmail`. Each worker sets `anton@test.com` before `book()`.

The mix-up: one `@WithMockUser` / one set on `concTest` is enough for background threads.

**What the test proves**

You assert the **invariant**: no extra tickets, stock not negative. You do not always prove *which* failure (stale stamp vs already `0`).

Here, 1 seat → 1 success, 1 failure, seats `0`, booking count `1`. With wait: second loads `0` after the first commits.

The mix-up: “the test proved optimistic locking.” It proved we do not oversell.

**Wait back on `book()`**

Last-item write → wait **now**, then read the real number. Keep the stamp on the row for quiet updates (title). Two tools, one entity.

Here, `book()` uses `findByIdForUpdate` again. `@Version` stays on `Event`. GET still `findById`. Second waiter loads **0** → sold-out 409.

The mix-up: deleting `@Version` because book waits again.

**Title PATCH (quiet write)**

Rename is not the hot counter. Load without `FOR UPDATE`. Stamp is checked on `save`. Clash → 409 stale, not sold out. Retry can be 200.

Here, `PATCH /api/events/{id}`, `EventUpdateRequestDto` (title only), `findById` + `setTitle` + `save`. Organizer/admin. Attendee Book unchanged.

The mix-up: `POST /updateTitle/{id}` (verb in the URL). `FOR UPDATE` on title (puts rename in the seat line).

**Security: first match wins**

Spring uses the **first** matcher that fits, not the most specific.

Here, `POST /api/events/**` for organizer also matched `POST /api/events/5/bookings`. Attendee never reached the bookings line → **403**. Create is `POST /api/events` (no `**`). PATCH is its own line. Bookings matcher stays attendee.

The mix-up: “I also have a bookings line, so Book is fine.” Dead if a broader POST sits above it. Trailing slash `/api/events/` is not `POST /api/events`.

### What I built

- `ConcurrentBookingTest` — `@SpringBootTest`, 1-seat event, two threads + latch + `join`
- `BookingServiceImpl.book()` — `findByIdForUpdate` again
- `PATCH /api/events/{id}` — title only, stamp on save
- `SecurityConfig` — PATCH organizer; POST create exact path; POST bookings attendee

### Gate (weak spots)

- **Q1:** Race is two threads. Slice is `@SpringBootTest` (real `book()` + real H2), not a mocked controller `book`.
- **Q2:** Correct — `@Transactional` on `concTest` → workers do not see uncommitted event.
- **Q3:** Latch = workers wait for start. `join` = **test thread** waits until a worker **finished**.
- **Q4 (fixed after):** Wait → second does **not** pass `checkSeats` on `1`. After commit, second loads **0** → sold-out 409.
- **Q5:** First matching security line wins. `POST /api/events/**` ate Book.

### 60-sec

> Race test = full context + real DB. No `@Transactional` on the test. Latch starts together; `join` before you check. Login is per thread. Passing = no oversell. Last-seat `book()` waits. Title PATCH uses `findById` + stamp. First security match wins — do not use `POST /api/events/**` for organizer.

---

### Part 2 — two LCs — passed

---

#### How to notice “sort then two pointers”

You need **two values that add to a target**, and the list is **not** already sorted.

After sort: too small → throw left; too big → throw right (same as **#167**). Equal neighbors = easy unique pairs/triplets.

**Not this:** unsorted one pair + map OK → **#1 HashMap**. Already sorted one pair → **#167** only (no sort). Two **ends** because of **width** (walls), not a sum → **#11** (do not sort; indexes *are* the x-axis).

**One line:** If you would write “for each `i`, find a pair for a target,” sort first and use two ends — not a worker, not a map.

---

### LC 11 Container With Most Water (Medium) — passed

`height[i]` is a wall at x = `i`. Pick two walls. Water = **shorter wall × distance**. Return the **biggest** water. Not slanted.

`[1,8,6,2,5,4,8,3,7]` → `49` (`8` at index 1 and `7` at index 8: `min(8,7) × 7`).

---

#### Why two pointers

Two ends. Each step **throws one side away**. Width always gets smaller, so you only move if you can get a **taller short side**.

Here, `left = 0`, `right = last`. Area = `min(height[left], height[right]) × (right - left)`. Keep a best. Throw the **shorter** wall (`left++` or `right--`). Equal → move either one.

The mix-up: `left` is not “the min value” and `right` is not “the max value.” They are only indexes. Also: **prefix** (#238) is a running product. **#121** is best buy so far, then sell later. Neither picks two walls.

---

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **Prefix (#238)** | Running product while you walk. No two walls. No width. |
| **Index + worker** | Every pair. Correct, **O(n²)**. No throw-away rule. |
| **Running min (#121)** | Time order, one buy then one sell. Width does not count. |

---

#### The walk (learn this)

`[1, 8, 6, 2, 5, 4, 8, 3, 7]`

- walls `1` and `7`, area `1 × 8 = 8` → throw `1` (short)
- walls `8` and `7`, area `7 × 7 = 49` → throw `7` (short)
- keep going until they meet. Best stays **49**.

**O(n)** time, **O(1)** extra. `while (left < right)` is enough (same index = no water).

---

#### Cousin

Must check every pair / no throw rule → worker, **O(n²)**. Two values that sum to a target, array sorted → **#167** (throw by **sum**, not by shorter wall).

---

#### Interview sentence

> I start at both ends. Area is the short wall times width. Width only shrinks, so I throw the shorter wall away and keep the taller one. One pass.

---

#### Gate (weak spots)

- **Move glued to `else if`:** updating best and moving are **two** steps. Old code only moved when the area was *not* a new best. Tests still passed (next lap moved). Interview: always compute area, **then** throw the shorter wall.
- Names `min` / `max` for indexes — say `left` / `right`.

---

### LC 15 3Sum (Medium) — passed

Return every triplet of **different indexes** whose values add to **0**. Same three values only **once**. Order does not matter.

`[-1,0,1,2,-1,-4]` → `[[-1,-1,2],[-1,0,1]]`. `[0,0,0]` → `[[0,0,0]]`.

---

#### Why sort then two pointers

Freeze one number. You still need a **pair** that sums to `-that`. Sorted pair = **#167**.

Here, `Arrays.sort`. For each `i`: `left = i + 1`, `right = last`. Sum of three: `0` → save, move both; too small → `left++`; too big → `right--`. Skip the same value on `i` / `left` / `right` so duplicates are not recorded twice.

The mix-up: calling the outer `i` a **worker**. A worker walks every `j` with no throw rule (**O(n³)** if a third loop). Inside, you still throw one side per step.

---

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **Index + worker** | For each `i`, every `j`, every `k`. **O(n³)**. Worker **restarts**. |
| **HashMap (#1)** | Unsorted pair. Duplicate triplets are messy (`[0,0,0,0]` must be **one** result). Extra O(n) map. |
| **Neighbors only** | After sort, `j` and `j+1` only. Misses `[-4,1,3]` in `[-4,0,1,2,3]`. First tests were **lucky**. |

---

#### The walk (learn this)

`[-1, 0, 1, 2, -1, -4]` → sort `[-4, -1, -1, 0, 1, 2]`

`i` on **-4**, need pair sum **4**. `left` = next, `right` = last. Never get 4. No triplet.

`i` on first **-1**, need **1**:

- `-1 + 2 = 1` → `[-1,-1,2]`. Move both.
- `0 + 1 = 1` → `[-1,0,1]`

Next `i` is the **second** `-1` → **skip** (same first number → same triplets).

**O(n²)** time. Extra besides the answer list: **O(1)**.

---

#### Cousin

Two numbers, unsorted → **#1**. Two numbers, already sorted → **#167** only. Four numbers → same picture with two frozen indexes, then ends.

---

#### Interview sentence

> Sort. For each `i` I two-pointer the rest for `-nums[i]`. Too small I throw left; too big I throw right. Skip duplicate values so each triplet is once.

---

#### Gate (weak spots)

- **First guess:** index + worker. That is the brute cousin, not the interview pattern.
- **Neighbor scan:** `j` and `j+1` after sort. Five tests passed by luck. `[-4,0,1,2,3]` needs `[-4,1,3]` (not neighbors) → `[]`.
- **Hang:** `left = 0` plus `left != i` on every branch. When `i` is 0, **nothing moves**. `left` must start at **`i + 1`**. Drop `left != i`. Use `while (left < right)`.
- **Skip duplicates:** not in the code yet. Tests use a Set, so extra `[0,0,0]` copies can still pass. Interview: skip same value; do not `contains` on the result list.
- Drop `System.out.println`.

---

### Part 3 — OOP + design

**Map:** [oop-design-map.md](oop-design-map.md). Do not repeat SRP / who owns `book()`.

**Encapsulation**

Hide inventory insides. The client sends the change they are allowed to make. Seats and stamp stay on the server.

Here, `EventUpdateRequestDto` is **title only**. Organizer must not send `version` or `availableSeats` (fake `999` stock, or a fake stamp).

The mix-up: this is not SRP. SRP = which class owns Book vs title. Encapsulation = the body cannot set those fields.

**Java threads**

Latch = start gun (`await` until `countDown` to 0). `join` = **the caller** waits until that worker **finished**. `synchronized` = this JVM only. `ExecutorService` = a pool; you **submit** work instead of `new Thread()`.

Here, both workers `await()`, test `countDown()`, then both `book()`. `concTest` does `t1.join()` / `t2.join()` before assert. `synchronized` on `book()` + two app servers still oversell — lock is the DB row.

The mix-up: `join` means t1 waits for t2. Latch locks the event row.

**Two tools, one row**

Pick the tool by **job**. Hot counter → wait now. Quiet field → stamp on save. Same entity can use both.

Here, `book()` = `FOR UPDATE`. Title PATCH = `findById` + `@Version`. **Name who commits first.** Book first → **201** + **409 stale** (retry PATCH can be 200). PATCH first → **200** + Book **201** (seats still 1).

The mix-up: organizer 409 = sold out. PATCH with `FOR UPDATE` (rename stands in the seat line).

**Isolation (see vs write)**

Default reads can both see `1` while two transactions are open. `@Transactional` is not a wait.

Here, keep two pictures apart. `findById` lab → both **see** `1`; version does **not** stop the read (it fails the **save**). `FOR UPDATE` → second **waits**, then **loads `0`**. No refresh.

The mix-up: version stops the read. Refresh stops the race. Mixing the lab (`findById`) with production (`FOR UPDATE`) in one answer.

**PUT vs PATCH + click id**

PATCH = only the fields you send. PUT = full replace (omit a field → wipe or you must send the whole resource). POST create is **not** safe to retry unless you send a click id.

Here, title is PATCH. Two Book clicks, no click id → **two** attempts (two bookings if seats exist). JWT is who you are, not which click.

The mix-up: JWT twice = same click.

**403 ≠ 409 (first matcher)**

First security line that fits wins. **403** = we know you, you may not. **409** = `book()` ran, state clash (sold out or stale).

Here, `POST /api/events/**` above bookings also matches `POST /api/events/{id}/bookings`. Attendee never reaches the attendee line → **403**. `book()` never ran.

The mix-up: 403 means sold out.

### Gate (Part 3 weak spots)

- **`join`:** thought t1 waits for t2. `join` = **test thread** waits until that worker finished.
- **Isolation:** first said version / refresh stop the read. Version stops the **write**. Refresh is GET.

### 60-sec (Part 3)

> PATCH body is title only — client does not set seats or version. Latch starts together; `join` is the test waiting, not t1 waiting for t2. Java `synchronized` is not the seat lock. Book waits; title uses the stamp — say who commits first. `findById` both see 1; `FOR UPDATE` waits then sees 0. Two Book clicks without a click id = two attempts. `POST /api/events/**` above Book → 403, not 409.

**Calendar:** Day 3 closed. Day 4 = WebClient + HTTP to Event. **Fri:** long HLD. **Sat/Sun off.**

---

## Week 3 Day 4 — Booking asks Event over HTTP

**Date:** 2026-09-03  
**Goal:** Seats are no longer changed inside `book()`. Event has its own HTTP door. Booking calls that door, then saves only the ticket.

### Quick recall

In any Spring app, once two pieces stop sharing one method, they also stop sharing one database “bucket.” You talk over HTTP. That is a new request. It commits on its own. The caller cannot roll that work back.

Here, `book()` used to lock the concert, check seats, save the ticket, and subtract — all in one go. Now Event owns the concert row. Booking sends POST `/api/events/{id}/seat-reservations` with how many seats. Event locks, checks, subtracts, commits, and lets the lock go. Only then does Booking save the ticket.

The mix-up: thinking Booking’s `@Transactional` still wraps the seat change. It does not. Event already finished.

`WebClient` is the HTTP client. Make it one bean and reuse it. Do not `new` a client on every Book. The mix-up: treating `WebClient` like a throwaway helper inside `book()`.

`WebClient` does not wait by itself. `.block()` means: sit here until Event answers. Without that, you might save a ticket before you know if a seat was taken.

The Book click sends a Bearer token. The call to Event is a **second** request, so it has no token unless you copy `Authorization`. Event’s JWT filter runs before Event’s controller, same as always. The mix-up: “we already know the user in Booking, so Event is logged in too.”

What Event returns is JSON (`EventResponseDto`). That is not a table row. Do not `new Event()` from it. You only need “this ticket belongs to event 5” (`findById` or `getReferenceById` while we still have one database). The mix-up: copying JSON fields into a new `Event` and calling that the entity.

### What I built

- `WebClient` bean and `event.service.base-url`
- `reserveSeats` on Event + POST `.../seat-reservations`
- `EventClient` (copy token, wait for the answer)
- `book()` calls Event, then saves the booking only

### 60-sec

> Booking asks Event for seats over HTTP. Event locks and commits. Booking only saves the ticket. Copy the token. Don’t turn the JSON into an Event entity.

### Gate (weak spots)

**Q1:** `WebClient` is Booking’s client to call Event. It is not Event’s bean.

**Q3:** Booking’s transaction does not undo Event’s seats. Event already committed. You need a later undo or hold. We did not build that.

**Gate is in the chat.** Next weekday: Fri HLD. Sat/Sun off.
