# Interview notes — Week 9

Added by Anton on 2026-09-25.
Goal: good working knowledge of Spring Boot interview questions, Hibernate, Docker, Kubernetes, whiteboard HLD, and AI for a Java developer (Spring AI).

Same day order: Part 1, then Part 2, then Part 3.
Sat/Sun **off**.

**Day 1 Part 1 closed** (Mon 2026-10-12).
#17 overtime, coach patched the next digit.
#637 on-time, he coded.
S1 and S2 taught.

**Day 2 closed** (Tue 2026-10-13).
#22 Generate Parentheses, coach-written, untimed.
#108 Convert Sorted Array to BST, coach-written, untimed.
S3 and S4 were not reached.
Part 2 was the event-list query count and the Book transaction.
Part 3 waiting room is closed.
The AI-for-developers slot is scrapped.

## Week 9 Day 1 Part 2 — persistence context (H1)

Anton asked for this written out (2026-10-12). H2 (lazy loading) is not started.

Hibernate is the library that turns an `Event` object into a row in the `events` table, and a row back into an `Event`. Spring Data calls Hibernate when the repository runs. You do not write the `UPDATE` yourself.

The persistence context is Hibernate's memory of entity objects for **one database transaction**. Spring opens that memory when a `@Transactional` method starts and closes it when the method commits or rolls back. One transaction, one persistence context. The next request gets its own.

Three different things that get mixed up:

| Thing | What it is |
|---|---|
| HTTP request | The phone's call. It is not Hibernate's memory. |
| `events` table | The rows on disk. |
| `Event` class | The Java shape of one concert. |
| Persistence context | The copies Hibernate is **watching** while this transaction is open. |

### Four states of an `Event` object

| State | When | What `setTitle` does |
|---|---|---|
| **Transient** | `new Event()`. No id. Hibernate is not watching it. `createEvent` is here until `save()`. | Changes the Java field only. Nothing is in the database yet. |
| **Managed** | Loaded or saved **inside** the open transaction. `updateEvent`'s `findById` result is this. | Hibernate is watching. On commit it writes an `UPDATE` if a field changed. |
| **Detached** | It has an id, but the transaction that was watching it is **over**. The object you still hold after `updateEvent` returns is this. | Changes the Java field only. The database does not change. |
| **Removed** | Marked deleted inside the transaction. | The `DELETE` goes out on commit. |

### What `updateEvent` does now

`EventServiceImpl.updateEvent` is `@Transactional`. It loads the event, calls `setTitle`, and does **not** call `save()`. The response is built from that same `event`.

1. `findById` runs one `SELECT` and puts that `Event` into the persistence context. Hibernate stores a snapshot of the fields from that `SELECT`. The object is **managed**.
2. `setTitle` changes the managed object. No SQL yet.
3. `save()` is how you hand Hibernate an object it is **not** already watching. That is `createEvent`: a transient `Event` becomes managed and is inserted. A managed event does not need `save()` to be updated.
4. When `updateEvent` returns, the transaction commits. Hibernate compares the object to the snapshot. The title differs, so Hibernate writes the `UPDATE`. That comparison is **dirty checking**.

Anton asked how it saves with `save()` removed. That is step 4.

### First-level cache

The persistence context is also a map from id to that Java object. This is the first-level cache. It lives only for this transaction. It is not Redis and not Spring `@Cacheable`.

A second `findById` for the same id, still inside `updateEvent`, does **not** send a second `SELECT`. It returns the same object. The title on it is already the new one, even though the `UPDATE` has not been sent. The `UPDATE` waits until the method finishes.

Anton: the second load returns the updated title. Right.

### After the method returns

The transaction is closed, so that object is **detached**. Another method still holding it and calling `setTitle` does not change the database. Anton: "no it should not." Right.

### `merge`

`merge` is for an `Event` that already has an id but is detached. It copies that object's fields onto the `Event` Hibernate is watching in the **current** transaction, and it **returns the watched one**. The object you passed in stays detached.

On commit, the database has the title that was copied onto the **returned** object. A `setTitle` on the object you passed in does not get written. Change the object `merge` returned if the commit should see the new title.

