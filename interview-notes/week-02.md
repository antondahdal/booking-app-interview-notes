# Interview notes — Week 2 

Auth + booking v1: register, JWT (Day 2), roles.

Each day's **Quick recall** is **general** (any Spring app) → **here** → **trap**. You can read a day without the chat.

---
## Week 2 Day 1 — User + register + BCrypt

**Date:** 2026-08-24  
**Goal:** Public register. Store a BCrypt hash. Never return the password or the hash.

### Quick recall (Spring — Part 1)

**Unique + 409**  
- **General:** A DB **unique** index stops two identical keys, including races. Unique alone with no handler often becomes **500**. **409** = the API: request is valid, **current state** says no (email taken, sold out, already booked — same meaning, different resources).  
- **Here:** Unique on email + `DuplicateException` → 409.  
- **Trap:** 409 ≠ “duplicate email only.”

**Role**  
- **General:** The client must not pick their own privilege. Server sets the default role. Permission is checked **per request**, not once at signup.  
- **Here:** Service sets `ATTENDEE`. Never on the JSON.  
- **Trap:** “I blocked create-event at register” — register is not create-event.

**BCrypt / `encode` / `matches` / salt**  
- **General:** Never store plaintext. Fast hashes are guessable. BCrypt is **slow on purpose**. **Salt** = random extra, stored **inside** the hash. Same password → different hashes. **No decode.** Register: `encode(raw)`. Login: `matches(raw, storedHash)`.  
- **Here:** `BCryptPasswordEncoder` bean; hash never in the 201 body.  
- **Trap:** Login must not `encode` again and compare strings. Unknown email and bad password → same **401**.

**`@Bean` vs `@Service`**  
- **General:** `@Service` is for **your** class. Library types (`BCryptPasswordEncoder`) cannot wear it. `@Configuration` + `@Bean` = method **return value** is the bean. Inject the **interface**.  
- **Here:** `PasswordEncoderConfig` → `PasswordEncoder` into `AuthServiceImpl`.

**HTTP register**  
- **General:** Create resource **201**. Bad JSON **400**. State clash **409**.  
- **Here:** Register 201; email taken 409.

### What I built

- `User` (`users` table), `Role` (`STRING`), `existsByEmail`
- `POST /api/auth/register` → 201, `DuplicateException` → 409
- `PasswordEncoder` `@Bean` + constructor injection into `AuthServiceImpl`

### Part 2 — LC 155 Min Stack (Medium) — passed

**What the question needs:** a stack with four ops, **each O(1)**: `push(val)`, `pop()`, `top()` (look, do not remove), `getMin()` (smallest value **still** in the stack). Extra space O(n) is allowed. Scanning on `getMin` is O(n) — not the interview answer.

**Why Medium:** a normal stack is Easy. Medium is `getMin` in O(1) **after** `pop` removes the current min. One `int min` cannot go backwards.

**Pattern:** two stacks — values + mins. Keep last unmatched / last min so `pop` can **undo**.

**Why not**

- **HashMap** — not “does this key exist?” Order of push/pop is the whole problem.
- **#121 running min** — min only moves forward in a scan you never reverse. Here `pop` undoes.

| Method | Values stack | Min stack |
|---|---|---|
| `push(val)` | always push | push if empty **or** `val <=` current min (`<=` so two equal mins both stay) |
| `pop()` | always pop | pop only if popped value **equals** current min |
| `top()` | **`peek` only** | do not touch |
| `getMin()` | — | **`peek`** |

Walk: `push(3)` mins `[3]` → `push(5)` mins still `[3]` → `push(2)` mins `[3,2]` → `pop` uncovers `3`.

**Bugs I hit**

1. `top()` copied `pop()` — one `top()` at the end of the LC example still passed. Tests: `top()` twice; `top()` then `getMin()` must still be the same min.
2. Duplicate mins: `peek() > val` (`<` only) → second `1` never on min stack → first `pop` empties it → `EmptyStackException`.

**Cousin:** min of a **moving window** on an array → sliding window / deque. If `getMin` may be O(n) → scan.

**Interview sentence:** “I keep a second stack of mins; I push when `val <=` current min, and I pop that min stack when I pop the same value — all O(1).”

### Spring weave-in — singleton service, no “current user” field

**Question:** `AuthService` is a singleton. Why must you **not** save “the user who is registering” in a **field** on that service?

**Idea (plain):** one shared `AuthService` for the whole app. I call register → you store me on the service. Someone else calls register → **same object** → they see me, or they overwrite me and my request sees them.

**Technical:**

- Spring beans default to **singleton**: **one instance per application context**, reused for every HTTP request.
- A **field** on that class is **shared mutable state**. Two requests = two threads (or overlapping calls) writing/reading the **same** field → **race**: mixed users, lost updates.
- Request data belongs in **method parameters** and local variables (the `RegisterRequestDto` lives only for that call). Those are not shared.
- This is **thread-safety**, not “wrong role.” `PasswordEncoder` as a singleton is fine: `encode` / `matches` have **no per-user fields**.

**Model answer:**

> Spring services are singletons — one instance, many requests. I don’t store the current user on the service. Request data stays in method parameters. A field would be shared across threads and mix users up.

### Part 3 — OOP + design

**OOP — interface vs abstract class** (picture was right; names were mixed)

- **Abstraction** (the idea) = hide how, show what. Two Java tools:
- **Abstract class** = is-a + shared **fields/code**. B and C `extends` A so they don’t copy. One parent.
- **Interface** = **contract** (methods). `implements`, many allowed. No instance fields to inherit.
- If B and C do the **same** work, put the **body** on the abstract class. Empty methods = they do it **differently**, or you only need a contract → interface.
- `AuthService` is an **interface** because of **DIP / mock** (controller + `@WebMvcTest`), not because two subclasses share User fields.

---

**Design drill — who may create an event?**

This is **not** “draw Kafka.” Mid-level **design materials** (Friday boards + weekday drills) are mostly: **who can do what**, **where the truth lives**, **what the API returns when it goes wrong**. Same family as Friday’s oversell (critical request + 409 + lock the row, not the table).

**The prompt:** `POST /api/events` already exists (create event DTO: title, seats, venue, …). Register is public. Later, create-event must be **organizer-only**. The controller does **not** take user/role on that DTO — and it must not.

**Wrong instinct:** “At register I check role; if not organizer I block create-event.”  
Register **always** sets `ATTENDEE`. You are not calling create-event yet. Permission is not a one-time gate at signup.

**Right design**

1. **Public vs protected.** Register = anyone. Create event = only `ORGANIZER` / `ADMIN`. That split is an API design decision (what is anonymous, what needs a logged-in actor).
2. **Trust boundary.** Role is **not** in the JSON. The client would send `ADMIN` and self-promote (same bug as role on register). Identity rides **outside** the body:

```
POST /api/events
Authorization: Bearer <JWT>     ← who you are (after login)
Body: { title, seats, venueId } ← the event only
```

3. **When to check.** On **that** POST. A filter (Spring Security, Day 2) reads the token **before** the controller, puts principal + role on `SecurityContext`. Then `@PreAuthorize` / matcher: organizer only. Service may still read the principal for `createdBy`; it does not take `role` from the DTO.
4. **Failure as part of the contract** (interviewers grade this):

| Status | Meaning | This app |
|---|---|---|
| **401 Unauthorized** | No / bad identity | Missing token |
| **403 Forbidden** | Identity OK, not allowed | `ATTENDEE` hits create-event |
| **409 Conflict** | Valid request, state clash | Duplicate email, oversell |
| **201** | Created | Organizer create succeeds |

**Authn vs authz (say these words)**  
**Authentication** = who are you? (login, JWT). **Authorization** = are you allowed to do **this**? (role on this URL). 401 is authn failure; 403 is authz failure.

