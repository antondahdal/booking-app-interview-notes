# Cross-cutting cheat (all weeks so far)

### Request path (happy path)

```
HTTP → Controller (@Valid DTO) → Service (map + rules + orElseThrow)
     → Repository → DB → Entity → Service maps → Response DTO → JSON
```

### Error path (not found)

```
Service throws ResourceNotFoundException
  → @RestControllerAdvice @ExceptionHandler
  → 404 ProblemDetail
```

### Status cheat

| Code | Meaning | Example |
|---|---|---|
| 200 | OK read/update | GET event |
| 201 | Created | POST venue/event |
| 204 | No body | DELETE / some PUT |
| 400 | Bad input | `@Valid` fail |
| 401 | Not authenticated | No / bad login |
| 403 | Authenticated, not allowed | Attendee hits `POST /api/events` |
| 404 | Missing | Unknown id |
| 409 | Conflict with **current state** (valid request, cannot apply it now) | Email taken; last seat gone; already booked — not “duplicate only” |
| 502 | Downstream failed / hang (we got no good answer) | Event 5xx or WebClient timeout → `DownstreamServiceException` |
| 500 | Unhandled server bug | Forgotten exception handler |

**`@Transactional` vs lock (W4 Fri):** wrap = commit all / roll back all. **Not** `FOR UPDATE`. The open transaction still **holds a pool connection**. Don’t keep it open across HTTP `.block()`. Event’s row lock is Event’s DB.

## Dependency injection (interview)

**What it is:** you do not `new` your collaborators (`new UserRepository()`). You declare what you need. Spring’s **IoC container** creates the objects (**beans**) and **passes them in**. That passing-in is injection.

**What happens on startup:** `@SpringBootApplication` → component scan → find `@Service` / `@Configuration` `@Bean` / controllers → create beans → wire constructors → start Tomcat. If a required bean is missing → fail at **startup**, not on the first request (constructor injection).

### The three types (memorize all three; prefer one)

| Type | How | Use when | Downside |
|---|---|---|---|
| **Constructor** | Deps as constructor args. Fields `final`. | **Always prefer** for required deps (`UserRepository`, `PasswordEncoder`). | Circular constructors can fail (A needs B, B needs A). Rare if layers are clean. |
| **Setter** | `@Autowired` on `setX(...)`. | Optional deps you might swap after construct. | Object can exist **half-wired**. Fields not `final`. Easy to forget a setter in tests. |
| **Field** | `@Autowired` on the field. | Almost never in production code. | Hidden deps, **cannot be `final`**, needs reflection, painful to unit-test (`new AuthServiceImpl()` does not fill the field). |

**What you say in the interview:**  
“I use constructor injection. Required dependencies are obvious, fields can be `final`, and in a test I pass fakes into `new AuthServiceImpl(fakeRepo, fakeEncoder)` with no Spring. Field `@Autowired` hides what the class needs and is harder to test.”

**`@Autowired` today:** if the class has **one** constructor, Spring injects it **even without** `@Autowired`. You can omit it. Multiple constructors → mark one with `@Autowired` (or better: only keep one).

**Two beans of the same type:** Spring does not know which to inject → startup error. Fix with `@Qualifier("beanName")` on the injection point, or `@Primary` on the bean you want by default. `@Bean` method name is the default bean name (`passwordEncoder` vs a method named `BCryptPasswordEncoder`).

**`@Component` / `@Service` / `@Repository` vs `@Bean`:**  
Stereotypes = “scan this **class**, Spring calls the constructor.”  
`@Bean` = “run this **method**, use the return value” — for objects you cannot annotate (library types like `BCryptPasswordEncoder`).

**Singleton + request data (Week 2 Day 1):** default scope is **one instance per context**. Do **not** put “current user” on `AuthService`. That field is shared by every request (race / mixed users). Keep the user in method args. Stateless beans (`PasswordEncoder`) should be singleton.

**60-sec DI story**

> Spring creates beans and injects them. I prefer constructor injection: immutable, testable, fail-fast. I do not use field injection. Encoder is a `@Bean` because it is a library class, not our code. Services are singletons — no request state on fields.

---

## Annotations (interview cheat)

Skim this before interviews. **Why** matters more than the name. Tables below are the job; extra “say this” lines are the interview answer.

### Lombok — constructors

