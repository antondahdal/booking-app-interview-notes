# Interview notes — Week 9

Added by Anton on 2026-09-25.
Goal: good working knowledge of Spring Boot interview questions, Hibernate, Docker, Kubernetes, whiteboard HLD, and AI for a Java developer (Spring AI).

Same day order: Part 1, then Part 2, then Part 3.
Sat/Sun **off**.

**Not started yet.** Say `Start Week 9 Day 1` when you get here.

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
| Kubernetes | Liveness checks the DB. DB blip → every pod restarts. DB belongs in readiness. |
| Kubernetes | Secret in a `ConfigMap`. Or a Secret thought of as encrypted (it is base64 by default). |
| Spring AI | Letting the model call `book()`. Money and seats need a human confirm, and the model output is not trusted input. |

---

## Part 3 — HLD whiteboard + AI for developers

OOP still-need list is empty, so the OOP slot becomes the AI talk this week.

| Piece | Time | What |
|---|---|---|
| HLD whiteboard | ~40 min | Boxes, data, sequence, 10×, what breaks. Five families from [design-map.md](design-map.md). |
| AI for developers | ~20 min | Explain a bit, then **he talks first**. Trap + one interview sentence. |
| Friday | 60–75 min | Long HLD of an AI feature on this app. |

| Day | HLD whiteboard (~40) | AI for developers (~20) |
|---|---|---|
| Mon | **Payment for Book:** Stripe-style provider, webhook back, idempotency key, ledger row, reconcile job. What if the webhook arrives twice or never | **LLM basics for engineers:** tokens, context window, temperature, why it hallucinates, cost and latency per call, structured output (JSON schema → Java record) |
| Tue | **Flash sale waiting room:** 1M users for 5k seats. Virtual queue, entry token with TTL, protect Booking and the event row | **Spring AI map:** `ChatClient`, prompt templates, advisors (memory, logging), structured output, switching model by config. Where LangChain4j fits |
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
