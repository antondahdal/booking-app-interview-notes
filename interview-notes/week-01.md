# Interview notes — Week 1

Monolith bootstrap: REST, JPA, pagination, tests.

Each day’s **Quick recall** is **general** (any Spring app) → **here** → **trap**. You can read a day without the chat. “What I built” under that is the story of that session.

---

## Week 1 Day 1 — Bootstrap + entities + layered architecture

**Date:** 2026-08-17  
**Goal:** Spring Boot app with packages and JPA entities (`Venue`, `Event`).

### Quick recall

**Layers**  
- **General:** Split HTTP, rules, and DB. Controller = URL/status/JSON. Service = rules + throws. Repository = load/save. Model = table row. Interviewer wants one job each.  
- **Here:** `VenueController` / `EventController`; `*Service`; `*Repository`; `Venue` / `Event`.  
- **Trap:** Empty packages ≠ knowing the jobs.

**DI**  
- **General:** You do not `new` collaborators. Spring creates **beans** and passes them in. Prefer **constructor** (`final`, `new Foo(fake)` in tests, missing bean fails at startup). Field `@Autowired` hides deps.  
- **Here:** Controllers/services take deps in the constructor.  
- **Trap:** DI is “someone hands you what you need,” not “I used annotations.”

**`@SpringBootApplication`**  
- **General:** `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Scan this package → beans → auto-config (JPA/web if starters present) → start Tomcat.  
- **Here:** On `EventBookingPlatformApplication`.  
- **Trap:** Not “magic HTTP.” Scan + auto-config + server.

**Profiles**  
- **General:** Same Java, different config. `spring.profiles.active=dev` loads `application-dev.properties`. Prod = another profile, not rewrite Java.  
- **Here:** `dev` → H2.  
- **Trap:** Config switch, not “H2 class vs Postgres class.”

**`@Entity` / `@Id` / `@GeneratedValue(IDENTITY)`**  
- **General:** `@Entity` = table. `@Id` = PK. `IDENTITY` = **DB** auto-increments on INSERT; Java does not invent the id. After `save`, Hibernate fills `getId()`. Client does not send id on create.  
- **Here:** Venue, Event (later User, Booking).  
- **Trap:** “No duplicates” is unique constraints. IDENTITY = who **assigns** the number.

### What I built

- Layered packages: `controller`, `service`, `repository`, `model`, `dto`, `exception`
- Entities: `Venue` (name, address, capacity), `Event` (title, dateTime, seats, `@ManyToOne` Venue)
- H2 `dev` profile + console

### Under the hood (cheat)

**Dependency injection**  
You do not `new EventService()`. Spring scans `@Service` / `@Repository` / `@RestController`, builds objects (**beans**), and wires constructor args. That is DI: the framework provides dependencies.

**`@SpringBootApplication`**  
Shortcut for: `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. On startup: scan your package → create beans → auto-config DataSource/JPA/web if starters are on the classpath → start Tomcat.

**Why IDENTITY for PKs**  
`IDENTITY` = database generates the id (auto-increment). After `save`, Hibernate fills `entity.getId()`. You never send `id` on create from the client.

**Profiles vs hardcoding H2**  
`application.properties` can set `spring.profiles.active=dev`. `application-dev.properties` has H2 URL. Same code, different config. Prod later swaps profile → PostgreSQL without rewriting Java.

### Interview answers (model)

1. **DI?** Spring creates and injects beans (prefer constructor injection).
2. **Four layers?** Controller HTTP · Service logic · Repository DB · Model entity.
3. **`@GeneratedValue(IDENTITY)`?** DB auto-increments PK on insert.
4. **Profiles?** `dev` → H2; prod → real DB. Config switch, not hardcoded one DB.
5. **`@SpringBootApplication`?** Scan + auto-config + start server.

### Weak spots (Day 1 — review often)

- DI: Spring creates beans, injects via constructor  
- Layers: empty packages ≠ understanding — say one job each  
- Profiles: `dev` profile → H2  

### 60-sec story

> "Layered Spring Boot monolith. Venues host events; events track seats. Lazy `@ManyToOne` Event→Venue. Packages mirror how I'd split services later."

---

