# Interview notes — Weeks 10 and 11

Added by Anton on 2026-10-07.
Goal: recall what was already learned, under a real clock, without knowing the family in advance.
No new families, no new design products.
Mock interviews happen outside this platform.

Week 10: Mon 2026-10-19 to Fri 2026-10-23.
Week 11: Mon 2026-10-26 to Fri 2026-10-30.
Sat/Sun **off**.

**Book path, in call order:** [book-path.md](book-path.md). Read that from start to end. The day notes are the quiz, not the order.

**Split-the-bill study notes:** [split-bill-api.md](split-bill-api.md). API and balances. Closed 2026-10-21.

**Day 1 Part 1 closed** (Mon 2026-10-19).
LCs done. S1 and S2 done. Ch 11 token bucket recap done.
Do not re-ask S5. Do not repeat the Ch 10 consistent-hashing prompt from Week 8.
Ch 11 is closed for this recap: one token had dripped, first Book **201**, immediate second tap **429**. He first said click-id mismatch.

**Coding Anton asked for on 2026-10-19, after Part 1.** Walk it tomorrow, one piece at a time. Part 3 after that is the design boards (event search, then the Kubernetes deploy).

- Book gate is a token bucket: 5 tokens, one back every 15 seconds. Compose uses Redis. A Maven run keeps the jar in memory. Key is the JWT subject, otherwise the IP.
- Compose uses PostgreSQL, one server, a database each for Auth, Event, and Booking. Maven and the tests still use H2.
- The UI is `event-booking-demo`. It already calls the gateway. `js/config.js` picks the API address.
- Booking, Event, and gateway tests passed. `legacy-monolith-tests` is not on the Maven test path.
- UI is on GitHub Pages: https://antondahdal.github.io/event-booking-demo/
- API blueprint is on Render (`render.yaml`). First image build failed because `.dockerignore` dropped `src`. Fix is on `master` (`bc24966`). Services were still building after that push.

## Hosting — say this

Two repos. Two pipelines. This is not Jenkins.

Jenkins is a CI server you run yourself. A Jenkinsfile lists stages (test, build, deploy). An agent runs those stages. You own the server.

Here nobody runs Jenkins. A push still starts work, but two hosted products do it.

**The page.** Repo `event-booking-demo`, branch `main`. The UI is static HTML and JavaScript. No Java process. On every push to `main`, GitHub Actions runs the workflow `Publish demo` on a GitHub Ubuntu machine. The steps are: check out the repo, pack the folder, publish it to GitHub Pages. That is the piece that looks like a Jenkins job: a file in the repo, a trigger on push, steps on a runner. The live page is https://antondahdal.github.io/event-booking-demo/

`js/config.js` chooses the API. On localhost the page calls `http://localhost:8081`. Anywhere else it calls `https://event-booking-gateway.onrender.com`. The browser talks only to the gateway.

**The API.** Repo `SpringBoot-bookingApp`, branch `master`. `render.yaml` is the blueprint: one free Postgres, one free Redis, four web services (auth, event, booking, gateway). A push to `master` makes Render pull the repo, build a Docker image from that service's Dockerfile, and start the container. Maven runs inside the image build, so the runner compiles the jar. Render then injects `PORT`, `DATABASE_URL`, `REDIS_URL`, the shared `JWT_SECRET`, and each service's public URL. Health checks are `/actuator/health` on auth, event, and booking, and `/health` on the gateway.

Free web services cannot call each other on a private network. Auth, Event, and Booking call the public `https://….onrender.com` URLs. On a real cluster those calls would use private service names. The gateway still sits in front: the page never calls Booking directly. The gateway allows the Pages origin `https://antondahdal.github.io`, including on a 429, or the browser hides the status.

Free limits, if they ask: each service sleeps after 15 minutes with no traffic, and the first request takes about a minute to wake it. The free Postgres expires 30 days after it was created. Redis on this plan is memory only and forgets its data on restart. One Postgres database is shared; the table names do not collide.

**Interview line.** GitHub Actions publishes the static page the way a Jenkins job would: a workflow file, a push trigger, steps on a runner. Render is not Jenkins. It is the host. The blueprint says what to run, and a push builds the image and rolls the service. I would use Jenkins if I needed one place to test and deploy onto servers I operate.

**Trap.** Calling Render a Jenkins server. Calling the GitHub Pages site a Java app. Saying the browser calls Booking directly.

## Part 3 — shows page, from the app (2026-10-19)

No new search box. The app has no search by words. `GET /api/events` is `getAllEvents`: a page of ids, then those concerts with venues, no cache, so each open reads Event's database. `getEvent` is the cache, and only after the id is known. Order: click, Gateway, Event, database. Status **200**. A million of those opens while Maria books: he said her Book is slow because of the connections on that database. That was the answer. The seat row still decides **201** or **409** after she gets a connection. Kubernetes deploy board, from `k8s/event.yaml`. Old pod stays until the new one is ready, then it is killed. He first had the new pod taking the concerts from the old one, then the load balancer. Concerts stay in Postgres. Both pods read that database during the swap. He said the old pod's call to the database breaks if the new image renamed a column. That call fails because the old SQL still uses the old name. Add the new column first, ship the image, drop the old column only after the old pod is gone. A Postgres blip fails readiness, not liveness. He first said the pod restarts. Then he said removed: the Service drops that pod, and the pod stays. HPA cap is 3, so a fourth Event pod does not start. `app-secret` is base64, not encryption. The real JWT is not in git. `k8s/postgres.yaml` is one Postgres pod. `init.sql` creates `authdb`, `eventdb`, and `bookingdb` on that server. Deploy board closed.

---

