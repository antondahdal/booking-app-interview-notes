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

**Not started yet.** Say `Start Week 7 Day 1` when you get here.

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