## Week 1 Day 2 — REST + DTOs + validation

**Date:** 2026-08-18  
**Goal:** Expose venues over HTTP without returning JPA entities.

### Quick recall

**Why DTOs**  
- **General:** JSON contract ≠ table shape. Returning an entity leaks fields, breaks when you add columns, and Jackson walking **lazy** associations causes extra SQL or `LazyInitializationException`.  
- **Here:** `VenueCreateRequestDto` / `VenueResponseDto` (same later for Event, User, Booking).  
- **Trap:** “DTO is for performance” is weak. Reasons: **contract, safety, lazy**.

**Create vs response**  
- **General:** Request has **no** generated id (DB assigns it). Response **has** id after `save`.  
- **Here:** Create venue JSON has no id; 201 body includes id.  
- **Trap:** Id appears **after** save, not before.

**`@RestController` vs `@Controller`**  
- **General:** `@Controller` often returns a **view name** (HTML). `@RestController` = `@Controller` + `@ResponseBody` → return value **is** the HTTP body (JSON).  
- **Here:** All API controllers are `@RestController`.

**`@Valid` + `@RequestBody`**  
- **General:** `@RequestBody` = JSON → Java. `@Valid` on that argument runs Bean Validation **before** the method. Fail → **400**, method never runs. `@NotBlank` = not null/empty/spaces. `@Email` without `@NotBlank` can pass `""`. `@PathVariable` = `{id}` in the URL. `@RequestParam` = `?page=0`.  
- **Here:** `@Valid` on create (later register/book).  
- **Trap:** Validation is not inside the method.

**HTTP create/read/update/delete**  
- **General:** Create **201** (new resource). Read **200**. Update **200/204**. Delete **204**. **400** = bad input. **404** = missing id. **409** = valid request, **state** says no. **500** = unhandled bug.  
- **Here:** POST venue 201, GET 200.  
- **Trap:** 409 is not “duplicate email only” (Week 2).

**Constructor injection**  
- **General:** Constructor + `final` = required deps obvious; tests `new Controller(fakeService)`. Field `@Autowired` needs Spring. One constructor → `@Autowired` optional.  
- **Here:** `VenueController(VenueService)`.

### What I built

- `VenueCreateRequestDto` (`@NotBlank`, `@Min`) — inbound, **no id**
- `VenueResponseDto` — outbound, **with id**
- `VenueRepository`, `VenueService`, `VenueController` (`POST` / `GET` list / `GET` by id)
- `@Valid` on create body

### Under the hood (cheat)

**Request flow**  
JSON → Jackson fills DTO (needs no-arg ctor + setters/getters or Lombok) → `@Valid` runs → controller → service maps DTO→entity → `save` → DB generates id → map entity→response DTO → Jackson → JSON.

**Why not return `Venue` entity**  
1. Client could send/`see` fields you never meant (security + API stability).  
2. Entity may gain audit/`version` later — API would change by accident.  
3. Relationships + LAZY: Jackson calls getters → extra SQL or `LazyInitializationException`.

**`@Controller` vs `@RestController`**  
`@Controller` often returns a **view name** (Thymeleaf). `@RestController` = write the return value as the **HTTP body** (JSON).

**Constructor vs field injection**  
Constructor + `final`: required deps are obvious; tests can `new Controller(fakeService)`. Field `@Autowired`: hidden, harder to test, mutable.

### Interview answers (model)

1. **DTOs?** Decouple API from JPA; control JSON; avoid lazy serialization issues.
2. **`@RestController`?** Controller + ResponseBody (JSON).
3. **`@Valid`?** Triggers Bean Validation on the argument before the method runs.
4. **HTTP verbs/status?** POST 201, GET 200, PUT 200/204, DELETE 204.
5. **Constructor injection?** Prefer — immutable, explicit, testable.

### Check traps you hit

- Create request has **no** id; response **has** id (id appears **after** `save`, not before).
- Mapping: `@GetMapping("/{id}")` is relative to class `@RequestMapping("/api/venues")`.

### 60-sec story

> "Venue REST: request DTO for create (validated), response DTO with generated id. Never expose the JPA entity. Thin controller, mapping in the service."

---

## Week 1 Day 3 — JPA / Event CRUD / lazy + N+1

