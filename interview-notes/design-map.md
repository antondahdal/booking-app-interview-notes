# Design map (mid-level Java)

This is the **design** source of truth (same job as the OOP tables). Part 3 **calendar** (which day, which prompt) still lives in [oop-design-map.md](oop-design-map.md). If a topic is **Done**, do not run the same prompt again.

**Role:** mid-level Java / Spring. They will not ask you to invent Kafka. They **will** ask: who may do this, where the truth lives, which status, what breaks at 10×, what you do when a hop is slow.

**Three kinds of “design” — do not mix them**

| Kind | When | What you draw / say |
|---|---|---|
| **API + this app** | Weekday ~45 min, Weeks 1–5 | One picture from *this* booking app. Statuses. One bottleneck. |
| **HLD** | Friday 60–75 min | Boxes, sequence, 10× traffic, what you lock / retry / drop. |
| **LLD** | Weeks 6–8 only | Classes / fields / methods for a **different** small product. No Java files. |

Sat/Sun **off**.

---

## What they always ask (five families)

Every weekday prompt and every Friday board is one of these. If a prompt is not in this list, it is trivia — skip it.

| Family | Question in human words | Example here |
|---|---|---|
| **Who** | Who may call this? Where does identity live? | Token vs body. 401 vs 403. First security line wins. |
| **Truth** | Which store / service is the number? | Seats on the event row. Later: Event service over HTTP. |
| **Status** | Valid request, but we cannot apply it — what code? | 409 sold out vs 409 stale vs 403 vs 503 vs timeout 500. |
| **10×** | Last seat, 100 POSTs, pool size 10. What waits? | Row lock vs connection pool. Wait vs stamp. |
| **Slow hop** | Bank / other service / email is slow. Do you hold the lock? Retry? | Pay after commit. Timeout. No retry on book unless click id. |

---

## Design — done (do not repeat the same prompt)

| Topic | Family | When | One line | Trap |
|---|---|---|---|---|
| 400 vs 404 vs 409 | Status | W1 D4 | Junk / missing / valid but **state clash** | 409 = “duplicate email” only |
| Pagination as API | API | W1 D4 | `Page` = slice + total. Not a raw `List` | Throw away `Page` metadata |
| Last seat + row lock | Truth / 10× | W1 D5, W3 D1 | `@Transactional` is not a wait. Lock **that event row** | Lock the table / Venue / GET |
| Public vs protected | Who | W2 D1 | Register public. Create event = organizer. Role not in JSON | Check role at register once |
| 401 vs 403 | Who / Status | W2 D1, W2 D3 | 401 = we don’t know you. 403 = we know you, no | Filter “throws 403” |
| Seats on one POST | API | W2 D4 | Quantity is `seats` on that call | Timer “if two clicks under 1s” |
| Click id ≠ JWT | Who / Status | W2 D4, W3 D1 Q7 | Double-click = one intent. Hours later = new click | “JWT twice → reject” |
| `synchronized` / two JVMs | 10× | W3 D1 Q4 | Lock in the **DB** | HashSet of event ids on the service |
| Stale UI / don’t lock GET | Truth | W3 D1 Q5 | POST book is truth. Browse is a snapshot | `FOR UPDATE` on GET |
| Pay after the lock | Slow hop | W3 D1 Q6 | Lock = check + write + commit, then Stripe | One transaction including HTTP |
| Timeout ≠ 409 | Status | W3 D1 Q3 | Never got the row → retry maybe. Saw 0 → stop | Map lock timeout to sold-out |
| Pool vs row | 10× | W3 D1 Q8 | Row = event 5. Pool = doors to the **whole** DB | “Other concerts are always fine” |
| Wait vs stamp (when) | Truth / 10× | W3 D2 | Hot counter → wait. Quiet title → stamp | Optimistic = “one row” |
| Door A vs Door B | Status | W3 D2 | Sold out 409 → **stop**. Stale 409 → **retry** | Every 409 = stop |
| Title bump ≠ sold out | Status | W3 D2 Q5 | Stamp is the **whole row** | Any version change = no tickets |
| Two servers, one DB | 10× | W3 D2 Q4 | Stamp/`FOR UPDATE` run in **one** database | `@Version` like `synchronized` |
| Rollback with the bucket | Truth | W3 D2 weave | Event write fails → booking insert rolls back | `save` already committed |
| Two tools, one row | Truth / Status | W3 D3 | Book waits. Title PATCH uses stamp. Name who commits first | Organizer 409 = sold out |
| Isolation see vs write | Truth | W3 D3 | `findById` both see `1`. `FOR UPDATE` waits then sees `0` | Version stops the **read**. Refresh stops the race |
| PUT vs PATCH + click id | API | W3 D3 | PATCH = title only. PUT = full replace. Two Books = two attempts | JWT is the click id |
| First matcher 403 ≠ 409 | Who / Status | W3 D3 | `POST /api/events/**` ate Book → **403**. `book()` never ran | 403 = sold out |
| HLD last seat at 1 / 10 / 100 | 10× / HLD | W3 Fri | Row waiters hold pool doors. Stamp ≠ sold-out 409. Don’t stamp Book because concert 9 starves | Maria waits before HTTP. Every 409 = sold out. Wait on Event → 503 |
| Correlation id | Slow hop | W4 Day 3 leftover (Tue slot) | Log sticker. Keep if sent, else mint. Copy to Event/Auth. Return so the phone can see it | Mix with JWT / click id. Forget copy → throw |
| HTTP test vs mock | Truth | W4 Day 4 | Mock 201/409 = handler wiring. IT = hop + seats dropped. Autowired `book()` ≠ Tomcat | Mock = Event ran. Concurrent test = EventClient HTTP |
| Three-box HLD | HLD | W4 Fri | Phone → Booking → Event (seats) / Auth (who). Filter ≠ Auth box. Hang → 502, unknown if Event wrote. `@Transactional` holds a connection, not `FOR UPDATE` | EventClient is a box. Timeout = Event wrote nothing. Correlation id = click id. Filter = Auth service |

