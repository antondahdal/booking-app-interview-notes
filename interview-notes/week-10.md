# Interview notes — Weeks 10 and 11

Added by Anton on 2026-10-07.
Goal: recall what was already learned, under a real clock, without knowing the family in advance.
No new families, no new design products.
Mock interviews happen outside this platform.

Week 10: Mon 2026-10-19 to Fri 2026-10-23.
Week 11: Mon 2026-10-26 to Fri 2026-10-30.
Sat/Sun **off**.

**Not started yet.** Say `Start Week 10 Day 1` when you get here.

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
| 1 | 215 | Kth Largest Element in an Array | overtime, coach-fixed |
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

Two weeks give ten redo slots, and the pool has 28, so it will not be cleared.
Priority 1 alone is 12, so it fills most of the two weeks.
Priority 2 and 3 only get a slot when priority 1 is cleared or waiting out its three days.

## Done

| Day | Redo | Fresh | Spring | Design recap |
|---|---|---|---|---|