**Date:** 2026-08-19  
**Goal:** Persist Events linked to Venue; see lazy loading and fix N+1 for list.

### Quick recall

**Lazy vs eager**  
- **General:** `@ManyToOne(fetch = LAZY)` = load the parent only when you touch it (proxy until then). **Eager** = always load with the child (heavy). Lazy is not “never load.” After the session ends, touching the proxy can explode (`LazyInitializationException`) or Open-Session-In-View silently runs extra SQL.  
- **Here:** `Event.venue` is lazy. `findById(Event)` does not load venue until you need name/address.

**N+1**  
- **General:** 1 query for the list + **1 query per row** when you touch the association (Jackson `getVenue().getName()`). Fix: `@EntityGraph` or `JOIN FETCH`. With **paging**, JOIN FETCH + `LIMIT` can mean “20 joined rows,” not “20 events” — use `@EntityGraph`.  
- **Here:** Event list must fetch venue with the query (Day 4: `@EntityGraph` + `Pageable`).

**`Optional`**  
- **General:** `findById` might find nothing. Empty **box**, not an empty entity. `orElseThrow` → not-found exception. `null` from repo → NPE → looks like 500.  
- **Here:** Missing venue on create-event → `ResourceNotFoundException` (Day 4 → 404).

**`JpaRepository` vs `CrudRepository`**  
- **General:** Crud = save/find/delete. Jpa = that **plus** paging/sorting (`Pageable`, `Page`).  
- **Here:** Event list needs `Page`.

**`ddl-auto=update`**  
- **General:** Hibernate patches schema to match entities. Fine for local H2. **Not** prod — use Flyway/Liquibase.

### What I built

- `EventRepository`, Event DTOs, `EventService`, `EventController`
- Create: load Venue by `venueId` → `orElseThrow` → `setVenue` → save Event (`availableSeats = totalSeats`)
- List: avoid N+1 (e.g. `findAllWithVenue` / JOIN FETCH) — Day 4 switched list to `@EntityGraph` + `Pageable`

### Under the hood (cheat)

**Lazy story (plain English)**  
`findById(Event)` runs `SELECT` on `events` only. `venue` field is a stand-in (proxy): “FK is 5; if you ask for name, I’ll query then.” That query only works while the Hibernate **session** (DB conversation) is open. After the service/transaction ends, Jackson calling `getVenue().getName()` can explode → `LazyInitializationException`. Or with open-in-view, it “works” but silently runs extra SQL (N+1).

**Why create takes `venueId` (Long), not nested Venue JSON**  
Client **points at** an existing venue. Service loads managed `Venue` and sets the FK. Nested Venue would let clients invent junk venues or confuse “create venue” with “create event.”

**Repository returns entity, not DTO**  
Repo = persistence. Service maps entity ↔ DTO. Controller never sees `Venue` entity if you keep layers clean.

**`Optional` wording**  
Not “empty Event.” Empty **Optional** = no row. `orElseThrow` → your `ResourceNotFoundException` (Day 4 maps to 404). If repo returned `null`, easy NPE → looks like 500 bug.

**Composition in code**  
`EventService` **has** `EventRepository` + `VenueRepository` (not extends). Event **has** Venue (`@ManyToOne`). Inheritance = “is a” (e.g. exception extends RuntimeException).

### Interview answers (model)

1. **Lazy vs eager (Event→Venue)?** Lazy = load venue when needed; eager = always join/load. Default association often LAZY to avoid heavy graphs.
2. **N+1?** One query for parents + one per child association access.
3. **`Optional`?** Makes absence explicit; avoid null NPEs; `orElseThrow` for 404 path.
4. **`JpaRepository` vs `CrudRepository`?** Jpa = CRUD + paging/sorting (+ JPA extras).
5. **`ddl-auto=update`?** Auto schema change; fine for local H2; prod uses migrations.

### Part 2 — LC 217 Contains Duplicate

- **Pattern:** HashSet, one pass, early return  
- **Complexity:** O(n) time, O(n) space  
- **Alt:** Sort then neighbors — O(n log n), O(1) extra space  
- **Spring weave:** HashSet in one request ≠ unique across users → **DB unique constraint** is source of truth  