## Day shape (Mon–Fri, same every day)

| Slot | What | Clock |
|---|---|---|
| LC 1 | **Redo**: one problem he already did, from the redo pool below | Medium 25 / Easy 15 |
| LC 2 | **Fresh**: one Top Interview 150 problem he has never done | Medium 25 / Easy 15 |
| Spring | Two Spring Boot interview questions | ~20 min |
| Design recap | **Mon and Wed only**: one weak LC-SD piece | ~10 min |

Mocks are done elsewhere; if he shares the feedback, add the weak spots to the pools here.

## Family is hidden

The point of both LCs is the pattern call.
The coach picks the problems **on the day**.
Do not write the day's problems in this file or in the map ahead of time.

In the Cover, never say the week, the family, or "you did this in Week 7".
For a redo, give the problem as if it were new: plain words, the LeetCode URL, the picture.
Then the same gate: pattern, why, time and extra space.
Same rules as before: no hints, no indirect hints, swap-or-learn if he does not know the why.

## LC 1 — Redo rules

Pick from the pool, oldest weak spot first inside the priority order.
Do not pick two from the same family on the same day, or the same family two days in a row.

Clear it (on-time, he coded it, no coach fix): **retired**, cross it off.
Miss it (time up, coach-fixed, coach-written): back in the pool, earliest **three weekdays** later.
Notes record which hole came back, not a full new write-up; the review sheet section already exists.

## LC 2 — Fresh rules

Top Interview 150 only, and not already in LC-Practice.
Medium or Easy, no Hard.
Any family, including ones he has seen, but not the same family as today's redo.
Stub and JUnit in `src/main/java/questions/week10/` (or `week11/`) and `src/test/java/test/week10/`.
After it closes, add it to the review sheet like any other LC.

## Spring Boot question rules

Two questions each weekday, ~20 min together.
Same format as Week 9: topic and one sentence why for the role, short explain, he talks first, then trap and one interview sentence.
Pick first from the Week 9 rows he was weak on ([week-09.md](week-09.md)), then from the Week 9 Part 2 topics (Hibernate, Docker, Kubernetes, Spring AI).
Tie it to the booking app where it fits.
Do not re-ask a row he answered well in Week 9.

## Design recap rules (Mon and Wed)

About 10 minutes, after the Spring questions.
Only pieces that were weak in the LC-SD notes; no new chapters, no new products.
Same format as LC-SD: chapter and topic with the course URL, he talks first, then trap and one interview sentence.
Use a new angle on the same piece, not the prompt he already saw.
Read the Week 8 and Week 9 notes first so a piece recapped there is not asked again.

Weak pieces still open on 2026-10-07:

| Piece | Why it is weak |
|---|---|
| Ch 11 token bucket | Thought every limiter resets on the minute; words alone did not land |
| Ch 10 consistent hashing | Recapped W8 Day 4, still mixed up which scheme moves only the dead node's keys |

## Redo pool

Priority: Weeks 7–8 first (newest and weakest), then Week 6, then Weeks 3–5.
Week 9 misses join the pool when Week 9 closes.

| Priority | # | Problem | Last result |
|---|---|---|---|
| 1 | 200 | Number of Islands | coach-written |
| 1 | 207 | Course Schedule | coach-written |
| 1 | 198 | House Robber | coach-written |
| 1 | 146 | LRU Cache | coach-written |
| 1 | 69 | Sqrt(x) | unfinished |
| 1 | 35 | Search Insert Position | time up, coach-fixed |
| 1 | 33 | Search in Rotated Sorted Array | coach gave the fix |
| 1 | 215 | Kth Largest Element in an Array | **retired** 2026-10-19, on-time. Clock version was a max-heap of every value. |
| 1 | 56 | Merge Intervals | overtime |
| 1 | 228 | Summary Ranges | overtime, coach-fixed |
| 1 | 452 | Minimum Number of Arrows to Burst Balloons | overtime, coach-fixed |
| 1 | 57 | Insert Interval | coach-fixed, no clock |
| 2 | 102 | Binary Tree Level Order Traversal | overtime, coach-fixed |
| 2 | 98 | Validate Binary Search Tree | overtime |
| 2 | 230 | Kth Smallest Element in a BST | overtime |
| 2 | 199 | Binary Tree Right Side View | overtime, coach-fixed |
| 2 | 100 | Same Tree | overtime |
| 2 | 112 | Path Sum | overtime, coach-fixed |
| 2 | 236 | Lowest Common Ancestor of a Binary Tree | coach-fixed |
| 2 | 101 | Symmetric Tree | overtime, coach-fixed |
| 2 | 572 | Subtree of Another Tree | untimed |
| 3 | 92 | Reverse Linked List II | overtime, coach-fixed |
| 3 | 82 | Remove Duplicates from Sorted List II | overtime |
| 3 | 143 | Reorder List | coach-fixed |
| 3 | 2 | Add Two Numbers | coach-fixed |
| 3 | 160 | Intersection of Two Linked Lists | on-time, still buggy |
| 3 | 36 | Valid Sudoku | coach walked it |
| 3 | 11 | Container With Most Water | parked, passed next day |

Two weeks give ten redo slots, and the pool has 27 after #215 retired, so it will not be cleared.
Priority 1 alone is 11, so it fills most of the two weeks.
Priority 2 and 3 only get a slot when priority 1 is cleared or waiting out its three days.

## Done

| Day | Redo | Fresh | Spring | Design recap |
|---|---|---|---|---|
| Mon 2026-10-19 | #215 on-time, retired | #55 Jump Game, overtime, coach-patched | S1 closed (DataSource vs entity). S2 closed (env var beats the file) | Ch 11 token bucket closed |