**Why this belongs in design materials (not “only Spring”)**

Friday-style systems all have the same questions: **which requests are public**, **where identity lives**, **what you refuse and with which status**. URL Shortener: who may create vs who may redirect. Notifications: who may send, workers vs user. Rate limiter: 429 is a designed refusal. Cart / booking / payments: who may mutate, and you never trust the client for “I am admin” or “charge $0.”

JWT / `@PreAuthorize` is **how** you implement it (Day 2). The **design** is: DTO = resource; token = actor; check per request; 401 vs 403 vs 409.

**Interview sentences**

> Authorization is per request, not at register.  
> The create-event body has no user. Identity is the token. Attendee → 403, not 401.

### 60-sec story

> Public register creates a User. Password is BCrypt-hashed with a per-call salt. Role is ATTENDEE, never from the client. Duplicate email is 409. Encoder is a `@Bean` injected by constructor. The service is a singleton so I never keep the current user on a field. Min Stack: second stack of mins so `getMin` is O(1) after pop. AuthService is an interface (contract), not an abstract class. Create-event is authorized per request: token = who, DTO = event; attendee gets 403. That split (public vs protected, trust boundary, 401 vs 403) is the same kind of design as oversell / 409.

---

## Week 2 Day 2 — Login + JWT + SecurityFilterChain

**Date:** 2026-08-25  
**Goal:** Login with `matches`, issue a signed JWT, keep `/api/auth/**` public.

### Quick recall

**JWT parts vs claims**  
- **General:** Three Base64 pieces: **header.payload.signature**. **Claims** (`sub`, `exp`, role) live **inside** the payload. Payload is **readable**, not encrypted. Safety = **signature** (stops edits, not copying).  
- **Here:** Login `generateToken` HS256. No password in the token.  
- **Trap:** “JWT is subject, time, and signature” mixes parts with claims.

**`matches` vs `encode`**  
- **General:** `encode` = store a new hash (register). `matches` = check login. No decode.  
- **Here:** `AuthServiceImpl.login` uses `matches` only.

**401 vs 403**  
- **General:** **Authentication** = who are you? Fail → **401** (missing/bad/expired). **Authorization** = may you do this? Fail → **403** (we know you, not allowed).  
- **Here:** Bad login / no token → 401. Attendee `POST /api/events` → 403.  
- **Trap:** HTTP says “Unauthorized” for 401. Spoken: 401 = we don’t know you.

**`permitAll` + `SecurityFilterChain`**  
- **General:** Filters run **before** the controller. `permitAll` = ignore the identity box (public URL), **not** a role. Login must be public or nobody can get a token. First matcher wins; `anyRequest()` last.  
- **Here:** `/api/auth/**` public. Config writes **URL rules**, it does not store this user’s role.  
- **Trap:** `/error` not public can turn a real 400 into an empty 403.

**STATELESS + CSRF**  
- **General:** STATELESS = no server session; JWT **is** the session. CSRF targets **cookie** auto-send. Bearer in a header is not that — copied JWT still works until `exp`. Password change does not kill today’s tokens (no denylist).  
- **Here:** CSRF off, STATELESS on.  
- **Trap:** CSRF does not block Postman with a pasted `Authorization`.

### What I built

- `findByEmail`, `InvalidCredentialsException` → 401
- `JwtService` / `JwtServiceImpl` (`generateToken`, HS256)
- `POST /api/auth/login` → 200 + token
- `SecurityConfig`: `/api/auth/**` public, rest `authenticated()`, CSRF off, STATELESS
- **Not yet:** JWT filter that reads `Authorization` (create-event still 401 even with a token)

### Interview answers (model)

1. JWT = header.payload.signature. Payload is readable. Signature (HMAC + secret) stops tampering.
2. Authn = who (401). Authz = what you may do (403). Create-event: no token → 401; attendee → 403.
3. `SecurityFilterChain` = security filters in front of every request. Controller never runs if the filter rejects.
4. STATELESS = no server session. CSRF off because Bearer is not auto-sent like a cookie.
5. Role is not on the create-event JSON (client would send ADMIN). Role is in the signed JWT (copied from the User row at login). Config later **checks** that role on the URL.

### Weak spots (review tonight)

- JWT **parts** vs **claims** (`sub`, `exp`)
- `SecurityFilterChain` (before controller)
- CSRF vs STATELESS — CSRF does **not** block a copied Bearer token
- `permitAll` ≠ role. Role source = token/DB, not `SecurityConfig` storing the user.
- Expired JWT → **401**, not 403
- Stolen JWT still works after password change (no denylist today)
- **400** = bad input. **401** = no identity. Do not mix them.
- **409** = conflict with **current state** (valid request, cannot apply it now). Book: seats gone / already booked. Register: email taken. Same code, different resources — not “409 means duplicate email.”

### Part 2 — LC 232 Queue using Stacks (Easy) — passed

**What the question needs:** a FIFO **queue** built only from **stacks** (LIFO). Four ops: `push(x)` enqueue at the back, `pop()` dequeue the front and return it, `peek()` look at the front (do not remove), `empty()`. You may not use `Queue`. Extra space O(n) is allowed.

**Why it is not “just use a stack”:** one stack pops the **newest**. A queue must pop the **oldest**. You reverse order by pouring into a second stack.

