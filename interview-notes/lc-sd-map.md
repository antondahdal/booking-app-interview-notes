# Part 1 — LC System Design (from Week 4)

**Anton 2026-09-07:** this slot is **talk** from LeetCode **[System Design for Interviews and Beyond](https://leetcode.com/explore/interview/card/system-design-for-interviews-and-beyond)**. Not Design-tag coding (#146 LRU). Not Unique IDs unless that course names it.

**This is still Part 1 (LC).** It does **not** replace Spring or Part 3.

Course chapters used as the bank (same names as the card / published outline):

1. How to define system requirements  
2. How infrastructure shapes system qualities — **skip** for this role (2 min if they ask)  
3. Foundations of reliable, scalable, and fast communication  
4. How caching improves performance  
5. Why queues matter in distributed systems  
6. Data store internals — **skip** (RocksDB / LSM)  
7. How to build efficient communication between components  
8. How to deliver data reliably  
9. How to deliver data quickly  
10. How to deliver data at scale  
11. How to protect servers from clients  
12. How to protect clients from servers  
13. Practical exercises: URL Shortener · Fraud Detection · Auth — **Fraud skip**; URL shortener = **W7 Fri Part 3**

## Cadence

| When | What |
|---|---|
| **Mon–Thu** | Two coding LCs (Weeks 6–8: three), **then ~15 min** talk. No Java. |
| **Friday** | Coding LCs only. Long HLD stays Spring / Part 3. |
| **Sat/Sun** | **Off.** |

## How to run (~15 min)

**Anton 2026-09-07:** do **not** lecture the whole board.

1. **Cover today:** **Chapter N** + **topic** (same title as the card).
2. **Why for your role:** one sentence.
3. **Explain the topic a bit** (what it is, what problem it solves). A few sentences, not a course dump.
4. **Then the interview question** — he talks first (product, actors, 2–3 calls, boxes, one 10×, one status if it matters).
5. After he answers: trap + one interview sentence. Stop.

Do not invent Kafka. Do not implement. Do not fill in actors/boxes before he speaks.

Skip if that product is **already** that day’s Part 3 or Friday HLD.

## Bank (this role). Do not invent extras.

| Slot | Course piece | Why they ask (this role) | Do not |
|---|---|---|---|
| **W4 Mon** | Ch 11 — Rate limiting | 429 = too many calls | Call this sold-out **409** | **Done** |
| **W4 Tue** | Ch 8 — Timeout / retry / idempotency | Event HTTP is slow; retry only with a click id | Retry `book()` blindly | **Done 2026-09-08** |
| **W4 Wed** | Ch 3 + 5 — Sync vs queue | Do not hold a DB door across a slow hop | Email inside the lock | **Done 2026-09-09** |
| **W4 Thu** | Ch 12 — Circuit breaker | Open circuit ≠ **201 ticket** | Fallback looks like booked | **Done 2026-09-10** |
| **W4 Fri** | — | — | — |
| **W5 Mon** | Ch 4 — Cache (aside, TTL) | Speed browse; book is still truth | Cache “1 seat left” as gospel. **Skip** if Part 3 is already cache | **Done 2026-09-14** |
| **W5 Tue** | Ch 13 — Auth practical. **Plus** cache on this app (extra, not a new chapter) | Login, token, who may call. Then cache until key / 10× / Book skips it | Full OAuth vendor sketch. `@Cacheable` code (W7 Wed) | **Auth skip** (Mon Part 3). Cache extra **Done 2026-09-15** |
| **W5 Wed** | Ch 10 — Hot key / partition lite | One event is hot, others are not | Redesign Kafka | **Done 2026-09-16** |
| **W5 Thu** | Ch 1 — Requirements on **this app**. **Scrapped.** Replaced with **Ch 7** service-to-service | Booking waits on Event HTTP | 201 before Event answers | **Done 2026-09-17** |
| **W5 Fri** | — | — | — | **Off** (coding only) |
| **W6 Mon** | Ch 5 — Queue + at-least-once | Outbox after commit | Exactly-once mail | **Done 2026-09-21** |
| **W6 Tue** | Ch 8 recap if weak, else Ch 7 service-to-service | Booking → Event | 2PC | **Done 2026-09-22** (Ch 8 recap) |
| **W6 Wed** | Ch 9 — CDN / edge **skip** unless they ask; use batching/timeout instead | — | YouTube | **Done 2026-09-23** (batching/timeout; CDN skip) |
| **W6 Thu** | Weak recap (rate limit **or** retry) | — | A new Hard product | **Done 2026-09-24** (Ch 11 rate limit drill; retry already Tue) |
| **W6 Fri** | — | — | — |
| **W7 Mon** | Ch 10 — Consistent hashing lite | Add a cache node, few keys move | URL shortener (that is **Fri Part 3**) |
| **W7 Tue** | Ch 11 recap or leaderboard skip | — | — |
| **W7 Wed** | **Skip** if Part 3 is cache | — | Same prompt twice |
| **W7 Thu** | Weak recap | — | Full bit.ly |
| **W7 Fri** | — | URL shortener is **Part 3** (course practical #1) | Do not run it here |
| **W8 Mon–Thu** | Recap whichever course piece was weak | — | Fraud pipeline, Uber, YouTube |
| **W8 Fri** | — | Mock HLD is **Part 3** | — |
| **W9 Mon–Thu** | **No LC-SD.** Slot = two Spring Boot interview questions; Fri rapid-fire ([week-09.md](week-09.md)) | Anton 2026-09-25 | Run an LC-SD chapter here |

## Done

| Slot | Course piece |
|---|---|
| **W4 Mon** | Ch 11 — Rate limiting (explained in notes; not a drill) |
| **W4 Tue** | Ch 8 — Timeout / retry / idempotency |
| **W4 Wed** | Ch 3 + 5 — Sync vs queue (commit, then enqueue, then 201; ack ≠ Gmail) |
| **W4 Thu** | Ch 12 — Circuit breaker (open = 503 try later, never 201) |
| **W5 Mon** | Ch 4 — Cache / TTL (browse GET; Book is Event’s row; never “1 left” → 201) |
| **W5 Tue** | Ch 13 Auth skip (Mon Part 3). Cache extra (key / 10× browse / Book skips cache) |
| **W5 Wed** | Ch 10 — Hot key / partition lite (event 7; GET can cache; Book still one row; 409 sold out ≠ 502 lock wait) |
| **W5 Thu** | Ch 1 scrapped. **Ch 7** service-to-service (Booking HTTP wait; 201 only after Event answers) |
| **W5 Fri** | — (coding only) |
| **W6 Mon** | Ch 5 — Queue + at-least-once (outbox with the book; worker may run the same row twice) |
| **W6 Tue** | Ch 8 — Timeout / retry / click id recap |
| **W6 Wed** | Ch 9 — batching / timeout (CDN skip; do not merge separate Books) |
| **W6 Thu** | Ch 11 — rate limit drill (per user per window, before `book()`, shared counter; 429 not 409; not per commit) |
