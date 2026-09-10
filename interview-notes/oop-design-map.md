# OOP + Design map (mid-level Java)

**Pick today’s slot here. Do not invent. Do not repeat Done.**

| Map | What it is |
|---|---|
| **This file** | How Part 3 runs + **calendar** (which day → which OOP + which design prompt) |
| [oop tables below](#oop--done-do-not-repeat) | OOP done vs still need |
| **[design-map.md](design-map.md)** | Design done vs still need **for the role** (five families, traps, skip list) |

**Role:** mid-level Java / Spring. Interviews ask **OOP every weekday** and **design** (API + this app + Friday board). **LLD** starts **Week 6**. Not before.

**How a weekday Part 3 runs**

**Opener (every Part, and when switching OOP → Design):** Cover today (bullets) + **one sentence** why a mid-level Java role gets this. Then the question.

| Piece | Time | What |
|---|---|---|
| OOP | ~10 min, **some days ~25** (Java core) | **New idea(s)** from the remaining OOP list. |
| Design | ~45 min, **some days ~70** | **Prompt(s)** from that day’s slot. Full list: [design-map.md](design-map.md). |
| Friday | 60–75 min, **W7 Fri longer** | HLD (+ 2 min OOP). W7 Fri = extra classic HLD. |
| Sat/Sun | — | **Off.** |

Days marked **longer** in the calendar: do both items. Anton asked to be safe, not to skip the extra.

**Weeks 1–5:** no parking-lot class sketches. **Weeks 6–8:** add LLD (see bottom).

---

## OOP — done (do not repeat)

| Topic | When | One line | Trap |
|---|---|---|---|
| Layers (one job each) | W1 D1 | Controller HTTP, service rules, repo DB, model row | Empty packages ≠ knowing the jobs |
| Constructor DI | W1 D1–D2, W3 D3 Part 1 | Production beans: constructor. Test class: field `@Autowired` is OK | `@Autowired` vs constructor as opposites |
| DTO vs entity (seed) | W1 D2 | JSON is not the table row | Expose `Event` and lazy/version leak |
| Composition (has-a) | W1 D3, W2 D2 | Service *has* repos. Event *has* Venue. Filter chain *has* filters | `extends` for reuse |
| ISP + DIP | W1 D4–D5 | Smallest type; controller → service **interface** | Repo in the controller |
| Interface vs abstract class | W2 D1 | Interface = contract / mock. Abstract = shared fields + code | Mixing the names |
| Singleton service | W2 D1 | One bean, many requests. No “current user” field | User on the service mixes threads |
| Chain of responsibility | W2 D2 | Each filter one job, then pass on | JWT + URL rules in the controller |
| SRP (who vs may vs book) | W2 D3–D4, W3 D1 Q1 | Filter = who. Matcher = may this URL. BookingService = ticket | “It’s on Event so EventService.book()” |
| Stamp ownership | W3 D2 Q1 | `@Version` on the inventory object, not Booking, not GET JSON | “JPA mapping” as the OOP answer |
| Encapsulation | W3 D3 | PATCH body = title. Client must not set seats / `version` | Calling this SRP |
| Java threads | W3 D3 | Latch = start. `join` = caller waits. Not `synchronized` on `book()` (two JVMs) | `join` = t1 waits for t2 |
| Strategy (job picks wait vs stamp) | W3 Thu, W3 Fri HLD | Book waits. Title PATCH stamps. Pool pain ≠ switch Book to stamp | Re-ask “which method / the job” |
| Adapter | W4 Day 3 leftover (Tue slot) | `EventClient` **uses** `WebClient`. `book()` says reserve seats, not `post`/`.block()` | HTTP in `book()` = two services merged |
| LSP (timeout ≠ ticket) | W4 Day 4 | Stand-in must not return success if Event did not take a seat. Catch must throw. | Fake DTO → **201** |

**SOLID so far:** S, I, D, L (timeout ≠ 201). Encapsulation. **Not yet:** O (Wed leftover).

---

## OOP — still need (this role)

Must-have. Includes **Java core** (collections, threads) — mid interviews ask these even when the round is “OOP.”

| # | Topic | Why they ask | Slot |
|---|---|---|---|
| 5 | **OCP** | New downstream error → new mapping, not a giant `if` in `book()`. | W4 Wed |
| 6 | **LSP** | Timeout / fallback must not look like **201 booked**. | W4 Thu **done** + W5 Wed (circuit) |
| 7 | **equals / hashCode + Collections** | Entity id after persist. `HashMap` uses both. `ArrayList` vs `LinkedList` (random access vs middle insert). | **W5 Mon (longer OOP)** |
| 8 | **Immutability + `Optional`** | Request DTO / `String` don’t mutate. `Event` is mutable (the row). `Optional` = empty box, not empty entity. | W5 Tue |
| 9 | **Observer / events** | `book()` commits, then outbox — not email inside the lock. | W6 Mon |

**Nice if leftover:** records vs class, Factory as `@Bean`, Facade = Gateway, checked vs unchecked (domain = unchecked).

**Skip:** Visitor, Prototype, Flyweight, Mediator, square-rectangle, JVM GC tuning.

---

## Design — calendar only (full skill list → [design-map.md](design-map.md))

Do not repeat prompts in **design-map.md → Done**. The five families are: **who**, **truth**, **status**, **10×**, **slow hop**.

### Rest of Week 3 (monolith + two tools)

| Day | Design prompt (~45 min) | OOP that day |
|---|---|---|
| **W3 D3** | **Two tools, one row** + **PUT vs PATCH** + **isolation** + first matcher (403 ≠ 409). | Encapsulation **+ Java threads** — **done** |
| **W3 Fri** | **Long HLD.** Last seat at 1 / 10 / 100. Wait vs stamp. Pool. Title clash. Boxes. **Done.** | Strategy already on the board — **do not re-ask** |

### Week 4 (three processes)

| Day | Design prompt | OOP |
|---|---|---|
| Mon | Seats live in **Event service**. Booking calls HTTP. Who is source of truth? What if Event is slow? | Strategy **done** W3 Thu/Fri — skip; do not re-ask |
| Tue | Correlation id: what you log, what you return, why the user never sends it | Adapter |
| Wed | Downstream 404 vs 409 vs 503 vs timeout — what Booking returns. Retry or not. | OCP |
| Thu | What an HTTP integration test proved vs a mock | LSP (client contract) | **Done W4 Day 4** |
| Fri | **HLD** of the three boxes | — |

### Week 5 (gateway + resilience)

| Day | Design prompt | OOP |
|---|---|---|
| Mon | JWT at gateway vs again in the service | equals/hashCode **+ Collections (longer OOP)** |
| Tue | Which calls may **retry** (GET vs book). Need a click id to retry book. | Immutability + `Optional` |
| Wed | Circuit open: fallback status. Must not look like a successful ticket | LSP |
| Thu | Live vs ready. Kill a pod that still has `FOR UPDATE` | ISP (health ≠ book) |
| Fri | **HLD** traffic through the gateway | — |

### Weeks 6–8

| When | Design | OOP / LLD |
|---|---|---|
| W6 Mon–Thu | Outbox, at-least-once mail, metrics on `book()` | Observer + **LLD** (see bank) |
| W6 Fri | HLD async | — |
| W7 Mon–Tue | Index, N+1, secrets — as **bottleneck** talk | More LLD |
| **W7 Wed (longer)** | **Cache** (aside, TTL, don’t cache “1 seat left” as truth) | LLD |
| **W7 Fri (longer)** | **Classic HLD:** URL shortener. **429** rate limit ≠ 409 sold out. | — |
| W8 Fri | Long mock: **HLD + one LLD** | — |

---

## LLD bank (Week 6+, not this booking app)

Whiteboard only. No Java files. Rotate. Do not repeat the same product.

Name the product → actors → 4–8 classes → fields + 2–4 methods each → say has-a vs is-a once.

**Bank:** parking lot, library, hotel rooms, food-delivery order, split-bill, URL shortener (classes, not AWS), chat (User / Message / Room), notification outbox.

**Critique:** missing entity, god-class, wrong is-a.

---

## Next session — Week 4 Friday

Part 1 / 2 / 3 Day 4 **done**. Do **not** rerun LSP, IT vs mock, Adapter, correlation id, Ch 12 as Part 3. Do **not** rerun queue/email. Do not re-ask Strategy.

**Fri:** LC first (coding only — no LC-SD). Then **HLD of the three boxes** (phone → Booking → Event / Auth). Small OOP: **OCP** leftover (new Event error → mapping in the client, not a giant `if` in `book()`). Weak: WebMvc has no port. Stub seats = SELECT. Concurrent ≠ HTTP hop. Timeout ≠ Event wrote nothing.

Classic product HLD lite lives in **Part 1** ([lc-sd-map.md](lc-sd-map.md)). Part 3 stays this app until W6 LLD. URL shortener stays **W7 Fri Part 3**.
