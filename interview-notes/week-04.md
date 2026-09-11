# Interview notes — Week 4

Microservices split: 3 services, WebClient, correlation IDs.

**Packed:** **two** Spring topics each Mon–Thu. Friday = one HLD. Full calendar: [part2-map.md](part2-map.md).

**Part 1:** two coding LCs, then **~15 min talk** from [System Design for Interviews and Beyond](https://leetcode.com/explore/interview/card/system-design-for-interviews-and-beyond) Mon–Thu ([lc-sd-map.md](lc-sd-map.md)). Friday = coding LCs only. Always name **Chapter N + topic**. From Tue: explain a bit, then **question** (Anton talks first).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Booking calls Event over HTTP | `WebClient` bean | **Pulled to W3 Thu** |
| Tue | Third service (auth/users) | Correlation-id header | **Done W4 Day 1** |
| Wed | Downstream 4xx/5xx mapping | Client timeout | **Done W4 Day 3** |
| Thu | Remaining split glue | One integration test for the call | **Done W4 Day 4** |
| Fri | HLD of the three boxes | — | **Done W4 Day 5** |

---

## Week 4 Day 1 — Consecutive run + window with k changes + rate limit

**Date:** 2026-09-07  
**Goal:** Two coding LCs + Part 1 Design talk. Spring / Part 3 still open (auth/users + correlation-id; seats over HTTP).

Coding source from today: **Top Interview 150 + Blind 75 only**. #424 was NeetCode-only — last off-list pick.

### Part 1 — two LCs — passed + Design talk

---

#### How to notice “HashSet for a value-run” (#128)

You need the longest run of numbers that **belong next to each other by value** (`1,2,3,4`), and the list is **jumbled**. Time must be **O(n)** — sort is too slow.

Put every number in a set. Only **start** a walk if `x-1` is **missing**. Then walk `x+1`, `x+2`, … Keep the max length.

**Not this:** sort then scan (right answer, wrong time). Sliding window (contiguous **indexes**). One global counter of “has a neighbor” (two runs of 2 become 3). Walking down to `0` (negatives die).

**One line:** Start only at the beginning of a run; consecutive means value ± 1, not “toward zero.”

---

### LC 128 Longest Consecutive Sequence (Medium) — passed

Unsorted `nums`. Return the **length** of the longest consecutive value-run. `[100,4,200,1,3,2]` → `4`. `[1,2,10,11]` → `2` (two runs). `[-1,0,1]` → `3`. Empty → `0`.

Budget: **O(n)** time, extra **O(n)** fine.

First code counted every `x+1` into **one** counter. Tests with a single run passed; `twoSeparateRuns` expected 2 got 3. Next try walked down to 0 and skipped `x>0` — missed negatives.

#### Why HashSet

Need “is this **value** in the bag?” Start a run only when `x-1` is missing so each number is walked a constant number of times (`[1..n]` from every start is O(n²)).

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **Sort then scan** | Works. **O(n log n)**. This round forbids it. |
| **Sliding window (#3)** | Contiguous **in the array**. Here 1 and 2 can sit far apart. |

#### Cousin

Drop the O(n) cap → **sort then scan**.

#### Interview sentence

> Put every number in a set. Only start counting when `x-1` is missing, walk forward while `x+1` exists, keep the longest run.

#### Gate (weak spots)

- Sort is the close wrong pattern (allowed if they drop O(n)).
- Ends-only / one global `longestSeq++` on every neighbor pair.
- `0` as the end of a sequence.

---

### LC 424 Longest Repeating Character Replacement (Medium) — passed (off-list)

`s` uppercase, at most `k` changes. Longest **contiguous** stretch that can become one letter.

`"ABAB"` k=2 → `4`. `"AABABBA"` k=1 → `4`. `"BAAAB"` k=2 → `5` (keep the A’s, change two B’s).

Budget: **O(n)** time, **O(1)** extra (26 boxes).

**Off-list:** NeetCode 150, not Top 150 / Blind 75. Do not pick another like it.

Anton locked the stretch to `s[left]`, spent `k` on mismatches, jumped `left`/`right`, reset length. Ends matching hid a letter in the **middle** (`AABA`: both ends A, B unpaid).

#### Why sliding window

Contiguous slice of **one** string + a budget of k changes. `right` grows. Counts in `[left,right]`. Illegal when `length - most common letter > k`. Then move `left` one, drop that letter’s count. Do not restart. Do not treat `s[left]` as “the” letter (`ABBB` keeps B).

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **#3** | Peel on a **repeat**. Here repeats are the goal; peel when changes needed > k. |
| **#128** | Value exists **anywhere**. Here letters must sit next to each other in `s`. |
| **#53** | Close this stretch, open a new one. Here `right` never goes back. |

#### Interview sentence

> Grow a window; it is legal while `length - most frequent letter ≤ k`; if not, drop from the left.

#### Gate (weak spots)

- Restart / `tmpmaxlen = 0` / `left = right`.
- `s[left] == s[right]` only sees two ends; need `int[26]` for the slice.
- “Ignore k times to keep the first letter” fails `"BAAAB"`.

---

### Part 1 Design — Chapter 11 How to protect servers from clients — Rate limiting

**Not a drill today.** The board was walked for him. There is **no Design weak-spot list** — he did not answer a question. From Tue: explain a bit, then ask.

Talk. No Java. Same course card as [System Design for Interviews and Beyond](https://leetcode.com/explore/interview/card/system-design-for-interviews-and-beyond). Not the Design **tag** (that is code: LRU).

#### What this chapter is

**Protect servers from clients** means: one person (or a bot) must not be able to melt the API by sending the same call as fast as they can.

**Rate limiting** is the tool: you allow only **N calls per key per time window**. Extra calls are refused **before** `book()` or login runs.

It is **not** sold out. Sold out is “this concert has 0 seats.” Rate limit is “you already asked too many times.” Different doors.

#### How it works (plain)

1. Pick a **key**: user id if they are logged in, otherwise IP.
2. Pick a **window**: e.g. 10 `POST /api/bookings` per minute, tighter on `POST /api/auth/login`, looser on `GET /api/events`.
3. Each allowed call **adds 1** to that key’s counter.
4. If the counter is already at N → **do not** run the handler. Return **429 Too Many Requests** (try later).
5. When the window ends, the count goes back toward 0 (or you use a sliding window — same idea).

The counter must live in **one shared place** (gateway or Redis). If each Booking **pod** keeps its own count in memory, two pods = double the cap. That is the 10× point.

Do **not** use `FOR UPDATE` on the event row for this. That lock is seats. This is “too many HTTP calls.”

#### Status (learn this)

| Code | Meaning here |
|---|---|
| **429** | Too many calls. Wait. The concert may still have seats. |
| **409** | Valid call, **state clash** (no seats / stale). They were allowed in. |
| **401** | We do not know who you are. Not a rate-limit. |

Trap if they mix them: every 429 = sold out. Wrong.

#### Interview sentence

> I rate-limit at the edge with a shared counter. Over the cap is 429. Sold out stays 409.

#### Next Design talks

Explain the topic a bit → then the **interview question**. He talks. No dump. Always **Chapter N + topic**.

---

### 60-sec (Part 1)

> #128: set, start only if `x-1` missing, walk forward, max run. #424: window, `length - maxFreq ≤ k`, peel left. Ch 11: shared counter, 429 ≠ 409.

**Weak spots (coding only):** two runs / one counter. Window restart / ends-only. Design was not practiced.

**Calendar:** Part 1 closed. Part 2 Spring **done**. **Next:** git, then Part 3 — seats over HTTP / slow hop. Do not re-ask Strategy.

---

### Part 2 — Auth over HTTP + correlation id

**Date:** 2026-09-07

#### Users live in Auth, not in Booking

If two services each have their own database, Booking cannot open the users table. It asks Auth over HTTP. Auth answers with JSON (id, email, role). That JSON is **not** a `User` row — no password hash, not something JPA should save.

Here: `book()` used to `findByEmail`. That was Auth’s table. Now `GET /api/users/me`. Booking takes the **id** from the JSON. We still have one H2, so `findById` only hangs the FK (same leftover as Event after seats HTTP). Later Auth’s own DB: just store `userId`, no `User` in Booking.

What I mixed up: `new User()` from the DTO. Missing hash. Fake object. Don’t copy JSON into an entity.

#### Token on the second call

A WebClient call is a **new** request. Auth does not see that Booking already logged in. Copy `Authorization` or Auth returns **401**.

`/me` is not `permitAll`. Register/login have to be open (no token yet). `/me` falls through to “everything else needs a login.” I had put `GET /api/users/**` as `permitAll` — that would let `/me` run with nobody logged in (crash/500, not 401).

What I mixed up: copy the token “to get the email.” I already had the email. Copy it so **Auth** has a token. Internal ≠ skip JWT.

#### Correlation id = log sticker

One string per Book tap. If the caller already sent `X-Correlation-Id`, keep it. If not, mint a UUID. Copy it onto Auth and Event. Grep that string, see the whole tap.

It does **not** change seats, who you are, or 409. Code only creates / keeps / copies it. Put it on the response so you can quote it.

A UUID you mint is **not** on `getHeader` — only on the attribute you set. That’s why clients read the attribute. Forget to copy: Auth mints a **second** id. App still works. **No throw.** Logs just don’t match.

Filter runs **before** JWT so a 401 still has an id.

### 60-sec

> Booking asks Auth who this token is over HTTP. JSON is not a User — take the id. Copy the Bearer; new request is not logged in. Correlation id is a sticker for logs, not business. Forget it → another UUID, no error.

**Weak:** token copy ≠ fetch email. `/me` = `anyRequest().authenticated()`. Missing sticker does not throw.

---

## Week 4 Day 2 — Shortest stretch ≥ target + nearby duplicate + timeout/retry

**Date:** 2026-09-08 (Tue)  
**Tired cut:** Anton asked 1 coding LC + 1 design. He then did a second Easy (#219) and chose **Part 1 LC Design** (not Part 3).

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LeetCode in `leetcode-practice` | #209 + #219. **Done.** |
| **Part 1 Design (LC-SD)** | ~15 min **talk** from the LeetCode course card, after the coding LCs. **Still Part 1.** Not Spring. Not LRU code. | **Chapter 8.** **Done.** |
| **Part 2** | Spring in `event-booking-platform` | **Nothing done today.** Opener only — no code, no mapping, no timeout. Tue topics (auth HTTP + correlation-id) were **Done** on Day 1. **Next Spring = Wed, from scratch:** downstream 4xx/5xx + client timeout. |
| **Part 3** | OOP + **this-app** design (~45 min) | **Not today.** Map: correlation id as a **design talk** + Adapter. Also leftover from Mon: seats over HTTP / slow hop. |

Do **not** call Chapter 8 “Part 3.” Part 3 is a later block the same weekday.

---

### Part 1 — two LCs — passed

---

#### How to notice “shortest stretch that clears a floor” (#209)

Positive numbers. Contiguous. Shortest length whose **sum ≥ target**. `n ≈ 10⁵` → nested every-stretch is too slow → **O(n)**. Extra space **O(1)**.

`right - left + 1` = how many cells when **both ends count** (6..9 is 4). Record that **before** `left++`.

**Memorize this:** outer `for right` (grow / add). Inner `while` peels `left` (drop). `left` has no own `for`.

**Not this:** prefix array + binary search (right math, **O(n log n)** follow-up). Kadane (#53) = best **sum**, not shortest **length**. Two ends (#167) = throw a side of a sorted pair.

#### LC 209 Minimum Size Subarray Sum (Medium) — passed

`target = 7`, `[2,3,1,2,4,3]` → `2` (`[4,3]`). No stretch → `0` (keep `best` as `MAX_VALUE`, then return 0).

First code moved `right` inside the peel loop and never saved `minarr`. Inner must be `while (sum >= target)`: record length, `sum -= nums[left]`, `left++`. One right-tick can peel 0, 1, or many.

Cousin: numbers may be **negative** → window is not monotonic → prefix, not this peel.

**Interview sentence:** Positive numbers, grow right, peel left while the sum is already enough — each index in and out once, O(n).

---

#### LC 219 Contains Duplicate II (Easy) — passed

Same value at `i` and `j` with `|i-j| <= k`. Not #217 (duplicate **anywhere**).

**Memorize this:** one `for i`. Map **value → last index so far**. If seen and `i - last <= k` → true. Then **always** `put(value, i)`.

Trap: fill the map with **last** indexes first, then scan. `[1,1,3,4,5,1]`, `k=1` is true (indexes 0 and 1) but last `1` is at 5, so that two-pass misses. Check **then** overwrite.

Cousin: drop `k` → #217 HashSet, no indexes.

**Interview sentence:** Store the last index of each value; a hit counts only if this index is within `k`.

---

### Part 1 Design — Chapter 8 How to deliver data reliably — Timeout / retry / idempotency

**This is Part 1.** Same course: [System Design for Interviews and Beyond](https://leetcode.com/explore/interview/card/system-design-for-interviews-and-beyond). Talk only. No Java.

It is **not** Part 3. It is **not** the correlation-id sticker (that is logs; already built in Spring Day 1).

Anton **did** answer (unlike Day 1 Ch 11). First answer: timeout on Event, retry X times, stop on success. That is right for a **read**. Trap is **Book**.

#### What this chapter is (three tools, one lie)

The network can say “no answer” **after** Event already took the seat. You need three words:

1. **Timeout** — stop **waiting**. You did not undo the other side.
2. **Retry** — call again. Safe on a **GET**. Dangerous on **Book** unless the same click is tagged.
3. **Idempotency** — same click id = **one** intent. Event applies it once.

#### Where the timeout lives (this was the mix-up)

Put the timeout on **Booking**, on the **HTTP client / WebClient** that **calls Event**.

- Booking is the one waiting for a response.
- “Timeout on Event” as the main answer is vague. Event can still **finish the write** after Booking hung up.
- Timeout ≠ Event failed. Timeout ≠ seat is free. Timeout = **we stopped listening**.

#### What to retry

| Call | Retry? |
|---|---|
| GET / browse seats | Yes. A few times. Read-only. |
| `book()` / take a seat | **Only with the same click id.** |
| Sold out **409** | **No.** Stop. |

Blind “retry Book X times until 200” can take **two** seats if the first call succeeded and Booking only saw a timeout.

#### How click id fixes it (UUID in Event’s DB)

The **browser** mints one UUID for **that Book tap**. Booking sends it every try of **that** tap. A later tap = **new** UUID.

**Event** stores that id **with the result** when the seat is taken (unique constraint so two in-flight retries cannot both insert).

On retry:

| DB has this click id? | Meaning | What Event does |
|---|---|---|
| **Yes** | First call **did** finish. Booking only lost the HTTP answer. | Return the **same** result. Do **not** take another seat. |
| **No** | Event never applied it (or not yet). | Take the seat **once**, then save the id. |

That is idempotency. JWT is **who**. Click id is **which tap**. Correlation id is **which log line**. Do not mix them.

#### Status (don’t mix with Ch 11)

| Code | Here |
|---|---|
| Timeout / no body | Booking does not know. Retry Book **only** with click id. |
| **409** sold out | State clash. **Stop.** Not “try again.” |
| **429** | Too many calls (Ch 11). Not this chapter. |
| **201** / same payload on replay | Click already applied. Success. Stop. |

#### Interview sentence

> Timeout means Booking stopped listening, not that the seat is free. I retry `book()` only with a click id stored on Event.

#### Gate (weak spots)

- Timeout “on Event” instead of on Booking’s HTTP wait.
- Retry Book X times with **no** click id.
- Click id = JWT (“same user, so reject”). Hours later is a **new** tap.
- Click id = correlation id (logs vs intent).
- 409 sold out → retry anyway.

---

### 60-sec (Part 1)

> #209: window, add `right`, peel `left` while sum ≥ target, length = `right-left+1`, else 0. #219: map value→last index, check `i-last<=k` then put. Ch 8: timeout on **Booking’s** call; GET may retry; Book only with click id in Event DB; timeout ≠ Event failed.

**Weak:** peel `right` / two-pass last-index map. Design: timeout belongs on the **caller**.

**Calendar:** Part 1 **closed** (coding + Ch 8). Part 2 **nothing done** (no `EventClient` change). Part 3 **not today**. **Next:** W4 Wed — LC first, then Spring from scratch (map Event 4xx/5xx + client timeout), then Part 3 leftover.

---

## Week 4 Day 3 — Sudoku go-over + isomorphic map + RandomizedSet + sync vs queue

**Date:** 2026-09-09 (Wed)

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LeetCode in `leetcode-practice` | **#36** go-over (not a grind). **#205** he coded. **#380** he coded. Skipped **#76** Hard. Skipped **#383** as a second Easy. **Done.** |
| **Part 1 Design (LC-SD)** | ~15 min talk from the course card. **Still Part 1.** | **Chapter 3 + Chapter 5** — sync vs queue. **Done.** |
| **Part 2** | Spring in `event-booking-platform` | Map Event 409/404/5xx + 3s wait on the `WebClient` bean. **Done** (same day, second chat). |
| **Part 3** | OOP + this-app design | Tue leftover **done** this chat: Adapter + correlation-id talk. Wed OCP + statuses still open. Mon leftover: seats over HTTP. Do **not** rerun the queue talk as Part 3. |

Do **not** call Chapter 3/5 “Part 3.” Full outbox / at-least-once mail is **W6 Mon** (still Ch 5, deeper).

Coding lists: Top 150 + Blind 75. #76 is on both — skipped because Hard. #36 and #380 are Top 150. #205 is Top 150 (not Blind 75).

---

### Part 1 — LCs

---

#### LC 36 Valid Sudoku (Medium) — go-over (Anton did not grind)

Not a puzzle to solve live. Pattern is **HashSet**: “have I already seen this digit in this group?”

One pass. Skip `'.'`. For each filled cell, three tickets into **one** set:

- `"5 in row 0"`
- `"5 in col 3"`
- `"5 in box 1"` — `box = (row / 3) * 3 + (col / 3)`

`add` returns false → duplicate in that group → invalid. **Do not solve** the grid.

**Memorize this:** one pass; skip dots; one set; three keys (row / col / box).

**Cousin:** fill the empties → backtracking, not this.

**Interview sentence:** I don’t solve it — I stamp each filled digit into its row, column, and box; a second stamp is invalid.

---

#### How to notice “translation, not tally” (#205)

Two strings, same length. Each `s` letter glues to **one** `t` letter, and no two `s` letters share a `t` letter. Walk the **same index**.

**Not this:** counts / “how many times” — that is **#242**. `"abab"` / `"aabb"` have the same counts and are still **false**.

**Memorize this:** map `s[i] → t[i]`. If `s[i]` already mapped, it must still be this `t[i]`. A `t` letter is taken by at most one `s` letter (second map, or “taken” set). `"foo"` / `"bar"` = same letter, two partners. `"ab"` / `"aa"` = two letters, one partner.

#### LC 205 Isomorphic Strings (Easy) — passed

Anton named HashMap + O(n). First “why” was counts — corrected before code.

His code: one `HashMap<Character,Character>` + `containsValue` for the reverse rule. Tests pass (including `"abab"` / `"aabb"`).

Interview note: `containsValue` is O(n) per step → **O(n²)**. Two maps (`s→t` and `t→s`) is **O(n)** and what you say out loud.

**Cousin:** drop the taken check → `"ab"` / `"aa"` wrongly passes.

**Interview sentence:** I glue each `s` letter to one `t` letter and refuse a second glue in either direction.

---

#### LC 380 Insert Delete GetRandom O(1) (Medium) — passed (shape)

Set **behavior** (insert twice is still one). Java `HashSet` cannot `getRandom` in O(1).

**Memorize this:** **List** = values (random index). **HashMap** = `value → index`. Insert: append + record index. Remove: swap victim with **last** slot, update **that one** moved value in the map, drop tail. `getRandom` = `list.get(random.nextInt(size))`. `getRandom` does **not** remove.

Do **not** `list.remove(0)` and rewrite the whole map. Do **not** `list.contains` / `indexOf` (O(n) — undoes the problem). Check = `map.containsKey`.

Anton mixed count-map (Ransom Note) and “remove 2 twice.” This problem stores **no duplicates**. Second `insert(2)` is `false`. One `remove(2)` clears it. Bag-with-counts is a follow-up (`RandomizedCollection`).

His tests passed. Weak: still `list.contains` on insert/remove — **fix to `map.containsKey`**. Last-element special case is optional (swap-with-last works on the tail too). Duplicate insert must return **false**.

**Cousin:** allow duplicates → map becomes `value → list of indexes`.

**Interview sentence:** List for O(1) random pick; map for O(1) “where is this value”; remove is swap with the last slot so I never shift.

---

### Part 1 Design — Chapter 3 Foundations of reliable, scalable, and fast communication + Chapter 5 Why queues matter in distributed systems — Sync vs queue

**This is Part 1.** Same course card. Talk only. No Java. Not Kafka.

Prompt: `POST /api/bookings` takes a seat, then the user must get a confirmation email. Sync on the request, or queue?

#### What Anton said

Email is slow → extra wait for the **client**. Put that work on a **queue** so it can be delayed and the client is not charged that time.

Then he sharpened it: **booking is fully done** → call enqueue → return **201**. Do not wait on the **email send**. He also said do not wait on the enqueue **response**.

First half is right. The enqueue-ack half is the trap.

#### Sync vs queue (why the chapter exists)

**Sync** = this request waits until the other side finishes (HTTP to Event, SMTP). Fine when the hop is short.

**Queue** = write down “send this mail,” return. A worker does Gmail later. `201` is “we took the seat,” not “inbox has the mail.”

Chapter 3: communication has a cost (thread, lock, client time). Chapter 5: a queue so you stop paying the **slow** hop on the user request.

#### Order (memorize)

1. Take the seat. **Commit.** Release the DB door.
2. Enqueue the mail job (cheap). Wait for **job accepted**, or write the job in the **same DB commit** (outbox — W6).
3. Return **201**.

Never: send mail (or call Gmail) **inside** the seat lock. Never: hold `FOR UPDATE` while SMTP runs.

#### Wait vs don’t wait

| Hop | Wait on `book()`? |
|---|---|
| Gmail / SMTP / worker send | **No.** That is the slow hop. |
| “Job is on the queue” (ack) or outbox row in the same commit | **Yes** (or same transaction). Cheap. |
| Fire-and-forget enqueue, then `201` with no ack | **No.** Network blip → seat taken, **no job**, no mail, client already got success. |

#### Status

| Code | Meaning here |
|---|---|
| **201** | Seat saved. Mail may still be in the queue. |
| Mail not in inbox yet | Not a failed book. Worker lag. |
| Enqueue failed after commit | Booking exists; mail job missing → retry/outbox (W6), not “pretend 201 and drop it.” |

Do not invent Kafka in this slot. “A queue” is enough.

#### Interview sentence

> Offload the slow hop; never hold the DB door across email. `201` after the seat is saved; wait on enqueue ack (or outbox), never on the mailbox.

#### Gate (weak spots)

- Email (or any slow hop) **inside** the seat lock.
- `201` means Gmail already sent.
- Fire-and-forget enqueue with no ack / no outbox.
- Calling this Part 3. Outbox drill is **W6 Mon**.

---

### 60-sec (Part 1)

> #36: one set, three keys (row/col/box), skip dots, don’t solve. #205: map `s→t` and refuse a taken `t`; not counts. #380: list + map `val→index`; remove = swap with last; `containsKey` not `list.contains`. Ch 3+5: commit book, enqueue, 201; don’t wait on Gmail; do wait on enqueue ack or outbox; never lock across mail.

**Weak:** #205 `containsValue` O(n²). #380 `list.contains`. Design: “don’t wait for enqueue” ≠ drop the job.

**Calendar:** Part 1 **closed** (coding + Ch 3/5). Part 2 **closed** (see below). Part 3 that day **done** (Adapter + correlation id). LC Thu is **Day 4** (#383 + #73 + Ch 12).

---

### Part 2 — Map Event’s HTTP answers + wait cap on the client

**Date:** 2026-09-09 (Wed, after Part 1)

#### Downstream status is a new HTTP response

When service B returns 4xx/5xx, the caller’s HTTP client does **not** turn that into B’s Java exception. `retrieve()` throws. If you do not map it, the phone sees **500** (caller crashed), not B’s 409/404.

Here: Event already returns 409 for sold out and 404 for a missing event. `EventClient.reserveSeats` `.block()`s. Catch `WebClientResponseException`, read `getStatusCode()`, throw **our** types so the existing handler can speak: 409 → `InsufficientSeatsException`, 404 → `ResourceNotFoundException`. Map in the **client**, not in `book()`, not on the shared `WebClient` bean (Auth’s 401 would share that bucket).

The mix-up: Event’s handler already ran. That does not help the phone. The phone talks to **Booking**.

#### 5xx from Event is not sold out

Event **500** means Event broke. Booking **409** means no seats. Mixing them makes the user stop as if the concert is full. Booking’s door for “the other service failed” is **502**.

Here: `is5xxServerError()` → `DownstreamServiceException` → handler **502**. Else `throw e` so 401 is not swallowed into `null`.

The mix-up: 5xx → seats exception.

#### Timeout = we stopped listening, not “Event wrote nothing”

A timeout is **no HTTP status**. Different throw: `WebClientRequestException`, not `WebClientResponseException`. 409 = we **saw** current state (0 seats). Hang-up = **we don’t know**. Event is another process; Booking’s clock does not undo Event’s row. Event may already have subtracted, still be in the lock, or never have started.

Here: `HttpClient.responseTimeout(3s)` plugged into the `WebClient` bean via `ReactorClientHttpConnector`. Catch request-exception → `DownstreamServiceException` (same 502 for now, not seats). Cap is on the **caller**, not on `@Transactional`.

The mix-up: timeout = nothing stored in the DB. That is the lie. Same as Ch 8: hang-up ≠ Event failed.

### 60-sec (Part 2)

> Event 409/404/5xx is a new response; `retrieve()` throws; map in `EventClient` to our exceptions (409 seats, 404 missing, 5xx → 502). Timeout is no status — cap on the `WebClient` engine; not 409; we do not know if Event wrote.

**Weak:** timeout = “DB stored nothing.”

**Calendar:** Part 2 **closed**. Part 3 Adapter + correlation-id talk **done** (same day).

---

### Part 3 — Adapter + correlation id (Tue leftover)

**Date:** 2026-09-09 (Wed, after Part 2)

#### Adapter

`EventClient` **uses** `WebClient`. It is not a WebClient. `book()` says take seats. HTTP (`post` / `uri` / `.block()` / status map) stays in the client. Moving that into `BookingServiceImpl` is two jobs on one class. It does **not** un-split the microservices — Event is still another HTTP door.

The mix-up: HTTP in `book()` = we failed the service cut.

#### Correlation id

General: one string per incoming request so logs across services grep as **one tap**. Keep the header if the caller sent it; else mint. Copy it on the next hop. Put it on the response so the **frontend can see** that same string.

Here: `CorrelationIdFilter` (before JWT). `EventClient` / `AuthClient` copy the attribute. Forget to copy → Event mints a second id. Book still works. Logs do not stitch. **No throw.** 401 still has the sticker because the filter ran first.

JWT = who. Click id = which tap (not in the app). Correlation id = log sticker. Two real clicks, same user = two tickets.

The mix-up: click id = “monitor.” Returning the sticker = “make the call unique.”

### 60-sec (Part 3)

> Adapter: `EventClient` uses `WebClient`; `book()` does not speak HTTP. Correlation id: keep or mint, copy, return for the phone to see; forget copy → two ids, no crash. Filter before JWT so 401 still has the sticker.

**Weak:** timeout = Event wrote nothing (Part 2). Click id vs sticker. Returning the id ≠ unique tap.

**Calendar:** Day 3 **closed**. Day 4 Part 1 **done** (see below). Part 2/3 that day still open at the time.

---

## Week 4 Day 4 — Ransom counts + matrix zeros + circuit breaker

**Date:** 2026-09-10 (Thu)

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LeetCode in `leetcode-practice` | **#383** he coded. **#73** he coded (Medium). Deleted **#290** (second Easy). **#76** still skipped. **Done.** |
| **Part 1 Design (LC-SD)** | ~15 min talk from the course card. **Still Part 1.** | **Chapter 12 — How to protect clients from servers** (circuit breaker). **Done.** |
| **Part 2** | Spring in `event-booking-platform` | Remaining split glue + one HTTP integration test for Booking → Event. **Done.** |
| **Part 3** | OOP + this-app design | **LSP** (fallback ≠ **201**) + **what the test proved** (HTTP integration vs mock). **Done.** OCP leftover → Fri small OOP. Do **not** rerun the circuit-breaker talk as Part 3 (that board is **W5 Wed**). |

Do **not** call Chapter 12 “Part 3.”

---

### Part 1 — two LCs — passed + Design talk

#### LC 383 Ransom Note (Easy) — passed

Build `ransomNote` from `magazine`; each letter at most once. Extra copies in the magazine are allowed.

First check required **equal** counts (`!=`) — that is **#242 Anagram**. Fail case: `"aa"` / `"aaa"` must be `true`. Fix: note count **≤** magazine count.

Two maps worked. Interview shape: one bag from magazine, spend on the note.

#### LC 73 Set Matrix Zeroes (Medium) — passed

On a `0`, zero that whole row and column. In place = mutate the given matrix, **not** “no extra HashSet.” Two sets of poisoned rows/cols (or stored `(i,j)` pairs) then a second wipe. Do not wipe while scanning.

`O(1)` extra is the **follow-up** (marks in row 0 / col 0). HashSet is **O(m+n)**. Sliding window is wrong.

Anton stored `HashSet<int[]>` pairs then replaced the row / zeroed the col. Passes. Cleaner: two `HashSet<Integer>`.

Test trap: `{1,0,3}/{4,5,6}/{0,8,9}` → col 1 dies, so the `5` becomes `0` (`[[0,0,0],[0,0,6],[0,0,0]]`).

**#290 deleted** (second Easy). Remaining hash easies: **#202**. **#76** still skipped.

#### Part 1 Design — Chapter 12 How to protect clients from servers — Circuit breaker

Talk. No Java. Still Part 1, **not** Part 3 (W5 Wed is circuit on this app).

**Closed:** calls Event. **Open:** Event failed enough; Booking **does not** call Event. **Half-open:** one trial later.

One dead **pod** ≠ open the whole Event circuit (LB skips that box). Open is **Event as a service** sick.

Anton: when open, the client must get a **proper error** — try again later. Not a ticket.

Trap: that error is **503**, not **201**. Not **409** (sold out). Not **429** (rate limit, Ch 11). Fallback that looks like booked is the lie.

#### Interview sentence

> Open circuit = fail fast, **503** try later. Never **201**. We did not take a seat.

### 60-sec (Part 1)

> #383: count magazine, spend on the note; extras allowed (`<=`, not `==`). #73: remember poisoned rows/cols, then wipe; in place ≠ O(1) extra. Ch 12: open = stop calling Event; user gets **503**, not a ticket.

**Weak:** open circuit → 201 / fake success. 503 mixed with 409 or 429.

**Calendar:** Part 1 **closed** (coding + Ch 12). Part 2 **closed** (see below). Part 3 **not yet.**

---

### Part 2 — Event pointer leftover + HTTP integration test

**Date:** 2026-09-10 (Thu, after Part 1)

#### After HTTP, Booking does not reload Event as inventory

Once service A has called service B over HTTP, A must not open B’s table to re-check B’s data. A only stores **B’s id** on its own row.

Here: `reserveSeats` already took seats. `findById` on Event would SELECT title/seats again — still one app. `getReferenceById(id)` is a stub with **only that id** so JPA can write `event_id`. Touch title/seats on that stub → Hibernate SELECTs anyway; then you paid for `findById` late. Two databases later: `Long eventId`, no `EventRepository` in Booking. Extra GET Event just to hang the ticket is a wasted hop — the Book URL already has the id.

The mix-up: stub Event = microservice split. It is still Event’s table. Also: stub is safe to **read** seats. It is not.

#### What an HTTP integration test proved vs a mock

A slice test that mocks the service can return **201** without the other service running. That proves the controller can write Created. It does not prove the hop.

Here: `BookingControllerTest` (`@WebMvcTest` + mock `BookingService`) = 201, Event never ran. `BookingEventCallIT` = real Tomcat, register/login/Book over HTTP, GET event, seats dropped. `WebClient` is a real client — it needs a **listening** port. MockMvc is not that. `${local.server.port}` is set **after** Tomcat starts; the `WebClient` bean is built **before**, so that placeholder dies. Test used a **known** port (8181) on both server and `event.service.base-url`. Boot 4: `@AutoConfigureTestRestTemplate` + `spring-boot-starter-restclient` (test).

The mix-up: controller 201 = Event subtracted seats. Also: “real repo” vs “real HTTP” — the IT’s proof is Event’s `FOR UPDATE` on **another** Tomcat thread.

### 60-sec (Part 2)

> After Event HTTP, Booking hangs `event_id` (`getReferenceById`), does not reload seats. Controller mock 201 ≠ hop. IT: listening port, client URL matches, seats drop.

**Weak:** timeout = Event wrote nothing (Wed). Stub Event = two services. Mock 201 = Event ran.

**Calendar:** Part 2 **closed**. Part 3 LSP + test-vs-mock **done** (same day).

---

### Part 3 — LSP + what the test proved

**Date:** 2026-09-10 (Thu, after Part 2)

#### LSP — stand-in must not look like a ticket

A substitute for Event (`EventClient.reserveSeats`) must keep the same promise: a **return** means seats were taken. Timeout / swallow → fake `EventResponseDto` → `book()` saves → phone **201** with no take. That is a broken stand-in. Catch must **throw** (`DownstreamServiceException` → 502). Covering the exception is a fake ticket, not 409.

W5 Wed does this again on circuit fallback. Do not treat Ch 12 as this slot.

The mix-up: returning a DTO on timeout “keeps going.”

#### What the test proved

Mock 201/409 = you scripted the service. No Event row. Handler only.

`ConcurrentBookingTest`: real H2, real `book()` Java. Default `@SpringBootTest` = **no Tomcat**. Does not prove `EventClient` HTTP.

`BookingEventCallIT`: Tomcat 8181 + client URL 8181 + POST Book. Proved the hop **in this process**. Still one H2 — `EventRepository` in Booking would be wrong on a second DB. Green IT ≠ three microservices.

The mix-up: autowired service = HTTP hop. `@WebMvcTest` uses the property port.

### 60-sec (Part 3)

> Timeout catch must throw or the phone gets a fake ticket. Mock 201 ≠ Event. IT = HTTP in this JVM, not two databases.

**Weak:** `@WebMvcTest` has a port. Stub `getAvailableSeats()` = null (it SELECTs). Concurrent test = EventClient HTTP.

**Calendar:** Day 4 **closed**. Day 5 Part 1 + Part 2 **done** (see below). Part 3 OCP leftover still open that day.

---

## Week 4 Day 5 — Happy Number + three-box HLD

**Date:** 2026-09-11 (Fri)

**Where each “part” is (read this first)**

| Name | What it is | Today |
|---|---|---|
| **Part 1 coding** | LeetCode in `leetcode-practice` | **#202** Happy Number. **#76** still skipped. **Done.** |
| **Part 1 Design (LC-SD)** | Course card talk | **Off** (Friday). |
| **Part 2** | Spring / HLD | **HLD of the three boxes.** No new code. **Done.** |
| **Part 3** | OOP + this-app design | **OCP leftover** (new Event error → mapping in the client, not a giant `if` in `book()`). **Done.** |

Sat/Sun **off**. After Part 3 → Week 5.

---

### Part 1 — coding LC — no LC-SD

#### LC 202 Happy Number (Easy)

Repeat: replace n with the sum of the squares of its digits. Happy if you hit **1**. Unhappy if you loop.

**HashSet:** put seen n. Next already in the set → cycle → false. Hit 1 → true.

Floyd (slow/fast on the same function) also works. Not a sliding window. Not sort.

**#76** still skipped.

### 60-sec (Part 1)

> #202: sum of digit squares. Seen set catches the cycle. 1 = happy. Friday = no Chapter talk.

---

### Part 2 — HLD of the three boxes

**Goal:** One board. Phone → Booking → Event / Auth. Headers, last seat, hang, connection vs lock. No new code. Gateway / retry / circuit stay Week 5.

#### Box = service

A box is a **running service** (own process, own HTTP door, own data). Not a Spring `@Service` class. `BookingServiceImpl` / `EventClient` live **inside** Booking.

| Box | Owns |
|---|---|
| **Auth** | Who you are (users, JWT, roles) |
| **Event** | Leftover seats (the concert row) |
| **Booking** | The ticket |

Phone talks only to Booking on Book.

#### Flow

Phone → **Booking**. JWT filter runs **inside Booking** (before the controller). That is not the Auth box.

Then `book()` hops HTTP → **Auth** (`/api/users/me`), then HTTP → **Event** (take seats), then save the ticket.

The hop is a **new** request. Copy `Authorization` or Auth returns **401**. Copy `X-Correlation-Id` or Event mints a second id — Book still works, logs do not stitch. **No throw.**

JWT = who. Correlation id = log sticker (keep if sent, else mint). Click id = which tap (**not in the app**).

#### Phone statuses

Event 409 / 404 / 5xx map in `EventClient` to our exceptions. Phone sees **409 / 404 / 502**, not Java type names.

#### Hang

Timeout is **no HTTP status**. `WebClientRequestException` → 502. Event may already have subtracted, still be in `FOR UPDATE`, or never started. **We don’t know.** Timeout ≠ “Event wrote nothing.” Timeout ≠ 409.

Today a second tap is a **new Book**. Same correlation id (if the phone resends it) does not make it one tap. Click id would return the **same ticket** (201), not “already booked.” “Already booked” is a different rule (one ticket per user/event).

#### Last seat

Wait is on **Event’s** row (`findByIdForUpdate`). One **201**, one **409**. Both can already be inside Booking and already on HTTP. Second does not wait in Booking before the call. Booking `.block()`s until Event answers.

#### `@Transactional` vs lock vs HTTP

`@Transactional` is **not** a lock. It wraps Booking SQL: commit all or roll back all. While it is open, Booking **holds a DB connection** from its pool — even during `.block()` with no Booking SQL.

Event’s `FOR UPDATE` is Event’s database, Event’s pool, a **row lock**. Two boxes, two DBs: Booking’s rollback does not undo Event’s seat write.

Here: `book()` opens the transaction, then waits on Auth + Event HTTP, then `save`. Ten Books can occupy ten pool slots on HTTP. Do not keep that connection open across the hop.

The mix-up: JWT filter = Auth box. `EventClient` = a box. Hang = Event wrote nothing. Same correlation UUID = same Book. `@Transactional` = the Event row lock.

### 60-sec (HLD)

> Phone → Booking only. Filter is local JWT; Auth box is a later HTTP hop (copy Bearer). Seats live on Event. 409 sold out, 404 missing, hang/5xx → **502**. Timeout: we don’t know if Event wrote. Wait on Event’s row, not in Booking before HTTP. `@Transactional` holds a connection, not a lock — don’t hold it across HTTP.

**Weak:** Auth filter = Auth service. Timeout = seats unchanged. Correlation id = click id. `@Transactional` = `FOR UPDATE`. Second tap with same sticker = one ticket.

**Calendar:** Part 2 **closed**. Part 3 **done** (see below).

---

### Part 3 — OCP leftover

**Date:** 2026-09-11 (Fri, after Part 2)

Open to **add**. Closed to **rewrite** working ticket logic.

**General:** a new Event HTTP error is an extension of the **mapping**, not of `book()`.
**Here:** 403 (may not Book this show) → new `if` in `EventClient`’s `catch (WebClientResponseException)`. Throw a domain exception. If the type is new, one handler so the phone gets **403**. `book()` stays “reserve, save.”
**Trap:** `if (403)` inside `book()` — then `book()` speaks HTTP. Also: map 403 on the shared `WebClient` bean (Auth’s 401 shares that bucket).

Do **not** code 403 today. Interview words only.

### 60-sec (Part 3)

> OCP: new Event status → extend `EventClient` catch + handler if needed. Do not grow `if`s in `book()`.

**Weak:** 403 in `book()`. Shared WebClient filter for all statuses.

**Calendar:** Day 5 **closed**. Sat/Sun **off**. **Next weekday:** Week 5 Day 1 — LC first, then gateway routes + first service behind it. OOP: equals/hashCode + Collections (longer).