Anton (2026-10-12): thought Rock would be saved, because the method is `@Transactional` and the persistence context would see a mismatch. The transaction opens the context. It does not enroll the argument. `save` returns a second Java object, and that one is the managed copy. `passedIn` and `returned` share an id and are not the same object. `setTitle("Rock")` on `passedIn` leaves the managed title at Jazz, so dirty checking finds no mismatch. Commit writes Jazz. `setTitle("Rock")` on `returned` would be the mismatch, and commit would write Rock.

### Flush

Hibernate sends the pending `UPDATE` when the transaction commits. It also sends it **before** a query that has to read the new title from the database (a search). `findById` for an id already in this persistence context does not need that: it uses the object in memory.

Anton (2026-10-12): thought the search would not see an `UPDATE` yet, and that the answer would come from the persistence context. A search is SQL. It does not return the managed object the way `findById` does. Hibernate flushes first, so the `UPDATE` is already in the database when the `SELECT` runs, and the search finds the new title.

**Line:** Hibernate watches a managed `Event` for the transaction and writes the `UPDATE` on commit, so a title change on an event I just loaded does not call `save()`. After the method returns the object is detached, and `merge` returns the watched copy.

## Week 9 Day 1 Part 2 — lazy venue (H2)

`Event.venue` is `LAZY`. `spring.jpa.open-in-view=false`. `getEvent` used `findById`, then `getVenue().getName()`. The repository session was already closed, so the phone got **500**. `GlobalExceptionHandler` does not catch `LazyInitializationException`.

Fix: `EventRepository.findByIdAndVenueId` is `JOIN FETCH e.venue` for that one id. `getEvent` calls it. The venue fields are already on the object when the session closes, so `getName()` does not run a new `SELECT`. Anton: "in the object, the join already loaded it." Right.

`getReferenceById` is not used in this app. It returns a proxy, not an `Optional`, and not null. A missing id does not throw until a real field is read.

**EAGER:** `@ManyToOne` is eager unless you set `LAZY`. Making `venue` `EAGER` would also kill the 500, because every `Event` load would bring the venue inside the session. Anton: more database work, more data loaded and held, often unused. Right. `reserveSeats` only changes `availableSeats`. With `EAGER` that lock query still loads the venue row. `JOIN FETCH` on `getEvent` pays for the venue only on the call that shows the name.

## Week 9 Day 2 Part 2 — event list query count and the Book transaction

Coach wrote this in the app (Anton 2026-10-13). He asked what was new, then why `REQUIRED` is on `writeBook` when nothing is already open.

The event list used to load the venue with the page, and Hibernate pulled every row. It now loads a page of event ids, then those events and their venues in one query. Five events, page of two, is three SQL statements. `EventListQueryCountTest` covers that. An empty id page returns without calling the venue query.

`getEvent` and the event list are `@Transactional(readOnly = true)`. Hibernate skips dirty checking on those reads. "My tickets" calls auth first, then reads the rows in its own read-only transaction.

`book()` has no transaction. It waits on Auth and Event first. `writeBook` is the method that opens one. The ticket and both outbox rows commit or roll back together inside that one transaction. If the Book rolls back, the outbox rows roll back with the ticket.

A plain `@Transactional` is enough. `Propagation.REQUIRED` is already the default, so writing it out changes nothing. `REQUIRED` means join a transaction if one is already open, otherwise start one. None is open on `book()`, so `writeBook` starts it.

The event service has an in-memory `CacheManager` bean named `events`, so `@Cacheable("events")` has a cache to use.

**Line:** I page the ids first, then load that page with the venue in one query, and the ticket and the outbox share the transaction that `writeBook` starts.

Anton (2026-10-13): Friday closes only what is still unfinished in the app. Do not add another topic that day.

Still open after Tuesday:

- Mapping, cascade, ids, batch inserts, locking.
- Multi-stage Docker image and a memory limit in the container.
- Gateway rate limit: extra Book taps get **429** before `book()`. Sold out stays **409**.
- Kubernetes: Deployment, Service, ConfigMap, Secret, probes, limits, rolling update, HPA.
- One `ChatClient` endpoint and one read-only tool. No tool that books.
- Click id on Book, so a retry of the same tap is one ticket.

## Week 9 Day 1 Part 3 — payment for Book

Charge runs after Event commits and releases the row. Same reason as notification: the bank is slow, and the seat lock must not wait on it. Anton got that.

