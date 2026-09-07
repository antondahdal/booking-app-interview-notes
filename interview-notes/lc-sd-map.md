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
| **W4 Tue** | Ch 8 — Timeout / retry / idempotency | Event HTTP is slow; retry only with a click id | Retry `book()` blindly |
| **W4 Wed** | Ch 3 + 5 — Sync vs queue | Do not hold a DB door across a slow hop | Email inside the lock |
| **W4 Thu** | Ch 12 — Circuit breaker | Open circuit ≠ **201 ticket** | Fallback looks like booked |
| **W4 Fri** | — | — | — |
| **W5 Mon** | Ch 4 — Cache (aside, TTL) | Speed browse; book is still truth | Cache “1 seat left” as gospel. **Skip** if Part 3 is already cache |
| **W5 Tue** | Ch 13 — Auth practical | Login, token, who may call | Full OAuth vendor sketch |
| **W5 Wed** | Ch 10 — Hot key / partition lite | One event is hot, others are not | Redesign Kafka |
| **W5 Thu** | Ch 1 — Requirements on **this app** | Functional vs 10× / latency first | Jump to boxes |
| **W5 Fri** | — | — | — |
| **W6 Mon** | Ch 5 — Queue + at-least-once | Outbox after commit | Exactly-once mail |
| **W6 Tue** | Ch 8 recap if weak, else Ch 7 service-to-service | Booking → Event | 2PC |
| **W6 Wed** | Ch 9 — CDN / edge **skip** unless they ask; use batching/timeout instead | — | YouTube |
| **W6 Thu** | Weak recap (rate limit **or** retry) | — | A new Hard product |
| **W6 Fri** | — | — | — |
| **W7 Mon** | Ch 10 — Consistent hashing lite | Add a cache node, few keys move | URL shortener (that is **Fri Part 3**) |
| **W7 Tue** | Ch 11 recap or leaderboard skip | — | — |
| **W7 Wed** | **Skip** if Part 3 is cache | — | Same prompt twice |
| **W7 Thu** | Weak recap | — | Full bit.ly |
| **W7 Fri** | — | URL shortener is **Part 3** (course practical #1) | Do not run it here |
| **W8 Mon–Thu** | Recap whichever course piece was weak | — | Fraud pipeline, Uber, YouTube |
| **W8 Fri** | — | Mock HLD is **Part 3** | — |

## Done

| Slot | Course piece |
|---|---|
| **W4 Mon** | Ch 11 — Rate limiting (explained in notes; not a drill) |
