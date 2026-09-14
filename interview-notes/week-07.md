# Interview notes — Week 7

Polish + performance: security, N+1, indexes, README.

**Packed:** **two** Part 1 topics each Mon–Thu. Friday = one HLD. Full calendar: [part1-map.md](part1-map.md).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | N+1 fix | Index on hot FK |
| Tue | Security pass (secrets/CORS) | README |
| Wed | One perf check **(+ Spring Cache lab if not done)** | Leftover polish |
| Thu | Remaining polish | Test |
| Fri | HLD/recap | — |

**Part 3:** continue LLD class-design (started Week 6). **W7 Wed longer:** cache talk on this app ([oop-design-map.md](oop-design-map.md)) **and** the Spring `@Cacheable` lab below.

**Not started yet.** Say `Start Week 7 Day 1` when you get here.

---

## Spring Cache lab (W7 Wed)

Same idea as Week 5 Chapter 4 talk, with Spring names. **Not** a Week 5 lab.

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

**Lab status:** notes only. **Not coded.** Do this on **W7 Wed**, not in Week 5.