The bank’s “paid” call can arrive twice. A stored flag for that payment must stop a second ticket and seats taken again. Anton first said “don’t return the confirmation twice.” The confirmation is harmless. The ticket and the seats are the thing that must not repeat.

No “paid” call before the deadline: revert the reservation. Maria has no ticket. If the bank did take the money and the call was lost, ask the card company about the payment id before releasing the seats. Paid → finish the ticket. Not paid → release.

Charge failed is not sold out. Maria hears that the payment did not process. Seats go back.

A special URL does not prove the caller. The bank signs the body. A bad signature is ignored.

**Line:** I charge after the seat row is committed. A repeated “paid” call does not book again. I verify the signature, and I ask the bank about the payment id before an expiry job gives the seats back.

## Week 9 Day 1 Part 3 — LLM basics

Taught before any question (Anton, 2026-10-12). He had not seen this box.

A chat model is a separate box from Event and Booking. It writes the next text. It does not run `findById` and it does not see the `events` table unless those rows are pasted into the text you send.

**Tokens.** The model does not read words. It reads tokens, chunks of about a word or a piece of a word. You pay and you wait per token, in and out.

**Context window.** One call can hold only so much text. That limit is the context window. Your whole database does not fit. You send the question plus the few rows that matter.

**Temperature.** A knob on how much the model varies the wording. 0 is the steady setting. It still does not look up your table. It can name a jazz concert in Haifa that you do not have. That made-up fact is a hallucination. Steady wording is not a database read.

**Cost and latency.** Each call costs money and time. It does not belong inside `book()` or inside the seat lock. Search or “describe this event” can call it. Taking seats cannot wait on it.

**Structured output.** You can require the reply to be JSON of a fixed shape, and map that JSON onto a Java record (`city`, `date`). An essay is not something `book()` can trust.

**Line:** Temperature 0 does not stop the model from inventing a concert. I only trust it when I pasted the real rows, or when the answer is a record I validate. The model stays off the Book hot path.

---

## Part 1 — LC coding + Spring Boot questions

Two coding LCs each weekday, Medium then Easy.
Top Interview 150 only.
Same gate, same timer (Medium 25, Easy 15).

Mon–Thu: after the two LCs, **~20 min, two Spring Boot interview questions** instead of LC-SD.
Friday: after the two LCs, **~15 min rapid-fire** of the rest of the bank (one line each, he answers, coach corrects).
Anton asked (2026-09-25) to make sure the Spring Boot questions are fully covered.
By Friday every row in the bank below is either asked or rapid-fired.

Spring question format: coach says the topic and one sentence why for the role.
Short explain.
Then the question.
**He talks first.**
Then trap + one interview sentence.
Stop.
No code unless he asks to see a snippet.
Tie every answer to the booking app where it fits.

