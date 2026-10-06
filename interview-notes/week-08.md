# Interview notes — Week 8

Interview ready: demo, architecture recap, mocks.

**Packed:** **two** Part 1 topics each Mon–Thu. Friday = long mock. Full calendar: [part1-map.md](part1-map.md).

| Day | Topic 1 | Topic 2 |
|---|---|---|
| Mon | Demo path | Architecture recap notes |
| Tue | Weak-topic drill | One fix |
| Wed | Mock prep | Leftover |
| Thu | Dry-run answers | Polish |
| Fri | Long mock (HLD + LLD) | — |

**Part 3:** one mock includes a 15–20 min LLD (classes / fields / methods for a small product). Plus demo + HLD recap.

---

## Week 8 Day 1 (Mon 2026-10-05)

### Part 2 — W7 leftovers (seat hold + confirm), closed

- `HoldExpiryJob` (Event side): `@Component`, `@Scheduled(fixedDelay = 5000)`. Finder `findByStatusAndExpiresAtBefore(HELD, now())`, then `eventService.expireHold(id)` per hold, through the bean.
- `Booking.status` (`BookingStatus`, `@Enumerated(STRING)`, not null). `writeBook` sets CONFIRMED.
- `OutboxPoller` 409 catch (`HoldDataExceedTimeException`): booking → CANCELLED + save, **then** row SENT. `DownstreamServiceException` stays PENDING (retry).
- Tests (coach wrote, Anton asked): `HoldExpiryJobTest` (two overdue holds → two `expireHold`; none → never). `OutboxPollerTest` fixed (did not compile) + new test: 409 → booking CANCELLED, row SENT. All pass.

**Checks:**
- Job calls `expireHold` per hold instead of `@Transactional` on the job: Anton — one failure must not roll back the 50 that worked (right). Plus: each row lock is held short.
- Row SENT before the cancel is saved, crash in between: Anton — row updated, ticket not cancelled (half). Other half: SENT is never polled again → ticket stays CONFIRMED forever with no seats. Fix: cancel first; a repeat cancel is harmless.

**Code slips:** `now().minusMinutes(10)` in the job (counts the 10 minutes twice — the deadline is already in `expiresAt`). Forgot CONFIRMED in `writeBook` (with `nullable = false` every Book would fail).

### Part 2 — Demo path

> The user logs in through Auth and gets a JWT. Every request goes through the gateway, which routes it, and the service's JWT filter validates the token. When the user taps Book, Booking calls Auth for the user and Event to reserve seats. Event locks the event row, decrements the seats, and saves a hold with a 10-minute deadline in one commit. Booking then saves the ticket plus two outbox rows (notify and confirm hold) in one commit. A poller delivers them: it sends the email and calls Event to confirm the hold, which is safe to repeat. If the confirm never arrives in time, a job on Event expires the hold and gives the seats back. A late confirm gets 409, and Booking cancels the ticket.

**Gaps in first try:** sounded like one app (no hops); said the gateway validates the JWT (it only routes — each service's `JwtAuthenticationFilter` does); "Event marked HELD" (the hold is a separate row); missed "one commit" on Booking; said the poller confirms (Event confirms, poller only calls).

### Part 2 — Architecture recap

| Box | Owns | Called by |
|---|---|---|
| Gateway | nothing — routes | browser |
| Auth | users, roles; issues the JWT | Gateway (login / register), Booking (who is the user) |
| Event | venues, events, seat holds | Gateway (browse, organizer), Booking (reserve seats; confirm hold) |
| Booking | tickets, outbox rows | Gateway (Book) |

Sync vs later: **reserve seats** is inside the user's request; **confirm hold** and email go later through the outbox poller. Only what the user needs for the answer runs while they wait.

**Calendar:** Part 2 **closed**. Part 3 **not run** (carried again). Spring pushed W8 Day 2 morning.

---

## Week 8 Day 2 (Tue 2026-10-06) — short day, job summit

Anton asked: CV drill instead of the normal day. Part 2 (weak-topic drill + one fix) and Part 3 (carried W7 items) **not run** — carry.

### CV drill

Full answers, examples and edits: [cv-prep.md](cv-prep.md).

- Tell me about yourself: first try was a word list. Shape: who → domain → one win → project → what you want.
- Two projects: Telefónica Hispam + customer migration after acquisition. No internal words ("Tef", "DFS").
- Migration story (strong): locked customers → retry table → second job reads the table, re-validates, reprocesses.
- Batch: described the internal batch framework (`init` / mapper / `run` / `finalActivities`, chunks of 100, rollback then one by one). Mapped to Spring Batch words. **Open:** check for `org.springframework.batch` imports; until then say "Spring-based batch framework".
- SQL tuning: tuned with the DBA (AWR report). First answer "add indexes" = the fix, not what you look for → full table scan / cost / skipped index. Second try good (index on the filtered column, PARALLEL for big batch scans).
- SOAP vs REST: the first answer ("SOAP is XML in WebLogic, REST is JSON") was half right. WebLogic is only the server. SOAP is a protocol with a WSDL contract and WS-Security; REST is a style over HTTP. Mixed up WSDL with encryption and with the token twice. Fixed with real XML and HTTP examples and the "menu vs ID card" picture.
- EJB: didn't know what an EJB is. Learned it, then mapped it to his own work: remote EJB APIs that CRM calls, with the client kit as the contract. His own good answer: EJB for internal Amdocs Java-to-Java calls, SOAP for outside partners like banks.
- Booking app: checked the code. It's one Spring Boot app plus a gateway (a modular monolith with service boundaries), not four deployed services. The CV line was reworded, and the list of what's missing to split it (Booking's `@ManyToOne` links to Event and User, one shared DB, one deployable, the gateway routing only Book) is in the notes.

**Carry:** Week 8 Day 2 Part 2 (weak-topic drill and one fix). Part 3's carried Week 7 items (notification-outbox critique, the 429 and queue board, the URL shortener). CV: one high-availability example with DevOps, one root-cause story.
