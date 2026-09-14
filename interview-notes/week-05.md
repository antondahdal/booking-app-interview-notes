# Interview notes — Week 5

Gateway + resilience: Gateway, Resilience4j, Docker Compose. **Spring Cache** is interview + code shape here (lab on a thin day or W7 Wed). Git: [git.md](git.md).

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
| **Part 2** | Spring | Gateway routes + first service behind it. **Not this chat.** |
| **Part 3** | OOP + this-app design | equals / hashCode + Collections (longer). **Not this chat.** |

Do **not** call Chapter 4 “Part 3.” W7 Wed is cache on this app as a **design** prompt. Spring `@Cacheable` lab = thin day or W7 Wed perf (below).

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

**Calendar:** Part 1 **closed**. Part 2 Gateway **open**. Part 3 equals/hashCode **open**.

---

## Spring Cache (this app) — was missing from Part 2; add here

**Not Monday’s Gateway lab.** Know it for interviews. Code on a **thin day** or **W7 Wed** (perf check). Same idea as Ch 4, with Spring names.

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

**Lab status:** notes only. **Not coded.**