| Annotation | What it generates | When you use it |
|---|---|---|
| `@NoArgsConstructor` | `public Venue() {}` | **JPA entities** (Hibernate `new` then set fields). **Request DTOs** (Jackson `new` then setters from JSON). |
| `@AllArgsConstructor` | One constructor with **every** field | **Response DTOs** — service does `new UserResponseDto(id, email, role)`. No setters → JSON out is read-only. |
| `@Getter` / `@Setter` | getX / setX | Jackson reads getters for JSON out; setters for JSON in. Entities need both for JPA. |

**Request vs response (memorize):**  
Jackson inbound needs **empty object + setters** → `@NoArgsConstructor` + `@Setter`.  
You **build** the response in Java → `@AllArgsConstructor` + `@Getter` is enough.  
JPA spec: entity must have a no-arg constructor. If you only add `@AllArgsConstructor`, Lombok **removes** the default no-arg unless you also add `@NoArgsConstructor`.

### JPA / model

| Annotation | Job |
|---|---|
| `@Entity` | This class is a table row. Hibernate manages it. |
| `@Table(name = "users")` | Real table name. Use `users` — `user` is a reserved SQL word. |
| `@Id` | Primary key. |
| `@GeneratedValue(IDENTITY)` | DB auto-increments the id on INSERT. Java does not invent it. |
| `@Column(nullable = false, unique = true)` | DB NOT NULL + unique constraint (email). Unique ≠ HTTP 409 by itself. |
| `@Enumerated(EnumType.STRING)` | Store `ATTENDEE`, not `0`. `ORDINAL` breaks if you reorder the enum. |
| `@ManyToOne(fetch = LAZY)` | Many events, one venue. Lazy = do not load venue until you touch it. |
| `@JoinColumn` | FK column name on the *many* side. |
| `@EntityGraph` | Tell JPA which associations to fetch with this query (avoid N+1). |

### Validation (request DTOs)

| Annotation | Job |
|---|---|
| `@Valid` | On `@RequestBody`: run Bean Validation **before** the controller method. Fail → **400**. |
| `@NotBlank` | String not null, not empty, not only spaces. |
| `@NotNull` | Not null (use on objects / `Integer`, not for “non-empty string”). |
| `@Email` | Looks like an email. **Empty string can pass** — pair with `@NotBlank`. |
| `@Min` / `@Size` | Numbers / length. Password: `@NotBlank` + `@Size(min = 8)`. |

**Say this:** `@Valid` does not run inside the method. DispatcherServlet binds JSON → Bean Validation → if fail, **400** and the controller method never runs. `@Email` without `@NotBlank`: empty string can pass.

### Web / HTTP

| Annotation | Job |
|---|---|
| `@SpringBootApplication` | Component scan + auto-config + start Tomcat. |
| `@RestController` | `@Controller` + `@ResponseBody` → return JSON, not a view name. |
| `@RequestMapping` / `@GetMapping` / `@PostMapping` | URL + HTTP method. |
| `@RequestBody` | JSON body → Java object (Jackson). |
| `@PathVariable` | `{id}` in the URL → method arg. |
| `@RequestParam` | Query string `?page=0`. |