**Pattern:** two stacks — **in** (original) + **out** (upside-down). Same family as Min Stack (#155), different job. Yesterday the second stack remembered **mins**. Today it remembers **front of the line**.

**Why not**

- **HashMap** — not “does this key exist?” Order of enqueue/dequeue is the whole problem.
- **Min Stack** — two stacks, but that second one is mins. You would still fail FIFO.
- **Pour back every pop** — that makes every `pop` O(n) and throws away amortized O(1).

| Method | In-stack (original) | Out-stack (upside-down) |
|---|---|---|
| `push(x)` | always push — **O(1)** | do not touch |
| `pop()` / `peek()` | pour **all** into out only if out is **empty** | then `pop` / `peek` here. Do **not** pour back. |
| `empty()` | empty **and** out empty | (items can live on either stack) |

**Amortized O(1):** one `pop` can pour n items (O(n) that call). Each value moves **in → out at most once**, so n ops cost O(n) total → average O(1) per call. `push` is always O(1) in this design. (The other design — reverse on every `push` — makes `push` O(n). We did not use that.)

Walk: `push(1) push(2)` in=`[1,2]`. `pop()` pours → out=`[2,1]` then pop `1`; leave `2` on out. `push(3)` stays on in. Next `pop` is still `2` (out). When out is empty, pour `3,4,5,6` in one `while` loop — Java `Stack` has **no** built-in pour.

**`peek` vs `pop`:** same pour rule. `peek` = look (`peek`). `pop` = remove (`pop`). Same trap as Min Stack `top`.

**Interview sentence:** “I push on an in-stack and only reverse onto an out-stack when out is empty, so each element moves twice and ops are amortized O(1).”

---

### Cousin — LC 225 Implement Stack using Queues

**Same idea, flipped.** #232: stacks (LIFO) → fake a queue (FIFO). **#225:** queues (FIFO) → fake a stack (LIFO).

A **queue** gives you the **oldest** first. A **stack** must give you the **newest** first. So you shuffle with **two queues** (or one queue + rotate) until the newest sits at the front.

Typical two-queue sketch (know the story, code later if asked):

- `push(x)`: enqueue `x` on the spare queue, then dump the main queue into it, then swap names — `x` is now at the front. `pop` / `top` are O(1) from that front. `push` is O(n).
- Other version: cheap `push`, expensive `pop` (rotate until last item is at the front). Same trade-off as #232’s two designs.

**If they allow a real `Queue` / `Deque`:** #232 disappears (`ArrayDeque` as queue). **If they allow a real `Stack`:** #225 disappears. The point of both problems is the constraint.

**Interview sentence for 225:** “Queue is FIFO, stack is LIFO — I use two queues and move everything so the newest element sits at the front.”

### Part 3 — OOP + design

**Cadence (so Friday does not feel like “more 401”):** weekday = one OOP idea + a **slice** of API contract (status codes, public vs protected, token vs body). Friday = **full HLD**: boxes (client → API → DB), booking sequence, lock on the event row, what breaks at 10× traffic. Same product, bigger picture. Today’s extra API table **assembled** Day 1 + Day 2 gate + Friday 409 — not a new topic.

**OOP — composition (has-a), not inheritance (is-a)**

- `MyQueue` **has** two `Stack`s. It does **not** `extends Stack` (a queue is not a stack — FIFO vs LIFO, not “different parameters”). Extending `Stack` leaks LIFO methods (`search`, extra pops) and callers can skip your FIFO rules.
- `AuthServiceImpl` **has** `JwtService` + `PasswordEncoder`. It **calls** them; it does not extend them.
- `JwtServiceImpl` **implements** `JwtService` — that is yesterday’s **contract**, not composition.

**OOP extra — chain of responsibility:** `SecurityFilterChain` is a **list** of filters (CSRF off → STATELESS → `authorizeHttpRequests` → later JWT filter). Each does one job and passes on. Do not put `matches` + `signWith` + URL rules in the controller.

**Model:** I compose. MyQueue has two stacks. AuthServiceImpl has JwtService. I do not extend Stack. Security is a chain, not one god-class.

---

**Design drill — stolen or expired JWT**

The token **is** the session (`STATELESS`). Anyone with the string **is you** until `exp`. Signature stops **edits**, not **copying**.

| Situation | What happens |
|---|---|
| Copied JWT in Postman / curl | They can call every URL **you** may call (attendee = attendee URLs). **CSRF does not block this.** CSRF targets **cookie auto-send**. We turned CSRF **off**. Bearer is not a cookie — they paste `Authorization` themselves. |
| Token expired / bad signature | **401** — no valid identity. Not 403 (403 = we know who, not allowed). |
| You change password now | Stolen token **still works** until `exp`. We do not store sessions and have no denylist. That is the STATELESS trade-off. Short `expiration-ms` limits the window. |

Password is not in the JWT, so they cannot mint a **new** token without `matches`. Refresh tokens / denylist = later.

---

**Extended design — Event Booking API (auth + create + book)**

Trust: JSON body = **the resource**. `Authorization: Bearer` = **who**. Never `role` / `userId` on create-event or book JSON.

| Method | Path | Who | Success | Body |
|---|---|---|---|---|
| `POST` | `/api/auth/register` | public | **201** | email, password → user (no hash) |
| `POST` | `/api/auth/login` | public | **200** | email, password → `{ token }` |
| `GET` | `/api/events` | **public** (this product) | **200** | — |
| `POST` | `/api/events` | logged-in **organizer** | **201** | title, seats, venue… **only** |
| `POST` | `/api/events/{id}/bookings` | logged-in **attendee** | **201** | `{ seats }` — not tomorrow’s code |

**200 vs 201:** `POST` is the verb. Register **creates** a `User` → 201. Login does not create a user → 200.

**`GET /api/events` public:** Eventbrite-style catalog. A guest can **see** events, then sign up / log in, then **book**. Invite-only (admin adds attendees, list is private) is a **different** product — do not mix it in.

Three actions (do not collapse them): **see** = GET · **create account** = register · **book** = `POST .../bookings`.

**Truth:** hash + role → DB. Ticket → JWT. Event / booking rows → DB. `createdBy` / booker from the token, not from JSON. STATELESS = any instance can check the signature.

**Gap until Wed:** no JWT filter yet → `POST /api/events` is 401 even with a valid token.

**Bookings status (trap: 401 ≠ bad input, 409 ≠ duplicate email)**

| Status | Book means |
|---|---|
| **400** | Bad JSON / validation |
| **401** | No / bad / expired token |
| **403** | Identity ok, this user may not book (e.g. blocked). An `ATTENDEE` **is** allowed to book. |
| **409** | State clash: last seat gone (oversell + row lock), or already booked |
| **201** | Booking created |

**Interview sentences**

> Composition: has-a, not extends. Security is a filter chain.  
> A copied JWT is me until exp. CSRF does not stop Bearer. Expired → 401. Password change does not kill today’s tokens.  
> Login 200, register 201. GET events is public so guests can browse. Book: 401 if not logged in, 403 if logged in but not allowed, 409 if seats are gone. 400 is bad input.

### 60-sec story

> Login uses `matches`, never encode. Same 401 for unknown email and bad password. JWT is header.payload.signature; payload is readable so no secrets; HMAC signature stops edits. `/api/auth/**` is permitAll so you can get a token. GET events is public so guests can browse; book needs a token. CSRF is a cookie problem; a copied Bearer is not CSRF. Role is not in the event DTO. Queue-from-stacks: pour to out only when empty — amortized O(1). Cousin #225 is two queues to fake a stack. MyQueue has stacks; it is not a Stack. Stolen JWT works until exp. Weekday design = API contract slice. Friday = boxes, booking sequence, lock, 10× traffic.

---

**After this session:** switch Windows user profile.

**Next (Wed):** JWT filter reads `Authorization` → `SecurityContext`. Then create-event can be 201 / 403 instead of 401 for everyone.

---

## Week 2 Day 3 — JWT filter + SecurityContext + roles

**Date:** 2026-08-26  
**Goal:** A Bearer token becomes “who you are” on this request. Guests can **see** events. Only an organizer can **create** one.

### Quick recall

**Why a filter (any servlet app)**  
- **General:** Security runs **before** the dispatcher/controller. If identity is only checked inside the controller, URL rules already rejected (empty box → 401).  
- **Here:** `JwtAuthenticationFilter` parses Bearer, fills the box, always `doFilter`. It does **not** pick 403.  
- **Trap:** “Thin controller” is SRP, not why create-event failed without the filter.

**The box (`SecurityContextHolder`)**  
- **General:** A singleton filter cannot store “current user” in a **field** (mixed requests). Identity lives in a **per-request** box Spring clears at the end.  
- **Here:** `setAuthentication` with email + `ROLE_ATTENDEE`. `getName()` later is that email.  
- **Trap:** A local variable in `doFilterInternal` dies when the method continues; the box must outlive that.

**`ROLE_` prefix**  
- **General:** `hasRole("ORGANIZER")` looks for authority `ROLE_ORGANIZER`. JWT claim can stay `ORGANIZER`; the filter adds the prefix. Skip it → organizer still 403. Parse is fine.  
- **Here:** Claim `ORGANIZER` → authority `ROLE_ORGANIZER`.

**GET public / POST organizer**  
- **General:** Same path, different rules per **HTTP method**. First match wins. Public GET is a product choice (catalog).  
- **Here:** `GET /api/events/**` permitAll (guests). `POST /api/events` organizer (create, **not** book).  
- **Trap:** POST events ≠ book. `anyRequest` last is a fallback, not a bundle.

---

### Connect the dots (Day 1 → 2 → 3)

Think of security as **three questions**, one per day. An interviewer is asking the same three.

| Day | Question | What we built |
|---|---|---|
| **Mon** | How do we store a password? | Register + BCrypt. Role `ATTENDEE` from the **server**, never from JSON. |
| **Tue** | How do we prove who you are **once**? | Login + `matches` + signed JWT. `/api/auth/**` public. |
| **Wed** | How do we remember who you are on the **next** request? | Filter reads `Authorization`, fills a per-thread box. URL rules: GET public, POST organizer. |

Without Wednesday, Tuesday’s token is a useless string. Spring never looks at it. Create-event stays **401** even with a perfect JWT.

```
Monday:  password  →  hash in DB
Tuesday: password  →  JWT string (ticket)
Wednesday: ticket on every request  →  “this is Anton, ATTENDEE”
```

The ticket is **not** a session in our server. STATELESS = we do not store “Anton is logged in” in memory. We **re-check the signature** on every call. Any app instance can do that. That is why microservices like JWT later.

---

### One HTTP request (the road)

Client:

```
POST /api/events
Authorization: Bearer eyJhbGciOi...     ← ticket (who)
Body: { title, venueId, seats }        ← event only (what)
```

What Spring does, in order:

```
1. JWT filter
      no Bearer?  →  leave the box empty, continue (guest)
      Bearer?     →  parseToken (signature + exp)
                      fail  →  box stays empty
                      ok    →  put email + ROLE_ATTENDEE in the box

2. URL rules (SecurityConfig) look in that same box
      GET  /api/events     →  anyone (empty box is OK)
      POST /api/events     →  need ROLE_ORGANIZER or ROLE_ADMIN
      /api/auth/**         →  anyone (how you get a ticket)
      /error               →  anyone (or a 400 becomes a fake 403)
      everything else      →  box must not be empty

3. Only then the controller runs
```

**Who does not do security**

- `EventController` does not read the header.
- The JSON body does not contain `role` or `userId`.
- `JwtService` does not decide 401 vs 403. It only **reads** the ticket.

Filter = identity. Config = permission. Controller = create the event.

---

### How `SecurityConfig` is connected to `JwtService` (who calls whom)

They do **not** call each other on a request. `SecurityConfig` never runs `parseToken`. It only **wires** the objects at **startup**. The **filter** is who calls `JwtService` later.

**Two uses of the same `JwtService` bean** (one object, two callers):

```
AuthServiceImpl          ── generateToken ──►  JwtService   (login)
JwtAuthenticationFilter  ── parseToken ────►  JwtService   (every later request)
                                    ▲
                                    │ Spring injects JwtServiceImpl
                            SecurityConfig constructor holds it
                            and does: new JwtAuthenticationFilter(jwtService)
```

`SecurityConfig` constructor takes `JwtService`. Spring sees the **interface** and injects **`JwtServiceImpl`** (`@Service`). Then `filterChain` passes that same bean into `new JwtAuthenticationFilter(jwtService)`.

Plain: **Config = recipe. Filter = work on each HTTP call. JwtService = sign / verify.** Config does not sit on the request path.

`JwtService` is an interface (DIP). The impl holds the HMAC `key` from `app.jwt.secret`. Generate and parse must use **that same key** or every token looks fake.

---

### When the app starts (before any HTTP)

1. `@SpringBootApplication` scans. Finds `@Service JwtServiceImpl`, `@Configuration SecurityConfig`, `@Service AuthServiceImpl`, controllers.
2. Spring creates **`JwtServiceImpl`**. Constructor reads `app.jwt.secret` + `expiration-ms` and builds the `SecretKey`. This is the only place the secret becomes a key.
3. Spring creates **`SecurityConfig`**. It needs `JwtService` → gets the impl from step 2. **`AuthServiceImpl`** also gets that same `JwtService` (plus encoder + user repo).
4. Spring calls the `@Bean` method **`filterChain(HttpSecurity http)`**. This is **not** a user request. It **builds** one `SecurityFilterChain`: CSRF off, STATELESS, URL matchers, then `addFilterBefore(new JwtAuthenticationFilter(jwtService), UsernamePasswordAuthenticationFilter.class)`.
5. `http.build()` returns the chain. Spring Security’s **`FilterChainProxy`** (one servlet filter in front of the app) will use it. Tomcat opens 8080.

`JwtAuthenticationFilter` is **not** `@Component`. Config **constructs** it so it does not run twice (servlet container + security chain).

---

### Layers when a request arrives (who is called by whom)

Security is **in front of** the controller. Outside-in:

```
Client
  → Tomcat
  → FilterChainProxy          (Spring Security’s outer filter)
       ├─ CSRF                (disabled — Bearer is not a cookie)
       ├─ JwtAuthenticationFilter          ← our class
       │     reads Authorization
       │     calls jwtService.parseToken   ← JwtServiceImpl
       │     setAuthentication on SecurityContextHolder
       │     then doFilter (must continue)
       ├─ UsernamePasswordAuthenticationFilter
       │     form login — disabled; we only use it as the “insert before” hook
       ├─ AuthorizationFilter
       │     THIS is the requestMatchers in SecurityConfig
       │     reads the same SecurityContextHolder box
       │     permitAll / hasAnyRole / authenticated
       │     fail → 401 or 403, controller never runs
  → DispatcherServlet         (only if URL rules passed)
  → AuthController / EventController / VenueController
  → AuthService / EventService
```

**Login** uses the same chain, but `/api/auth/**` is `permitAll`, so an empty box is OK. Then: `AuthController` → `AuthServiceImpl.login` → `matches` → `jwtService.generateToken`. No `parseToken` on that call.

**Create-event** needs the box filled **before** `AuthorizationFilter`. That is why JWT is `addFilterBefore`. If you parsed the token in the controller, the URL rule would already have returned 401.

---

### The pieces (enough to explain, not one line)

**`JwtService.generateToken`**  
Called from **login** (`AuthServiceImpl`), not from the filter. Writes id (`sub`), email, role, `exp`, then signs with the HMAC key. Returns one string — the ticket the client keeps.

**`JwtService.parseToken`**  
Called from **`JwtAuthenticationFilter`**, not from `SecurityConfig`. `verifyWith(key)` checks signature and expiry. The payload is Base64 and **readable** without the secret; skip verify and anyone can send `"role":"ADMIN"`. Maps claims into `JwtPrincipal`. If it throws, the filter catches and leaves the box empty.

**`JwtPrincipal`**  
Our type: id, email, `Role`. Not a REST DTO (client never sees it). Not a Spring class. The filter uses it to build `ROLE_…` authorities.

**`JwtAuthenticationFilter`**  
Servlet filter (`OncePerRequestFilter` = once per request). Header → parse → fill the box → always `doFilter`. No Bearer (login, GET catalog) is normal: skip parse, continue. It does **not** pick 401 vs 403 — that is `AuthorizationFilter`.

**`SecurityConfig`**  
Startup-only recipe. Constructor **receives** `JwtService`. `filterChain` **gives** it to the JWT filter and **writes URL rules**. Those rules run later inside `AuthorizationFilter`. You never “call SecurityConfig” from a controller.

**`SecurityContextHolder`**  
Spring’s per-thread box. `setAuthentication` means “this request is this user.” `AuthorizationFilter` / `hasRole` read the same box. Cleared when the request ends.

**`UsernamePasswordAuthenticationToken`**  
The object **inside** the box: email + `ROLE_ATTENDEE` (or ORGANIZER). Password is `null` because JWT already proved identity. This is **not** a filter.

**`UsernamePasswordAuthenticationFilter`**  
Old form-login filter in the chain. Form login is off. We use this **class** as the slot: put JWT **before** it. Different from Token.

**`hasAnyRole("ORGANIZER", "ADMIN")`**  
URL rule in `SecurityConfig`, executed by `AuthorizationFilter`. Spring looks for `ROLE_ORGANIZER` / `ROLE_ADMIN` — that is why the filter adds `"ROLE_"`. Attendee in the box → **403**. Empty box → no identity.

**`permitAll` / `authenticated()`**  
`permitAll` = ignore the box (GET events, `/api/auth/**`, `/error`). `authenticated()` = box must have a user (POST venue). First matching matcher wins; `anyRequest()` stays **last**.

---

### Where “current user” lives (interview favorite)

The filter is a **singleton**: one object, every request.

| Put the user here | What happens |
|---|---|
| Field on the filter `currentUser` | Shared. Two requests overwrite each other. Mixed users. |
| `SecurityContextHolder` | **ThreadLocal** = a box tied to **this request’s thread**. Request A cannot see Request B. Spring **clears** it when the request ends. |

The user does **not** die when `setAuthentication` returns. The controller still needs them. They live until the **request** ends.

Same idea as Monday: do not store “the user who is registering” on `AuthService`.

---

### Status codes (say these in the interview)

Identity vs permission vs input vs state — four different meanings.

| Status | Meaning | Today |
|---|---|---|
| **200** | Read / login OK | `GET /api/events`, `POST /api/auth/login` |
| **201** | Created | Register; organizer `POST /api/events`; attendee `POST /api/venues` |
| **400** | Bad JSON / `@Valid` | Body missing fields |
| **401** | No identity | No header, bad/expired JWT, no filter (header ignored) |
| **403** | We know who; not allowed | Attendee hits `POST /api/events` |
| **409** | **Conflict:** request is valid and allowed, but it **clashes with current server state** (cannot create/update this way right now). Not 400 (bad JSON) and not 401/403 (who / permission). | **Examples:** email already taken (register). **Later:** last seat gone, already booked (oversell). Trap: 409 is not “duplicate email only.” |

Trap we hit in the terminal: PowerShell broke JSON → real answer was **400** → Spring forwarded to `/error` → `/error` was locked → **empty 403**. `permitAll` on register does **not** cover `/error`. Matcher: `.requestMatchers("/error").permitAll()`.

Another trap: `hasRole` with an **empty** box sometimes shows **403** (anonymous is a weird “user”). Interview answer still: missing identity **should** be 401; wrong role is 403.

---

### What we proved in curl

| Call | Token | Result | What it proved |
|---|---|---|---|
| `GET /api/events` | none | **200** | Catalog is public |
| Login with `@login.json` | — | **200** + JWT | `matches` + `generateToken` |
| `POST /api/venues` | real attendee JWT (`Length` ~204) | **201** | Filter filled the box (`authenticated()`) |
| `POST /api/events` | same attendee JWT | **403** | Box has `ROLE_ATTENDEE`; URL wants organizer |

Placeholder `$TOKEN = "eyJ-paste-the-real-one"` is not a JWT. Parse fails → empty box → 403 that **looks** like “forbidden” but is really “not logged in.”

Role is copied into the JWT **at login**. `UPDATE USERS SET ROLE = 'ORGANIZER'` does not change an old token. Login again.

---

### 60-sec story (memorize this)

> Register stores a BCrypt hash and sets ATTENDEE. Login uses matches and returns a signed JWT. The payload is readable so we never put secrets in it; the signature stops edits. A later request sends Authorization Bearer. A filter verifies the token and puts the user on SecurityContextHolder — a per-thread box, not a field on the singleton. GET events is permitAll so guests can browse. POST events needs ROLE_ORGANIZER. Attendee gets 403, missing token 401. The event JSON has no role. Bad input is 400; if /error is not public it can look like 403.

---

### Interview answers (model)

1. **Current user:** Filter is a singleton. Identity lives on `SecurityContextHolder` (ThreadLocal = this request’s thread). A field `currentUser` is one variable for every request → mixed users.
2. **Why a filter:** Security runs before the controller. URL rules read the box first. Parse only in the controller → empty box → 401/403, `parseToken` never runs. Thin controller is extra, not the main reason.
3. **401 vs 403:** 401 = no valid identity (missing / bad / expired token). 403 = we know who; not allowed (attendee + create-event). Expired is 401, not 403.
4. **`ROLE_`:** `hasRole("ORGANIZER")` looks for authority `ROLE_ORGANIZER`. JWT claim stays `ORGANIZER`. Prefix is added when building authorities. Skip it → organizer still 403. Not a parse issue.
5. **GET vs POST:** Same path, two matchers, split by `HttpMethod`. GET `permitAll` (guests browse). POST `hasAnyRole` (create event, not book). First match wins → `anyRequest()` last.

---

### Weak spots (review before Part 2 / tonight)

**1. Where the user lives — you said “local thread”**

- **General:** A singleton bean is one object for the whole process. Request data cannot live on its fields if two requests can overlap (two threads). Per-request data goes in ThreadLocal, method args, or the request object. Spring’s name for the identity box is **`SecurityContextHolder`**. It uses ThreadLocal internally. After the request, Spring clears it.
- **Here:** `JwtAuthenticationFilter` calls `SecurityContextHolder.getContext().setAuthentication(...)`. That is “current user” for this HTTP call.
- **Trap:** “Local thread” is the right picture; say the class name in the interview. A local **variable** inside `doFilterInternal` dies when the method continues to `doFilter` — we need the user to **stay** until the controller finishes, which ThreadLocal does.

**2. Filter vs controller — you said thin controller**

- **General:** In a servlet app, **filters run before the dispatcher / controller**. Authorization that looks at “is this user authenticated?” must see identity **already set**. If you only check the token inside the controller, the security layer has already decided.
- **Here:** `AuthorizationFilter` uses `SecurityConfig` matchers. `POST /api/events` needs `ROLE_ORGANIZER`. Empty box → reject. Controller never runs → `parseToken` never runs. A valid `Authorization` header is ignored (yesterday’s 401).
- **Trap:** Thin controller is true (SRP) but not why create-event would fail. The reason is **order**.

**3. 401 vs 403 — you said 401 = bad JWT, 403 = not allowed**

- **General:** **401 Unauthorized** = **authentication** failed: we do not have a valid identity. **403 Forbidden** = **authorization** failed: identity is known, this action is not allowed. 401 is not “malformed JSON” (that is 400).
- **Here:** no header / bad token / **expired** token → 401. Attendee `POST /api/events` → 403.
- **Trap:** 401 is not only “bad JWT.” Missing header and expiry are 401 too. Expired is not 403.

**4. `ROLE_` prefix — you said it would mess up parsing**

- **General:** Spring distinguishes **roles** vs **authorities**. `hasRole("X")` / `hasAnyRole("X")` look for a granted authority named **`ROLE_X`**. `hasAuthority("X")` looks for exactly `X`. This is Spring Security’s convention, not JWT’s.
- **Here:** JWT claim is `"role":"ORGANIZER"`. After `parseToken`, the filter adds `"ROLE_"` when creating `SimpleGrantedAuthority`. Parse does not use `ROLE_`.
- **Trap:** Skipping the prefix does not break JJWT. The organizer is still “logged in” but `hasAnyRole("ORGANIZER")` does not match → **403**.

**5. GET public / POST organizer — you mixed create-event with book; `anyRequest` “wraps”**

- **General:** The same URL path can have different rules per **HTTP method**. Matchers are checked **in order**; **first match wins**. A catch-all (`anyRequest`) must be last or it shadows the specific rules. Public GET is a product choice (catalog); mutating POST is a different contract.
- **Here:** `GET /api/events/**` `permitAll` — **guests** (no token), not only attendees. `POST /api/events` `hasAnyRole("ORGANIZER", "ADMIN")` — **create an event**, not book. Book will be `POST /api/events/{id}/bookings` (attendee). `anyRequest().authenticated()` last = default for unlisted URLs (e.g. POST venue).
- **Trap:** POST events ≠ book. `anyRequest` last is not “it wraps like a bundle” — it is the **fallback** because first match wins.

---

**After lunch:** Part 2 LC 739 **done** (other chat). Part 3 below.

---

### Part 2 — LC 739 Daily Temperatures (Medium) — passed

Anton: this felt unnatural. That is normal. The code is short. The **thinking** is the thing to restudy.

---

#### What the question asks (plain)

You get one temperature per day. For **each** day, return **how many days you wait** until a **strictly hotter** day. If none, `0`.

`[73, 74, 75, 71, 69, 72, 76, 73]` → `[1, 1, 4, 2, 1, 1, 0, 0]`

Equal is **not** warmer (`71` then `71` does not count).

You return an **array** (one answer per day), not one number.

---

#### Why Stack is picked (read this first)

**General — when an interviewer wants a stack**

Pick a stack when the thing you still need to finish is the **most recent unfinished item**, and a later event **resolves it first** (LIFO). You do not need random lookup (HashMap). You do not need oldest-first (queue). You need “last waiting, first answered.”

Notice the wording: **next** greater / next warmer / matching closer / undo last. That “next” is in **time order** to the right, and it always answers the **nearest** waiting item before ones further back.

**Here — why #739 is that**

Each day is unfinished until a **later hotter** day exists. The waiting days must stay in time order. A new hot day answers the **nearest** colder days first (the ones just before it), then maybe older ones. That is exactly LIFO → stack of **indexes**.

You pick stack because:

1. You must remember **several** unfinished days, not one running min (#121).
2. The next hotter day answers them **nearest-first**, not oldest-first (that would be a queue, and it would be wrong: 75 is older than 69, but 72 answers 69 **before** it can answer 75).
3. You only ever need the **last** waiting day (the top). If today cannot beat the top, it cannot beat anyone under it (those are hotter). So a list/array you scan every time is wasted.

**Why not the other patterns**

| Pattern | What it is for | Why not today |
|---|---|---|
| **Two pointers** | Sorted array, or two **ends**; pointers **never reset** | This array is calendar order, not sorted. A worker that goes back to `i+1` **resets** → O(n²). |
| **#121 running min** | One pass, one **best-so-far**, **one** answer | You need an answer for **every** day. A single min cannot fill `answer[i]` for all i. |
| **HashMap** | “Have I seen this key?” | Not a lookup. Order and “next to the right” are the whole problem. |
| **Queue** | Oldest waiting first (FIFO) | Next warmer answers the **nearest** colder day (top of pile), not the oldest. |

**Interview pick-line**

> I need the next greater to the right for every index. Unresolved days are answered nearest-first, so I keep them on a stack. Two pointers would reset and go quadratic. Running min only gives one answer.

---

#### The thinking shift (this is the hard part)

**Natural (what you wanted):** stand on day `i`. Send a worker forward until it finds hotter. Then `i++` and **reset** the worker to `i+1`.

That is **correct**. It is also **O(n²)**. The worker **re-reads** the same later days for every `i`. Two integer variables do not make this the Two Pointers pattern.

**Interview Two Pointers (general):** the array is **sorted**, or you start at **both ends**. Each pointer only moves **forward** (or inward) and **never resets**. Total work O(n). Example: two-sum on a sorted array.

**New thinking for this problem (stack):** do **not** stand on a day and look into the future. Walk through the calendar **once**, left → right. Days that have no answer yet **come with you** on a pile. When a **hotter** day arrives, it turns around and answers the waiting days.

Same inversion as **parentheses**: you do not, from each `(`, scan forward for `)`. You wait. A later `)` **comes** and closes the **last** unmatched `(`.

| How you think | Picture |
|---|---|
| Natural | “From here, look ahead.” |
| Stack | “Unfinished days wait. The future comes to them.” |

**#121 (running min) is a different shift:** one pass, keep cheapest price so far, one profit number. You never go back and fill **every** old day’s answer. Today you must fill **every** index.

---

#### What a stack is (general, any problem)

A stack is LIFO: you only see the **top**. Interview use: **the last item that is not finished yet**.

- Parentheses: last unmatched opener.
- Min Stack (#155): last min, so `pop` can undo.
- Queue from stacks (#232): reverse order.
- **Next greater (#739):** last day that still has no hotter day.

**Next greater (general):** for each value, find the next element to the **right** that is **bigger**. #739 is that, plus the **distance** (`j - i`) instead of the bigger value itself.

People call this a **monotonic stack**: temperatures on the pile go **decreasing** toward the top (hotter days sit **under** colder recent ones). You only compare with the top. If today cannot beat the top, it cannot beat anyone **under** the top either — so you stop the `while` and just push.

---

#### The pile (this problem)

Each plate on the pile is a **day index** (`0, 1, 2, …`), not the temperature. You need the index to write `answer[old] = today - old` and to look up `temperatures[old]`.

`answer` starts as all `0`. Leftover plates at the end never found hotter → they stay `0`.

**Every new day `i`:**

1. While the pile is not empty **and** today is **strictly hotter** than the day on top: that old day is done. `old = pop()`, `answer[old] = i - old`. Repeat (one hot day can finish several colder days).
2. Always `push(i)` — today has not found **its** hotter day yet.

The `if` after the `while` is optional: after the `while` you always want to push `i`.

---

#### Tiny walk — learn this one first: `[70, 71, 69, 72]`

`answer` starts `[0, 0, 0, 0]`. Pile holds **indexes**.

| Today | Pile before | Action | Pile after | `answer` |
|---|---|---|---|---|
| day 0 = 70 | empty | nothing to pop, push 0 | `[0]` | `[0,0,0,0]` |
| day 1 = 71 | `[0]` | 71 > 70 → `answer[0] = 1-0 = 1`, pop, push 1 | `[1]` | `[1,0,0,0]` |
| day 2 = 69 | `[1]` | 69 > 71? no. push 2 | `[1, 2]` | `[1,0,0,0]` |
| day 3 = 72 | `[1, 2]` | 72 > 69 → `answer[2] = 1`. 72 > 71 → `answer[1] = 2`. push 3 | `[3]` | `[1, 2, 1, 0]` |

Day 3 (72) never looks “from day 1 toward the end.” It only talks to the **top**, then the new top.

LC example (same rules): day 2 (75) stays on the pile while 71, 69, 72 go on top. 72 pops 69 and 71 (colder than 72) but **not** 75. Day 6 (76) pops 72 then 75 → `answer[2] = 4`.

---

#### Why O(n), not O(n log n)

Each index is **pushed once** and **popped at most once**. The inner `while` does not restart a full scan. Total pops ≤ n. Time **O(n)**. Extra space **O(n)** (pile + answer).

The “current + worker that resets” version **re-visits** later days. That is O(n²). Tiny JUnit still passes. `n = 10^5` on LeetCode does not.

---

#### Bugs from today (trap)

- **`answer[i] = …`** — `i` is the **hot** day that arrived. The wait belongs to **`old`** (the waiting day). Write `answer[old]`.
- **`stack.push(temperatures[i])`** — then `stack.peek()` is `73`, and `temperatures[73]` is the wrong slot. Push **`i`**.
- Calling the worker version “two pointers” — two indexes + **reset** = nested scan. Real two pointers **do not reset**.

---

#### Cousin

If they want the **next greater value** (not the wait) → same stack, store `temperatures[i]` in the answer instead of `i - old` ([#496 Next Greater Element](https://leetcode.com/problems/next-greater-element-i/)). Scan **right → left** if they want the previous greater. If `n` is tiny and they allow O(n²) → the worker is fine. **#225** (stack from queues) is a cousin of **#232**, not of this problem — we skipped coding it because yesterday already explained it.

---

#### Interview sentence

> I scan left to right and keep a stack of indexes that have no warmer day yet. When today is hotter than the top, I pop and set answer[old] = today − old. Each index is pushed and popped once, so O(n). That is next-greater, not two pointers — two pointers would reset a worker and go quadratic.

---

### Part 3 — OOP + design

---

#### OOP — one class, one job (SRP)

**General**

Single Responsibility means a class has **one reason to change**. “Prove who this request is” and “decide if that person may call this URL” change for different reasons: crypto/JWT vs product rules. If one class does both, you cannot explain 401 vs 403, and a public GET can start returning 403 because you treated “attendee” as forbidden everywhere.

HTTP status for **auth** is not the service layer. Services throw domain problems (not found, duplicate, no seats). `@RestControllerAdvice` maps those to 404/409. **401/403 happen in the filter chain**, before the controller. The service never runs.

**Here**

| Piece | Job | When it “fails” |
|---|---|---|
| `JwtAuthenticationFilter` | Who — parse Bearer, fill the box, or leave it empty | Bad/expired token → catch `JwtException`, clear box. **Not** 403. |
| `AuthorizationFilter` + `SecurityConfig` | May they hit **this** URL? | Empty box on a protected URL → 401. Known user, wrong role → **403**. |
| Controller + service | Create the resource | Only if the two above passed. Then 201 / 400 / 409. |

An **attendee** JWT **parses**. There is no JWT exception. The filter stores `ROLE_ATTENDEE` and continues. 403 on `POST /api/events` is the **matcher** `hasAnyRole("ORGANIZER", "ADMIN")`.

Same token, three URLs, three outcomes — the filter cannot own 403:

- `GET /api/events` → 200 (`permitAll`)
- `POST /api/venues` → 201 today (`authenticated()` only)
- `POST /api/events` → 403 (organizer)

**Trap:** “The filter throws, the service picks the HTTP code.” Wrong exception, wrong layer. Catch in the filter = bad ticket = empty box. 403 = URL rule.

---

#### Design question (what we actually asked)

`POST /api/venues` is any logged-in user. `POST /api/events` is organizer-only. `GET /api/events` needs no token.

**The question:** Is “attendee may create a venue but not an event” a **product rule**, or an **accident** of `anyRequest().authenticated()`? For an Eventbrite-style app, who should create venues? If you lock create-venue, what status for an attendee?

**Your answer:** accident; lock-down → **403**. That is correct.

---

#### How this is “design material” (not a Spring trivia quiz)

Weekday Part 3 is **not** “draw Kafka.” Mid-level interviews (and Friday’s long HLD) keep asking the same family of questions:

1. **Who may do what?** — see vs create vs book. Public vs logged-in vs role.
2. **Where does the truth live?** — identity in the token, rules in the API config, event/venue rows in the DB. Not `role` on the JSON body.
3. **What does the API return when it goes wrong?** — 401 / 403 / 400 / 409 are **part of the contract**, not afterthoughts.

Monday’s drill: create-event is organizer-only; role not in the DTO; 401 vs 403.  
Tuesday’s board: GET catalog public; login 200 / register 201; book later 409 if seats gone.  
Friday: oversell — valid request, state clash, **409**, lock the **row**. Same idea: **named failure**, not a generic 500.

Today’s question is that family on a **hole in the current API**. We listed GET events and POST events on purpose. We **did not** list POST venue. The catch-all said “if you have any identity, OK.” That is a **default**, not “we decided attendees own the venue list.”

| Idea | General (any API) | This app |
|---|---|---|
| Public vs protected | Catalog vs mutate | GET events vs POST events / venues |
| Explicit vs default | Spell out who may create | `hasAnyRole` vs leftover `anyRequest` |
| 401 vs 403 | No identity vs wrong role | No token vs attendee on create-venue |
| 409 | Valid + allowed, **state** clash | Email taken; later no seats — not this drill |
| Trust boundary | Body = resource, token = actor | Venue JSON has no `role` |

Eventbrite-style product (say this in a design interview):

- **Guest** — browse catalog (GET). No token.
- **Attendee** — book seats (later POST bookings). Not create inventory.
- **Organizer / admin** — create **venues and events** (POST). Attendee on those URLs → **403**. Missing token → **401**.

So the question **covered:** public vs protected, defaults vs policy, 401 vs 403 as the contract, “who mutates inventory,” and that adding a rule is a **matcher** (product), not a new filter (identity).

It did **not** cover: locking the event row, 10× traffic, Kafka. Those stay Friday.

---

#### Where you type the venue rule (Spring weave-in)

**Do not** change `JwtAuthenticationFilter`. It already puts `ROLE_ATTENDEE` / `ROLE_ORGANIZER` in the box. A new URL does not change “how we read the ticket.”

**Do** add a matcher in `SecurityConfig`, **inside** `authorizeHttpRequests`, **with** the other path rules, **before** `anyRequest()` (first match wins):

```java
.requestMatchers(HttpMethod.POST, "/api/venues").hasAnyRole("ORGANIZER", "ADMIN")
```

`addFilterBefore(...)` before `return http.build()` is only “put the JWT filter in the chain.” That is not where URL policy goes. Mixing those two “ends of the method” is the trap.

---

**Interview sentences**

> Filter = who. Config = may they hit this URL. Controller = the resource.  
> anyRequest authenticated is a default, not a product decision. Attendee create-venue should be 403 like create-event.  
> That is the same design family as Monday’s 403 and Friday’s 409: the API says what wrong looks like.

### 60-sec (add to today’s story)

> The JWT filter does not return 403. An attendee token parses; the URL matcher refuses create-event. Create-venue 201 for an attendee is the catch-all, not Eventbrite policy. Lock it with another matcher, not a filter change. 403 means we know you; 401 means we do not.

---

**Next:** `Start Week 2 Day 4` (Thu) — booking v1 (`POST .../bookings`), not a new JWT filter.

---

## Week 2 Day 4 — Booking v1 (`POST .../bookings`)

**Date:** 2026-08-27  
**Goal:** Attendee with a JWT can book seats. Who comes from the box, not JSON. Not enough seats → 409. No lock yet (Week 3).

### Quick recall (the pictures — enough to say out loud)

**Booking row**  
- **General:** A sale is its own record (who, what, how many), not only a number on inventory.  
- **Here:** `Booking`: event, user, seats.

**Three sources**  
- **General:** Body = the resource. URL = which resource. Token = who. Never put who/role/id in JSON if the client could fake it.  
- **Here:** Body = `{ seats }`. URL = event id. Box = user.  
- **Trap:** Body `eventId` + URL id can disagree — one source: the path.

**The box / `getName()`**  
- **General:** Filter already put identity in Spring’s static box. Service reads it; you do not inject “current user.” `getName()` is whatever was stored as the principal name.  
- **Here:** We stored **email** → `findByEmail`.  
- **Trap:** Not a bean. Not ThreadLocal as a spoken requirement (the picture is enough).

**400 vs 409 vs 401 vs 403**  
- **General:** **400** = junk input. **409** = valid, **state** says no. **401** = we don’t know you. **403** = we know you, this URL is not for your role.  
- **Here:** `seats: 0` → 400. Sold out → 409. No token → 401. Organizer on book → 403. Attendee **may** book.  
- **Trap:** 409 is not “duplicate email.” 403 is security; 409 is the service.

**One DB bucket (`@Transactional`)**  
- **General:** Check + insert + decrement in one commit; throw → undo **that request**. Does **not** lock Maria’s request.  
- **Here:** On `book()`. Lock = Week 3.

### What I built

- `Booking` table: event, user, seats
- `POST /api/events/{eventId}/bookings` → 201, attendee only
- Sold out → `InsufficientSeatsException` → 409
- User from the box (`getName()` = email) → `findByEmail`. Then lower `availableSeats`

### Gate (my words — these were enough)

1. Box is what the JWT filter fills. Authorization uses it. Not a bean — static.
2. Authentication = who. Authorization = what you’re allowed to do.
3. 400 = bad request. 409 = conflict (sold out).
4. Transaction = one bucket of DB actions. Fail → undo that bucket. Throw sold-out → nothing should stick.
5. DTO so we choose what the client sees — don’t send the whole entity.

*(Names like ThreadLocal are optional reread, not required to say.)*

### Traps from today (keep)

- Body `eventId` + URL event id → mismatch. One source: the path.
- Skip the seats check → booking rows grow, inventory never moves.
- `POST /api/events` (create) is not `POST /api/events/5/bookings` (book). Guest 401 is the security chain, not EventController.
- Dummy `new Booking()` is not needed: `throw` then save — the save only runs if you didn’t throw.

### 60-sec

> Book: seats in JSON, event in the URL, user in the box. Sold out is 409, junk JSON is 400. Organizer on book is 403, no token is 401. Seat check then save booking and lower available seats in one bucket. No lock yet.

---

### Part 2 — LC 150 Evaluate Reverse Polish Notation (Medium) — passed

Tokens are numbers and `+ - * /`, already in **postfix**: the operator comes **after** the two numbers it uses. Return one integer.

`["2","1","+","3","*"]` → `(2 + 1) * 3` = **9**. Division cuts toward zero.

---

#### Why Stack (general → here → trap)

**General:** pick a stack when the thing you still need is the **last unfinished item**, and the next event uses that first.

**Here:** unfinished items are **numbers** waiting for an operator. A `+` (or `-` `*` `/`) always takes the **last two**, puts one result back. More than two numbers can sit on the pile; the operator still only takes two.

**Trap:** “total amount” / one running number. That is #121. Here several numbers wait. One `*` is **not** `7*3*8*9` — only the last two. You do **not** need “exactly two numbers then an operator” every time (`4, 13, 5, /, +` has three numbers before `/`).

---

#### Why not the cousins

| Pattern | Why not today |
|---|---|
| **#739** | Unfinished = **days** waiting for hotter. Today unfinished = **numbers** waiting for an operator. Same pile, different thing on it. |
| **HashMap** | Not “have I seen this key?” Order of the last two numbers is the problem. |

**#739 vs #150 in one line:** hotter day **answers** waiting days. Operator **consumes** waiting numbers.

---

#### The walk (learn this)

`4, 13, 5, /, +`

- numbers → pile `4, 13, 5`
- `/` → last two: `13 / 5` = `2` → pile `4, 2`
- `+` → `4 + 2` = `6`

`-` and `/` care about order: first pop is the **right** side. `5, 2, -` is `5 - 2`, not `2 - 5`.

Each token once → **O(n)** time. Pile **O(n)** extra.

---

#### Cousin

Infix (`2 + 1 * 3`, operator **between**, precedence) is not this scan. Nested unfinished strings → [#394 Decode String](https://leetcode.com/problems/decode-string/).

---

#### Interview sentence

> I scan left to right and keep numbers on a stack. An operator pops two (first pop is the right side) and pushes the result. One pass, O(n).

---

**Next:** Part 3 in the booking-app chat (OOP + short design). Then `Start Week 2 Day 5` (Fri) — long design (oversell / last seat). Weekends off.

---

### Part 3 — OOP + design

**OOP:** Event service = concert. Booking service = ticket. Don’t put `book()` on EventService because the URL sits under `/events`. URL grouping ≠ one class.

**Design:** Quantity is `seats` on one call. Double-click ≠ two tickets. A timer (“if under X seconds, 409”) is a guess. Fail duplicate → 409, in the service. Two different people, last seat → Friday.

**Spring:** Organizer on book = **403** (security: wrong role). Sold out = **409** (service: data). Not the same “no.”

### 60-sec (add)

> Booking and event are two jobs. Seats in one JSON; double-click is not a second ticket. 403 is security; 409 is the service.

---

**Next:** `Start Week 2 Day 5` (Fri) — long design: last seat, two people, lock. Small OOP tie-in. **Sat/Sun off.**

---

## Week 2 Day 5 — Booking slice tests + last-seat HLD

**Date:** 2026-08-28  
**Goal:** Prove book in tests (201 / 401 / 409) without a JWT or H2. Then design: two people, one seat, lock the event row. No lock code today (Week 3).

### Quick recall (tests)

**`@WebMvcTest` + `@MockitoBean`**  
- **General:** Web slice: controller + MockMvc, no JPA. Fake the **service**. Does not prove login, DB, or two people grabbing the last seat.  
- **Here:** `BookingControllerTest` mocks `BookingService`.

**`@WithMockUser`**  
- **General:** Test-only. Fills the **same box** the JWT filter would. No token. Unnamed `@WithMockUser("X")` is **username** (who). `roles = "ATTENDEE"` is **allowed**. Empty box on a protected POST → **401**; skip the stub (controller never runs).  
- **Here:** 201 and 409 tests use `roles = "ATTENDEE"`. `doBooking1` has no annotation → 401.  
- **Trap:** `spring-boot-starter-security-test` is only the jar.

**`@Import(GlobalExceptionHandler)`**  
- **General:** `@WebMvcTest` does not load `@RestControllerAdvice`. Thrown domain exception → **500** unless you import the advice.  
- **Here:** `InsufficientSeatsException` → 409 only with the import.

**`.with(csrf())`**  
- **General:** App may turn CSRF off in `SecurityConfig`. The slice often does **not** load that config. Default MockMvc still wants CSRF on POST.  
- **Here:** All book POSTs in the test use `csrf()`.

### What I built

- `spring-boot-starter-security-test` (`scope=test`)
- `BookingControllerTest`: attendee **201**; no user **401**; attendee + `InsufficientSeatsException` **409**

### Gate (weak spots)

- **Q1:** The starter is only the jar. The annotation is `@WithMockUser`.
- **Q3:** No advice in the slice → 500. `@Import` the handler.
- 409 on book = sold out (state), not “duplicate email.” 401 = we don’t know you, not 403.

### Traps

- `@WithMockUser("ATTENDEE")` ≠ role.
- Two test methods, two stories. Don’t comment out the 201 annotation to make a 401 test.
- Green tests do **not** prove login or the last-seat race (we mock `book`).

### 60-sec (tests)

> `@WithMockUser` puts a user in the box so URL rules pass. Mock the service — no DB. Empty box → 401. Sold-out throw needs the advice imported or you get 500. Not a JWT test. Not a race test.

---

### Part 3 — last seat (HLD)

**Boxes:** phone → POST book → security (box filled) → controller → `BookingService` → repo → DB (`availableSeats`).

**Today’s bug (no lock):** Anton and Maria both read **1**, both pass the check, both **201**, both write **0**. Two booking rows, one seat. Oversell. Not 500. Not usually **-1** with our Java `1-1=0` (both copies). **-1** if the DB subtracts twice with no lock.

**`@Transactional`:** one bucket **per request**. Does **not** make Maria wait. Two buckets can both see 1.

**Lock:** the **event row** (concert 5), not the table (too slow), not the venue (other shows), not a `HashSet` (one JVM). GET event 5 stays a normal read — do not lock browse. Other concerts use other rows.

**After Anton wins:** lock, 1→0, commit, 201. Maria was waiting **in the same `book()`**. She reads **0** → **409**. Screen can still show 1 (old GET). API does not need a refresh.

**Lock + subtract in one transaction.** Commit **releases** the lock. Split lock/commit then subtract later → Maria can 201 in the gap.

**10×:** bottleneck = many **book** calls on **event 5** (line on that row). Browse and other events stay fine.

**OOP:** locking read in `BookingServiceImpl` (ticket). Not the filter (who). Not `EventService` (concert).

**Week 3:** write the lock. Not today.

**Interview sentences**

> `@Transactional` is not a lock. I lock the event row. Loser gets 409 in the same call. GET does not take that lock. Commit ends the lock so check and decrement stay in one transaction.

---

**Next:** LC in the other chat (stack). Then **Sat/Sun off.** Monday: `Start Week 3 Day 1` — lock in code.

---


