# Interview notes — Week 4

Microservices split: 3 services, WebClient, correlation IDs.

**Packed:** **two** Spring topics each Mon–Thu. Friday = one HLD. Full calendar: [part2-map.md](part2-map.md).

**Part 1:** two coding LCs, then **~15 min talk** from [System Design for Interviews and Beyond](https://leetcode.com/explore/interview/card/system-design-for-interviews-and-beyond) Mon–Thu ([lc-sd-map.md](lc-sd-map.md)). Friday = coding LCs only. Always name **Chapter N + topic**. From Tue: explain a bit, then **question** (Anton talks first).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Booking calls Event over HTTP | `WebClient` bean | **Pulled to W3 Thu** |
| Tue | Third service (auth/users) | Correlation-id header | **Next (Spring)** |
| Wed | Downstream 4xx/5xx mapping | Client timeout |
| Thu | Remaining split glue | One integration test for the call |
| Fri | HLD of the three boxes | — |

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

**Calendar:** Part 1 closed. **Next:** Spring in booking-app — auth/users + correlation-id. Then Part 3: seats in Event over HTTP / slow hop. Do not re-ask Strategy.