**Asked, not built (keep as interview words only until code exists):** click id / idempotency key. Do not pretend it is in the app.

---

## Design — still need (this role)

Same shape as the OOP “still need” table. Must-have on a mid-level board. Each has a **slot** so Part 3 does not wander.

| # | Topic | Family | Why they ask | Slot |
|---|---|---|---|---|
| 6 | **Truth across HTTP** | Truth / Slow hop | Seats in Event **service**. What if Event is slow / 503? No 2PC. | W4 Mon |
| 8 | **Map downstream failure** | Status / Slow hop | Event 404 / 409 / 503 / timeout → Booking’s status. Retry or not. | W4 Wed (OCP talk still open) |
| 11 | **JWT at gateway vs service** | Who | Check at the edge? Again inside? | W5 Mon |
| 12 | **Retry which calls** | Slow hop | GET may retry. Book only with a **click id**. | W5 Tue |
| 13 | **Circuit + fallback status** | Status | Open circuit ≠ **201 ticket**. | W5 Wed |
| 14 | **Live vs ready + kill a holder** | 10× | Pod dies holding `FOR UPDATE`. | W5 Thu |
| 15 | **Gateway HLD** | HLD | Traffic + time budget. | W5 Fri |
| 16 | **Outbox / at-least-once mail** | Slow hop | Email after commit. Duplicate mail possible. | W6 Mon–Thu |
| 17 | **Async HLD** | HLD | Book path vs notify path. | W6 Fri |
| 18 | **N+1 and index as bottleneck** | 10× | What the user feels, what you measure. | W7 Mon–Tue |
| 19 | **Cache** | Truth / 10× | Cache-aside, TTL. Do not cache “1 seat left” as gospel. Book is truth. | **W7 Wed (longer)** |
| 20 | **Classic HLD + 429** | HLD | URL shortener. Rate limit **429** ≠ sold-out **409**. | **W7 Fri (longer)** |
| 21 | **Mock HLD + one LLD** | HLD / LLD | Week 8 Friday. | W8 Fri |

**Should-know one-liners (already have a slot, or 2 min if they ask):**

| Topic | Say this | Slot |
|---|---|---|
| Hold / reservation | “I was looking so I own the seat” is a **hold with a TTL**, not GET + lock | If they push on stale UI |
| No 2PC | Two services, two DBs → not one giant DB transaction | W4 Mon + W6 |
| At-least-once vs exactly-once | Duplicate mail possible; consumer must be safe | W6 |
| CAP | Book wants one latest count; browse may be stale | W6 Fri if they ask |

**Skip for this role unless they bring it:** multi-region active-active, custom Raft, Kafka internals, “draw the whole AWS account.”

---

## How a weekday 45 min should feel

Not five short quiz questions. **One prompt**, then stay on it:

1. Actors (Anton / Maria / organizer).
2. Sequence (who calls what, what the DB does).
3. Each caller’s **status**.
4. One change (“now 100 people” or “Event service is down”) — what breaks.

Your words first. Then trap. Then one interview sentence.

Friday = same families, **bigger picture** (boxes + 10×), not a new subject.

---

## LLD (Week 6+, not this app)

Bank and rules: [oop-design-map.md](oop-design-map.md) (bottom). Do **not** start in Weeks 1–5.

---

**LC-SD (from W4, Part 1 only):** talk from [System Design for Interviews and Beyond](https://leetcode.com/explore/interview/card/system-design-for-interviews-and-beyond), ~15 min, Mon–Thu. Always **Chapter N + topic**. From W4 Tue: explain a bit, then the question. Calendar: [lc-sd-map.md](lc-sd-map.md). Do not run the same product again in Part 3 that day. **URL shortener** stays **W7 Fri Part 3**, not Part 1.

## Next session — Week 4 Friday Part 3 leftover

Part 1 + Part 2 Day 5 **done**. Do **not** rerun three-box HLD, last-seat wait, hang=502, `@Transactional` vs lock.

**Still today:** small OOP **OCP leftover** (new Event error → mapping in the client, not a giant `if` in `book()`). Then Sat/Sun **off**. Week 5 = gateway.