### Part 3 — Composition, `@Transactional`, Batch (concepts)

| Topic | Cheat line |
|---|---|
| Composition | Service **has** repos; Event **has** Venue. Don’t `extends` repository |
| `@Transactional` | On **service** unit of work; AOP **proxy**; commit OK / rollback on `RuntimeException` |
| Proxy trap | `this.otherMethod()` skips proxy → `@Transactional` on otherMethod may not run |
| Batch | Same Boot app + starter; **chunk** = transaction size, not whole Job |

### Weak spots

- Optional = empty box, not empty entity  
- PUT update = 200/204, not 201  

### 60-sec story

> "Events belong to venues. Create loads Venue by id, sets association, saves Event. List must not N+1 — fetch venue with the query. DTOs keep Jackson off lazy proxies."

---

## Week 1 Day 4 — Pagination + global exception handling

**Date:** 2026-08-20  
**Goal:** Page list of events; map not-found → 404 Problem Details.

### Quick recall

**`Page` vs `List`**  
- **General:** `List` is all rows or a slice with no totals. `Page` = `LIMIT`/`OFFSET` **plus** `COUNT` so the UI knows “page 3 of 50.” `Page.map` keeps metadata; `.stream().toList()` throws it away.  
- **Here:** `GET /api/events?page=0&size=20`.

**`@EntityGraph` (with paging)**  
- **General:** Tell JPA which associations to fetch with this query. JOIN FETCH + `LIMIT` can count joined rows, not parents.  
- **Here:** `@EntityGraph(attributePaths = "venue")` on `findAll(Pageable)`.  
- **Trap:** Wrong import `DataWebProperties.Pageable` — use `org.springframework.data.domain.Pageable`.

**`@RestControllerAdvice` + `@ExceptionHandler`**  
- **General:** Service throws a **domain** exception. It does not pick HTTP. Advice maps **type** → status + `ProblemDetail` (RFC 7807: `status`, `title`, `detail`). Unhandled → **500**. Same type, many messages → one handler. New **endpoint** → no new handler. New **type** → new handler. Class without advice is never called.  
- **Here:** `ResourceNotFoundException` → 404 for Event **and** Venue. Later 409/401 handlers.  
- **Trap:** Spring does not invent 404 from the class name. `@WebMvcTest` does not load advice unless `@Import`.

**404 vs 400 vs 409**  
- **General:** **404** = that resource does not exist. **400** = junk input (`@Valid`). **409** = valid and allowed, **conflicts with current state**.  
- **Here:** Unknown id 404; later email taken / sold out 409.  
- **Trap:** 409 ≠ only duplicate.

### What I built

- `GET /api/events` → `Page<EventResponseDto>` + `Pageable` (`page`, `size`, `sort`)
- `EventRepository`: `@EntityGraph(attributePaths = "venue")` + `Page<Event> findAll(Pageable)`
- `GlobalExceptionHandler`: `@RestControllerAdvice` + `@ExceptionHandler(ResourceNotFoundException)` → 404 `ProblemDetail`

### Under the hood (cheat)

**Pagination**  
Browser: `?page=0&size=20&sort=dateTime,desc`. Spring builds `Pageable`. Repo runs (1) select with LIMIT/OFFSET (2) COUNT. `Page` JSON has `content` + metadata. Without COUNT, client cannot know “page 3 of 50.”

**Why not JOIN FETCH + Pageable**  
Join can duplicate parent rows; LIMIT then means “20 joined rows,” not “20 events.” `@EntityGraph` is the usual paging-safe fetch.

**Wrong import trap (you hit this)**  
`org.springframework.data.domain.Pageable` — **not** `DataWebProperties.Pageable` (config class, same simple name).

**Exception as it was (before advice)**  
Service correctly threw `ResourceNotFoundException`. Spring does **not** read the class name and invent 404. Unhandled `RuntimeException` → **500**. Exception = domain signal; advice = HTTP translation.

**One handler per type, not per message**  
`"Event Not Found 99"` and `"Venue Not Found 5"` = same type → one method; text from `ex.getMessage()`. New **endpoint** → no new handler. New **exception type** (e.g. validation) → new `@ExceptionHandler`.