| Day | Medium | Easy | Spring Boot questions |
|---|---|---|---|
| Mon | [#17 Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) | [#637 Average of Levels in Binary Tree](https://leetcode.com/problems/average-of-levels-in-binary-tree/) (W6 leftover) | S1 Auto-configuration + S2 Configuration and profiles |
| Tue | [#22 Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | [#108 Convert Sorted Array to BST](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree/) | S3 Proxies and AOP + S4 Bean scopes, lifecycle, circular dependencies |
| Wed | [#46 Permutations](https://leetcode.com/problems/permutations/) | [#392 Is Subsequence](https://leetcode.com/problems/is-subsequence/) | S5 Request path + S6 Errors and validation |
| Thu | [#39 Combination Sum](https://leetcode.com/problems/combination-sum/) | [#14 Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/) | S7 Up-to-date Boot 3 + S8 Testing slices |
| Fri | [#208 Implement Trie](https://leetcode.com/problems/implement-trie-prefix-tree/) | [#530 Minimum Absolute Difference in BST](https://leetcode.com/problems/minimum-absolute-difference-in-bst/) | Rapid-fire R1–R8 |

### Spring Boot bank — deep questions (Mon–Thu)

| # | Topic | What he must be able to say |
|---|---|---|
| S1 | Auto-configuration | `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Starters bring jars; auto-config classes are `@Conditional` (on class, on property, on missing bean). Your own bean wins (`@ConditionalOnMissingBean`). How to exclude one and how to see the report (`--debug`). |
| S2 | Configuration and profiles | `application.yml` vs profile files, `spring.profiles.active`. `@Value` vs `@ConfigurationProperties` (typed, validated, grouped). Precedence: env var / command line beat the file, which is how Docker and Kubernetes override config. Secrets never in the repo. |
| S3 | Proxies and AOP | Spring wraps the bean in a proxy for `@Transactional`, `@Async`, `@Cacheable`, security. Self-call skips the proxy. JDK proxy (interface) vs CGLIB (subclass). `private` / `final` methods cannot be advised. What an `@Aspect` with a pointcut is, one example (timing `book()`). |
| S4 | Bean scopes, lifecycle, circular dependencies | Singleton (default), prototype, request, session. Prototype injected into a singleton is created **once** (fix: `ObjectProvider`). Lifecycle: constructor → injection → `@PostConstruct` → use → `@PreDestroy`. `BeanPostProcessor` is where proxies are made. Circular dependencies fail by default since Boot 2.6; fix the design, not `@Lazy`. |
| S5 | Request path | Filter chain (security) → `DispatcherServlet` → handler mapping → argument resolvers (`@PathVariable`, `@RequestParam`, `@RequestBody`) → message converters (Jackson) → controller → return value → converter → response. `@RestController` = `@Controller` + `@ResponseBody`. |
| S6 | Errors and validation | `@RestControllerAdvice` + `@ExceptionHandler`, `ProblemDetail` (RFC 7807) in Boot 3. `@Valid` on `@RequestBody` → `MethodArgumentNotValidException` → 400. `@Validated` on a class for method params. Domain exceptions → 404 / 409, never a 500 with a stack trace. |
| S7 | Up-to-date Boot 3 | Java 17+ baseline, `jakarta.*` namespace. Virtual threads (`spring.threads.virtual.enabled`). `RestClient` (new sync) vs `WebClient` (reactive) vs `RestTemplate` (maintenance). Micrometer observations and tracing. Native image in one sentence (fast start, slow build, reflection limits). Docker Compose support and Testcontainers `@ServiceConnection`. |
| S8 | Testing slices | `@SpringBootTest` (whole context) vs `@WebMvcTest` (web layer, mock the service) vs `@DataJpaTest` (JPA + rollback per test). `@MockitoBean` replaced `@MockBean` in Boot 3.4. Testcontainers for a real Postgres instead of H2. What each test proves (W4 Thu: mock vs HTTP test). |

### Spring Boot bank — rapid-fire (Friday, one line each)

| # | Question | One-line answer |
|---|---|---|
| R1 | `@Component` vs `@Service` vs `@Repository` vs `@Controller`? | Same scan; `@Repository` also translates DB exceptions to `DataAccessException`. |
| R2 | `@Bean` vs `@Component`? | `@Bean` = method that builds a bean you do not own (W6 Factory). `@Component` = your class, found by scan. |
| R3 | How does `@Async` run, and what is the trap? | Needs `@EnableAsync`, runs on a task executor; default pool size and self-call both bite. Define your own executor. |
| R4 | `@Scheduled` on two pods? | Runs on **both**. Need a lock (ShedLock) or one worker. |
| R5 | Why is CSRF off for this API? | Stateless JWT in a header, no session cookie to steal. On for cookie sessions. |
| R6 | `@PreAuthorize` vs URL matchers? | Method-level rule on the service vs path rule in the chain. Needs `@EnableMethodSecurity`. |
| R7 | What does a Boot fat jar contain, and graceful shutdown? | App + deps + embedded Tomcat, `java -jar`. `server.shutdown=graceful` finishes in-flight requests before the pod dies. |
| R8 | Spring Data: derived query vs `@Query` vs projection? | `findByCityAndDateAfter` from the name; `@Query` for JPQL / native; interface or record projection to load only needed columns. |

Already covered in earlier weeks, recall only if he stumbles: constructor DI (W1), filter chain order (W2), `@Cacheable` (W5), Actuator health (W5), Factory `@Bean` (W6).

Same-week extras if he skips one: [#79 Word Search](https://leetcode.com/problems/word-search/), [#77 Combinations](https://leetcode.com/problems/combinations/), [#228 Summary Ranges](https://leetcode.com/problems/summary-ranges/).

Already done in Weeks 1–2, do not pick: #20, #155, #150, #242, #121, #232, #739, #1.

Stubs: `LC-Practice/src/main/java/questions/week09/`.
Tests: `src/test/java/test/week09/`.

### Spring Boot question traps

| Topic | Trap |
|---|---|
| Auto-config | "Boot scans every jar and makes beans of everything." It only applies `@Conditional` configs, and backs off when you define your own bean. |
| Proxies | "`@Transactional` on a private helper called from `book()` works." The call never goes through the proxy. |
| Request path | "The controller runs first, then security." The filter chain runs before `DispatcherServlet`. A 401/403 never reaches `@ControllerAdvice`. |
| Boot 3 | "Virtual threads make `FOR UPDATE` faster." They make waiting cheaper for the thread, not for the DB row or the pool. |
| Config | `@Value` scattered in ten classes, or the JWT secret committed in `application.yml`. |
| Scopes | "Prototype bean inside a singleton is new on every call." It is created once, at injection. |
| Circular dependency | Fixing it with `@Lazy` or field injection. The two services need a third piece or one less arrow. |
| Validation | `@Valid` missing, so bad JSON reaches `book()`. Or a 500 for a validation error. |
| Testing | `@SpringBootTest` for everything (slow), or H2 passing while Postgres `FOR UPDATE` behaves differently. |

---

## Part 2 — Hibernate, Docker, Kubernetes, Spring AI (booking app)

Same Part 2 protocol ([ai-spring.md](ai-spring.md)).
He types the code in [SpringBoot-bookingApp](https://github.com/antondahdal/SpringBoot-bookingApp).
Checks are on the interview idea, not Maven or imports.

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | **H1 Hibernate — persistence context** | **H2 Hibernate — lazy loading and proxies** |
| Tue | **H3 Hibernate — N+1 deep and proving query count** | **H4 Transactions with Hibernate** |
| Wed | **H5 Hibernate — mapping, cascade, IDs, batching, locking** | **Docker:** multi-stage `Dockerfile` (slim JRE, non-root, `.dockerignore`), layer cache order, JVM memory in a container (`-XX:MaxRAMPercentage`) |
| Thu | **Kubernetes (local kind or minikube):** `Deployment` + `Service` for Booking and Event, `ConfigMap` + `Secret` for DB URL / JWT key | **Kubernetes:** liveness / readiness probes on Actuator (W5 Thu talk becomes YAML), resource requests / limits, rolling update, what an HPA scales on |
| Fri | **Spring AI lab:** `ChatClient` endpoint in the app (for example "describe this event" or natural-language event search), model behind config (Ollama local or OpenAI key) | One read-only **tool** the model can call (find events by city / date). No tool that books. |

Compose was **W5 Thu**.
Health vs ready was **W5 Thu**.
Do not rerun those as new; build on them.

### Also this week — one database per service, and a rate limiter (Anton 2026-10-08)

These ride on the deploy days (Wed Docker, Thu Kubernetes). They do not replace a Hibernate topic.

From the W8 split, still unbuilt:

- One database per service. Auth, Event, and Booking stop sharing one schema.
- Three deployables, each its own container.
- JWT: Auth issues it. Event and Booking only check it.

Rate limiter, same work. Nothing in the repo returns **429** today.

- It sits on the gateway, in front of the pods.
- Extra Book taps get **429** there. They never reach `book()` and never lock the seat row.
- Taps that were let in, and the seats are gone, still get **409** from Event.
- The load balancer only picks a pod. A queue is the line where a phone waits for a worker. The limiter is the gate that refuses the overflow.
- Fixed window vs token bucket was Part 1 W7 Tue. Do not re-ask the algorithm names.
- This is the Book gate. It is not the Friday AI-gateway limit.

Anton asked (2026-09-25) to go deeper on Hibernate.
Hibernate now takes five topics (Mon, Tue, Wed first half).
Docker is one topic on Wednesday.
Every Hibernate topic turns on SQL logging first, so he **sees** the queries instead of guessing.

### Hibernate deep dive — what each topic covers

**H1 Persistence context (Mon).**
Transient, managed, detached, removed.
The persistence context is the first-level cache: same id in one transaction → same Java object, one `SELECT`.
Dirty checking: change a managed entity, commit, and Hibernate writes the `UPDATE` with no `save()`.
Flush happens before commit and before a query that could see the change.
`persist` vs `merge`: `merge` copies a detached object onto a managed one and **returns the managed copy**.
Lab: load an `Event`, change the title inside `@Transactional`, no `save()`, watch the `UPDATE` in the log.

**H2 Lazy loading and proxies (Mon).**
`@ManyToOne` / `@OneToOne` are eager by default; set them `LAZY`.
`@OneToMany` / `@ManyToMany` are lazy by default.
A lazy field is a Hibernate proxy until touched.
`findById` hits the DB now; `getReferenceById` returns a proxy and may fail later.
`LazyInitializationException` = touching the proxy after the session closed.
`spring.jpa.open-in-view=false` so that bug shows in tests instead of hiding as extra queries in the view.
Fix: fetch what you need in the query, or map to a DTO inside the transaction.
Lab: turn open-in-view off, hit the endpoint, see the exception, fix it with a query.

**H3 N+1 deep and proving query count (Tue).**
Start from W7's N+1, then go further.
Fetch join (`JOIN FETCH`), `@EntityGraph`, `@BatchSize` / `default_batch_fetch_size`: when each fits.
Two `List` collections fetched together → `MultipleBagFetchException` (use `Set` or two queries).
Fetch join + pagination → Hibernate pages **in memory** (warning HHH90003004); page ids first, then fetch.
DTO / record projection when the screen needs four columns.
Prove it: `spring.jpa.properties.hibernate.generate_statistics=true`, or a test that asserts the query count.

**H4 Transactions with Hibernate (Tue).**
Persistence context lives as long as the transaction.
Propagation `REQUIRED` (join) vs `REQUIRES_NEW` (separate commit, separate connection).
Outbox row must be `REQUIRED` with the Book.
An audit log that must survive a rollback may be `REQUIRES_NEW`.
`readOnly = true`: no dirty checking, flush mode manual, may route to a replica.
Rollback: unchecked by default, checked only with `rollbackFor`.
Isolation levels in one line each (read committed default on Postgres).
`@Transactional` on a controller or on a private method (proxy trap, S3 in Part 1).

**H5 Mapping, cascade, IDs, batching, locking (Wed).**
Owning side vs `mappedBy`: only the owning side writes the foreign key.
Cascade vs `orphanRemoval`: cascade passes the operation; orphan removal deletes a child dropped from the list.
Never `CascadeType.REMOVE` on `@ManyToMany`.
Entity `equals` / `hashCode` with a null id before persist (W5 recall): use a business key or a constant `hashCode`.
IDs: `IDENTITY` turns off JDBC batch inserts; `SEQUENCE` with `allocationSize` keeps them.
Batch inserts: `hibernate.jdbc.batch_size` + `order_inserts`.
Locking recap from W3 in Hibernate words: `@Version` → `OptimisticLockException`; `PESSIMISTIC_WRITE` → `SELECT … FOR UPDATE`.
Second-level cache vs Spring `@Cacheable`: one line, when you would not use it (hot, changing rows like seats).

### Part 2 traps

| Topic | Trap |
|---|---|
| Dirty checking | Calling `save()` on a managed entity "to make it update". It already updates on commit. |
| Lazy loading | Fixing `LazyInitializationException` with `EAGER` everywhere. Fix the query (fetch join / entity graph) or map to a DTO inside the transaction. |
| `REQUIRES_NEW` | "Outbox in `REQUIRES_NEW` is safer." Then a rolled-back Book still leaves an outbox row. Outbox must share the Book transaction. |
| `merge` | Keeping the object you passed to `merge` and changing it. The managed one is the return value. |
| `getReferenceById` | Using it to "check the event exists". It does not hit the DB until touched. |
| Fetch join + page | `JOIN FETCH` with `Pageable` looks fine in dev and loads the whole table in memory in prod. |
| Two bags | Fetch-joining two `List` collections at once → `MultipleBagFetchException`, or a cartesian blow-up. |
| IDs | `IDENTITY` and then wondering why `batch_size` does nothing on inserts. |
| Cascade | `CascadeType.ALL` on `@ManyToOne` (deleting a booking deletes the event). |
| `readOnly` | Thinking it blocks writes in the DB. It is a hint to Hibernate and the driver. |
| Docker | `latest` JDK image with the build tools inside the runtime image. |
| Rate limiter | **429** from `book()` or from the seat lock. Or **429** when the concert is sold out. Sold out stays **409**. The limiter is on the gateway. |
| Kubernetes | Liveness checks the DB. DB blip → every pod restarts. DB belongs in readiness. |
| Kubernetes | Secret in a `ConfigMap`. Or a Secret thought of as encrypted (it is base64 by default). |
| Spring AI | Letting the model call `book()`. Money and seats need a human confirm, and the model output is not trusted input. |

---

## Part 3 — HLD whiteboard + AI for developers

Anton 2026-10-13: scrap the AI-for-developers slot. Do not teach ChatClient, RAG, agents, or the Friday AI assistant. The app boards stay: waiting room, search, deploy.

**Tue Part 3 waiting room closed** (2026-10-13). Line is outside Booking. Entry token lasts a few minutes and does not take a seat. `book()` with 0 seats is **409**. A phone that never reaches `book()` times out. Ten more Booking servers still sell 5,000 seats, because the seats are one event row.

He first said **429** when `book()` saw 0 seats. **429** is the line turning a phone away before `book()` runs. He then said a phone still in the line hears **409**. That phone never calls `book()`, so the line drops it when the wait expires.

| Piece | Time | What |
|---|---|---|
| HLD whiteboard | ~40 min | Boxes, data, sequence, 10×, what breaks. Five families from [design-map.md](design-map.md). |
| AI for developers | ~20 min | Explain a bit, then **he talks first**. Trap + one interview sentence. |
| Friday | 60–75 min | Long HLD of an AI feature on this app. |

| Day | HLD whiteboard (~40) | AI for developers (~20) |
|---|---|---|
| Mon | **Payment for Book:** Stripe-style provider, webhook back, idempotency key, ledger row, reconcile job. What if the webhook arrives twice or never | **LLM basics for engineers:** tokens, context window, temperature, why it hallucinates, cost and latency per call, structured output (JSON schema → Java record) |
| Tue | **Flash sale waiting room:** **Done** 2026-10-13 | **Scrapped** |
| Wed | **Event search:** search index (OpenSearch) fed from the outbox / CDC, stale-OK browse vs Book truth, reindex | **Embeddings + RAG:** chunking, embedding model, vector store (pgvector), top-k, answer with sources. RAG vs fine-tune. When not to use AI at all |
| Thu | **Deploy board:** this app on Kubernetes. Ingress, services, HPA, DB outside the cluster, config / secrets, rolling deploy, zero-downtime DB migration (expand, then contract) | **Tools, MCP, agents, guardrails:** function calling, MCP servers, agent loop. Prompt injection, least-privilege tools, human confirm, evals. How to use AI coding tools well (small diffs, tests, review every line) |
| Fri | **Long HLD — AI assistant for the booking app:** user asks "two seats for jazz in Haifa next Friday". RAG over events, read-only tools, confirm step, then normal Book path. LLM gateway: rate limit, timeout, fallback, cost cap, caching, logging prompts without PII | 2 min recap of the week's AI traps |

### Part 3 traps

| Topic | Trap |
|---|---|
| Payment | Charge inside the `FOR UPDATE` transaction. Or trusting the webhook without verifying the signature. |
| Waiting room | "Just add more Booking pods." The single event row is still one row. |
| Search | Book reads seats from the search index. Index is a copy; Book is truth. |
| Deploy | Rename a column in one deploy while old pods still run. |
| LLM basics | "Temperature 0 means it never makes things up." |
| RAG | Stuffing the whole DB into the prompt. Or no source shown to the user. |
| Agents | Tool with write access and no confirm. Text in a retrieved document can steer the model (prompt injection). |
| AI HLD | LLM call on the Book hot path. Book stays the same `book()`; AI only prepares the request. |

### Interview sentences to own by Friday

> Hibernate tracks managed entities and flushes changes on commit, so I fix N+1 with a fetch join or entity graph, not with EAGER.

> I keep associations lazy, turn open-in-view off, and load what the screen needs in one query or a DTO projection, and I prove it with the query count.

> Spring Boot auto-configures beans only when their conditions match and backs off when I define my own; everything like `@Transactional` works through a proxy, so a self-call skips it.

> Liveness says restart me, readiness says send me traffic; the DB check goes in readiness.

> In Spring AI I call the model through `ChatClient`, map the answer to a record, and give it only read-only tools; anything that books or charges needs a user confirm.

> RAG means I embed my own data, retrieve the top matches, and make the model answer from them with sources.