**Say this:** `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Scan starts from this class’s package. `@RestController` writes the return value as the HTTP body (JSON). `@Controller` often returns a **view name**.

### Spring beans / layers

| Annotation | Job | Interview |
|---|---|---|
| `@Component` | Generic bean. Scan picks it up. | Parent idea. `@Service` / `@Repository` / `@RestController` are specializations (same mechanism + extra meaning). |
| `@Service` | Business logic bean. | Controllers depend on this, not on repositories. |
| `@Repository` | Persistence. Spring may wrap DB exceptions. | `JpaRepository` interfaces: Spring still builds a **proxy** bean. You never write the impl class. |
| `@Configuration` | This class has `@Bean` methods. | Processed so those methods run at startup. |
| `@Bean` | Method return value becomes a bean. | Use for library types (`PasswordEncoder`). Method name = default bean name. |
| `@Autowired` | Inject here. | Prefer constructor. Optional if there is only one constructor. Avoid on fields. |
| `@Qualifier` | Pick bean **by name** when several share a type. | Pair with `@Bean` method name. |
| `@Primary` | This bean wins when several share a type and no qualifier. | Default choice, not a replacement for clear design. |

**`@Configuration` + `@Bean` (PasswordEncoder):** we cannot put `@Service` on `BCryptPasswordEncoder` (not our class). So we write a method that `return new BCryptPasswordEncoder()`, mark `@Bean`, return type `PasswordEncoder`. AuthService asks for `PasswordEncoder` in its constructor — Spring passes that bean.

### Errors

| Annotation | Job | Interview |
|---|---|---|
| `@RestControllerAdvice` | Catch exceptions from **all** controllers in one class. | Keep HTTP mapping out of services. Service throws; advice chooses 404/409. |
| `@ExceptionHandler` | This exception type → this method. | Use the **same** `HttpStatus` in `ProblemDetail` and `ResponseEntity` (don’t mix 404 body with 409 header). |

### Tests

| Annotation | Job | Interview |
|---|---|---|
| `@WebMvcTest` | Web slice + MockMvc. | Mock the **service**. No JPA. Tests HTTP mapping, status, JSON. |
| `@DataJpaTest` | JPA slice + H2. | Real repository. Transaction **rolls back**. |
| `@SpringBootTest` | Full context. | Slow. Wiring / `contextLoads`, not every controller test. |
| `@MockitoBean` | Fake a bean in the test context. | Boot 4 name. Older docs: `@MockBean`. Same idea. |

### Security (Week 2)

| Piece | Job (general) |
|---|---|
| `PasswordEncoder` / BCrypt | Hash passwords. `@Bean`, not an annotation. `encode` register, `matches` login. Salt inside the hash. |
| `SecurityFilterChain` | Filters **before** the controller. URL matchers: `permitAll` / `hasRole` / `authenticated`. First match wins. |
| `JwtAuthenticationFilter` | Read Bearer → verify → fill `SecurityContextHolder`. Does not pick 403. |
| `SecurityContextHolder` | Per-request identity box. Not a field on the singleton. |
| `@WithMockUser` | Test-only. Fills that box. `roles =` is allowed; unnamed string is username. |
| `.with(csrf())` | MockMvc POST often needs this even if the app turned CSRF off. |
| `@Import` on `@WebMvcTest` | Pull advice (`GlobalExceptionHandler`) into the slice or throws become 500. |

### LC patterns so far

| # | Problem | Pattern | Time / Space |
|---|---|---|---|
| 217 | Contains Duplicate | HashSet | O(n) / O(n) |
| 121 | Buy/Sell Stock | Running min | O(n) / O(1) |
| 242 | Valid Anagram | HashMap counts | O(n) / O(n) |
| 155 | Min Stack | Two stacks (values + mins) | O(1) ops / O(n) |
| 167 | Two Sum II (sorted) | Two pointers (ends) | O(n) / O(1) |
| 125 | Valid Palindrome | Two pointers (ends, skip junk) | O(n) / O(1) |
| 238 | Product Except Self | Prefix | O(n) / O(1) extra |
| 26 | Remove Duplicates Sorted | Two pointers (write+read) | O(n) / O(1) |
| 11 | Container With Most Water | Two pointers (ends) | O(n) / O(1) |
| 15 | 3Sum | Sort then two pointers | O(n²) / O(1) extra |
| 53 | Maximum Subarray | Prefix / Kadane | O(n) / O(1) |
| 88 | Merge Sorted Array | Two pointers (tails) | O(m+n) / O(1) |
| 3 | Longest Substring No Repeat | Sliding window + HashSet | O(n) / O(n) |
| 49 | Group Anagrams | HashMap (sorted-letter key) | O(n k log k) / O(n) |
| 128 | Longest Consecutive Sequence | HashSet; start if `x-1` missing | O(n) / O(n) |
| 424 | Long Repeating Char Replace | Sliding window; length − maxFreq ≤ k | O(n) / O(1) |
| — | Two Sum, Valid Parentheses | (earlier — still passed) | — |

---

<!-- Template for NEXT day — paste mentor block; keep Week/Day title -->

## Week X Day Y — &lt;Subject&gt;

**Date:** YYYY-MM-DD  
**Goal:**

### Quick recall

| Piece | One-liner |
|---|---|

### What I built

-

### Under the hood (cheat)

-

### Interview answers (model)

1.

### Weak spots

-

### Part 2 / Part 3 (if any)

-

### 60-sec story

>

---