**`@ExceptionHandler` without `@RestControllerAdvice`**  
Plain class = Spring never calls it. Must be advice (global bean) or on a controller (local only).

**Trap you fell for**  
`VenueController` does **not** need its own 404 handler. Global advice already covers it.

### Interview answers (model) — gate

1. **`Page` not `List`?** Only load a slice (`LIMIT`); still know totals (`COUNT`) for UI pages. List is incomplete or loads everything.
2. **What in `@ControllerAdvice`?** Exception→HTTP handlers (`@ExceptionHandler`). Not business rules. (`@RestControllerAdvice` = advice + ResponseBody — that’s wiring, not the job description.)
3. **404 / 400 / 409?** Missing resource / invalid request (`@Valid`) / valid request but conflicts with current data (already booked, unique clash) — 409 ≠ only “duplicate.”
4. **RFC 7807?** Standard problem JSON (`status`, `title`, `detail`). Spring: `ProblemDetail`.
5. **Handle `ResourceNotFoundException`?** Throw in **service**; one `@RestControllerAdvice` maps type → 404 Problem Details; no try/catch per method.

### Weak spots (review before Day 5)

- Advice = **handlers**, not “ResponseBody” as the answer  
- 409 = **conflict**, broader than duplicate  
- Throw in service + global 404  

### Part 2 — LC 121 Best Time to Buy and Sell Stock

- **Pattern:** Running **min price so far** + `maxProfit` — one pass  
- **Complexity:** O(n) time, O(1) space  
- **Not** two pointers (array not sorted by value)  
- **Illegal:** global max of whole array if that day is **before** the buy  
- **Interview version:** two ints only (`minPrice`, `maxProfit`)  

### Part 3 — ISP + DIP

| Principle | Cheat line |
|---|---|
| ISP | Depend on smallest type you need — `JpaRepository` because list needs `Page` |
| DIP | Controller depends on **service abstraction**, not repository/JPA |
| Not DIP | Injecting repo into controller (glues HTTP to JPA; breaks SRP/layers) |
| Tests preview | `@WebMvcTest` mocks **service** (DTO in/out), not repository |

### 60-sec story

> "List returns `Page` with `Pageable`. LIMIT + COUNT. `@EntityGraph` for venue with paging. Not-found thrown in service; `@RestControllerAdvice` → 404 Problem Details for every controller."

---

## Week 1 Day 5 — Service interfaces + slice tests

**Date:** 2026-08-21  
**Goal:** Extract `EventService` / `VenueService` interfaces; `@WebMvcTest` + `@DataJpaTest`.

### Quick recall

**Interface + Impl**  
- **General:** Callers depend on a **contract** (DIP) so you can mock and later swap the impl. `@Service` stays on `*Impl`.  
- **Here:** `EventService` / `VenueService`; controllers inject the interface.

**`@WebMvcTest` vs `@SpringBootTest`**  
- **General:** `@WebMvcTest` = web **slice** (controller + MockMvc), no JPA. `@SpringBootTest` = whole context, slow; wiring / `contextLoads`.  
- **Here:** `EventControllerTest` is `@WebMvcTest(EventController.class)`.  
- **Trap:** Slice does not prove the DB.

**`@MockitoBean` (`@MockBean`)**  
- **General:** Replace a real bean with a Mockito fake **in the test context**. Stub/verify without the real DB/service. Boot 4 name: `@MockitoBean`.  
- **Here:** Fake `EventService` (later `BookingService`).

**`@DataJpaTest`**  
- **General:** JPA + H2, real repos, no web. Method runs in a transaction and **rolls back**.  
- **Here:** Save Venue then Event (`ManyToOne` required).  
- **Trap:** Tests live under `src/test/java`.

**What to mock**  
- **General:** Mock the **next layer down**. Controller test → service. Service test → repo. `@DataJpaTest` → usually no mocks.  
- **Here:** Same rule for booking tests (Week 2).

### What I built

- `EventService` / `VenueService` interfaces; `*Impl` keeps `@Service`; controllers inject the interface
- `EventControllerTest` (`src/test/java`): `@WebMvcTest` + `@MockitoBean EventService` + `MockMvc`
- Stub `getEvent(1L)` → `GET /api/events/1` → 200 + `jsonPath` (AssertJ / MockMvc matchers)
- `EventRepositoryTest`: `@DataJpaTest`; save `Venue` then `Event`; `findById` + AssertJ
- Tests live under `src/test/java` so `scope=test` deps resolve

### Under the hood (cheat)

- `src/main` ≠ test classpath; `@WebMvcTest` needs `src/test`
- Web slice does not start DataSource/JPA; stubbing the service is enough for HTTP mapping tests
- `@DataJpaTest` starts real JPA + in-memory H2; method runs in a transaction and rolls back by default
- `Event` requires a persisted `Venue` (FK / `ManyToOne optional = false`)
- Static imports: `get` / `status` / `jsonPath` from MockMvc; `assertThat` from AssertJ

### Interview answers (model)

1. **`@WebMvcTest` vs `@SpringBootTest`**  
   `@WebMvcTest` loads only the web layer (controllers + MockMvc). Collaborators like `EventService` are mocked — no full context, no JPA.  
   `@SpringBootTest` boots the whole application context (services, repos, DataSource, etc.). Heavier/slower; good for wiring checks like `contextLoads`, not for every controller test.

2. **`@MockBean` / `@MockitoBean`**  
   Replaces a real Spring bean with a Mockito mock in the test context so you can stub/verify without loading the real dependency (DB, heavy service, external API). Boot 4 name: `@MockitoBean` (same idea as older `@MockBean`).

3. **Testing pyramid (this project)**  
   Many fast narrow tests, fewer slow wide ones.  
   - Unit / slice: `@WebMvcTest` (mock service) or plain JUnit+Mockito on a service class  
   - Integration: `@DataJpaTest` (real JPA+H2) or `@SpringBootTest`  
   - E2E: running app + real HTTP (Postman/browser) through the full stack  

4. **Why service interfaces before a microservice split**  
   Interface = stable contract. Controller/tests depend on the abstraction (DIP) — easy to mock and to swap the impl later (in-process bean → remote client) without rewriting callers.

5. **What to mock**  
   Controller test → mock the **service**.  
   Service test → mock the **repository**.  
   `@DataJpaTest` → usually no mocks; real repo + H2.

### Weak spots (review)

| # | Topic | Notes |
|---|---|---|
| 1 | WebMvc vs SpringBootTest | Had WebMvc; learn full-context side |
| 2 | MockitoBean | Used it; name/purpose now in notes |
| 3 | Pyramid | Don’t call Day 4 “integration tests”; DataJpa/SpringBoot = integration |
| 4 | Interfaces / DIP | Say “contract + callers depend on abstraction,” not only “more independent” |
| 5 | What to mock | Strong — keep this line |

### Part 2 — LC 242 Valid Anagram

- **Passed.** HashMap char→count for `s` and `t`; length check first; compare with `intValue()`. O(n)/O(n).
- Not HashSet (no counts). Not sliding window (#242 is two whole strings).
- **Cousin:** anagram *inside* a longer string → sliding window (fixed slice; leftover = letters outside the window).
- Java: `map.get(c) != other.get(c)` compares Integer **objects**; use `intValue()` or `equals`.

### Part 3 — Design: oversell / book last seat

- Users: organizers (create) vs attendees (buy). Critical request = **book**, not browse. Oversell → **409**.
- `@Transactional` = one commit/rollback. Does **not** lock other requests by itself.
- Lock the **event row** (`PESSIMISTIC_WRITE` / `SELECT … FOR UPDATE`), not the table, not Venue.
- Two “Book now”: A locks, seats 1→0, commit. B was blocked on the same SELECT; continues in the **same** `book()`; reads **0** → 409. No UI refresh. No HashSet (one JVM / one request).
- Cadence: weekday Part 3 = **OOP + Design** (both); Friday = long SD; **Sat/Sun = off** (no mock).

### 60-sec story

> We depend on service interfaces for DIP and testability. `@WebMvcTest` checks HTTP with a mocked service; `@DataJpaTest` checks persistence on H2. Mock the next layer down: controller→service, service→repo. Booking must not oversell: transaction plus a row lock on that event; the second buyer waits, then sees zero seats.

---
