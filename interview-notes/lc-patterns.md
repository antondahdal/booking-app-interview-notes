# LC pattern recognition (every week)

Before coding, say in your own words:

1. **Pattern name**
2. **Why this one** (what state you keep while scanning)
3. **Time / extra space** target

**Anton 2026-09-10:** gate has no “why not the 2 closest.” Cousin is after tests pass.

| Pattern | Keep while scanning | Notice when |
|---|---|---|
| HashSet / HashMap | seen values / value→index | "exists?", complement, counts |
| Stack | last unmatched item | nest, match, undo, next greater, unused numbers waiting for an operator |
| Running min/max | best-so-far + answer-so-far | time order, max `A[j]-A[i]` with `j>i` |
| Two pointers | left/right (or slow/fast) | sorted, or two ends; palindrome; 2-sum on sorted |
| Sliding window | left..right contiguous | subarray/substring + constraint |
| Sort then scan | sorted neighbors | duplicates, intervals, 2-sum after sort |
| Prefix / Kadane | running sum or best subarray | range sums, max subarray |
| Binary search | mid of a sorted space | sorted array, or "min X that works" |
| Fast/slow | two speeds on a list | cycle, middle of linked list |
| BFS/DFS | queue / recursion | tree, graph, levels |
| Heap | k best | top K, merge K, running median |
| DP | subproblem table | later — "number of ways" |

## Interview templates (memorize after you name the pattern)

Say the pattern first (that’s the round). Then write this shape.

| Pattern | Memorize this |
|---|---|
| HashMap | One `for i`. Need = target − `nums[i]`. If map has need → return. Else put value→i. |
| HashMap (#219 nearby) | One `for i`. Value → **last index**. If `i - last <= k` → true. Then **put** (check, then overwrite). |
| HashMap (#205 glue) | Walk same index. Map `s[i]→t[i]`. Clash or `t` already taken → false. Not counts (#242). |
| HashMap + list (#380) | Map `val→index`. Insert append. Remove: swap with **last**, update that one index, drop tail. Random = `list.get(rand)`. Check = `containsKey`, not `list.contains`. |
| HashSet | One `for`. `contains` → yes. Else `add`. |
| HashSet (#36 board) | Skip `'.'`. Three keys: row / col / `box=(r/3)*3+(c/3)`. `add` fail → invalid. Don’t solve. |
| Stack (match) | Open → push. Close → pop and match. Leftover stack → false. |
| Running min | One `for`. Track `minSoFar`. Answer = best `x - minSoFar`. Not two pointers. |
| Two pointers (ends) | `while (left < right)`. Throw **one** side per step. Never reset a worker. |
| Sliding window | **Outer `for right` (grow). Inner `while` peels `left`.** `left` has no own `for`. |
| Prefix (#238) | Walk left→right storing left product. Walk right→left multiplying. |
| Kadane (#53) | `current = max(x, current+x)`. `best = max(best, current)`. |
| Write+read (#26) | Slow writes the next unique. Fast only scans. |
| Next greater | Stack of **indexes** waiting. New bigger → pop and fill `i - old`. |
| RPN | Number → push. Operator → pop two; **first pop is the right** side. |

**Week 1 so far:** HashMap (Two Sum, **#242 counts**), Stack (Parentheses), HashSet (Duplicate), Running min (#121).  
**Week 2:** Stack + min stack (#155). `top` = peek, not pop. Push min when `val <=` current min.  
**Week 2 Day 2:** two stacks as **in + out** (#232 Queue using Stacks). Pour only when out is empty. Amortized O(1). Cousin **#225** Stack using Queues (flip).  
**Week 2 Day 3:** stack as **next greater** (#739 Daily Temperatures). Stack of **indexes** still waiting. Hotter day pops and writes `answer[old] = i - old`. Not two pointers (worker that **resets** is O(n²)). Not #121 (one running min, one answer).  
**Week 2 Day 4:** stack as **unused numbers** (#150 Evaluate RPN). Operator takes last two; first pop is the **right** side. Not one running total. Not “one `*` multiplies the whole pile.” Cousin: infix / #394.  
**Week 3 Day 1:** two pointers from **ends**. **#167** Two Sum II — sorted pair; throw left if sum small, right if sum big; 1-based. Not #1 HashMap. Not index+worker (reset = O(n²)). **#125** Valid Palindrome — skip junk **one** side per step (`else if`); `while (left < right)`. Not stack.  
**Week 3 Day 2:** **#238** Product Except Self — prefix: left product walk forward, right product walk back; no division. **#26** Remove Duplicates from Sorted Array — two pointers write+read (`counter` writes uniques, `i` scans). Not ends. Not a worker.  
**Week 3 Day 3:** **#11** two ends, throw shorter wall. **#15** sort then two pointers (`left = i+1`).  
**Week 3 Day 4:** **#53** Kadane (current stretch vs best photo). **#88** two pointers from the tails.  
**Week 3 Day 5:** **#3** sliding window — peel `left`, do not reset the set. **#49** HashMap, key = sorted letters. Cover/Why never names the pattern.  
**Week 4 Day 1:** **#128** HashSet — start a value-run only if `x-1` missing; not sort; not window. **#424** sliding window — `length - maxFreq ≤ k`; peel left; not #3-on-repeat; not #128. Coding lists: Top 150 + Blind 75 only (#424 was last off-list). Part 1 Design: course **Chapter + topic**; explain then question.  
**Week 4 Day 2:** **#209** sliding window — outer `right`, inner peel `left`, `right-left+1`. **#219** HashMap value→last index; check then put. **Part 1 Design = Chapter 8** (timeout / retry / click id). Still Part 1, **not** Part 3. Timeout on **Booking’s** WebClient. GET may retry. Book only with click id in Event DB. Correlation id ≠ click id. **Anton 2026-09-08:** after he names the pattern, give the **Memorize this** template *before* he codes.  
**Week 4 Day 3:** **#36** HashSet three keys (go-over). **#205** HashMap glue both ways, not counts. **#380** list + map `val→index`, swap-with-last. Skipped **#76** Hard. **Part 1 Design = Chapter 3 + 5** (sync vs queue). Commit book → enqueue → 201. Don’t wait on Gmail; do wait on enqueue ack or outbox. Never lock across mail. Still Part 1, **not** Part 3. Outbox drill = **W6 Mon**.  
**Week 4 Day 4:** **#383** HashMap counts — spend magazine; extras allowed (`<=`, not `#242 ==`). **#73** HashSet poisoned rows/cols, then wipe; in place ≠ O(1); not sliding window. Deleted **#290**. **Part 1 Design = Chapter 12** circuit breaker. Open = **503** try later, never **201**. One dead pod ≠ open Event. Still Part 1, **not** Part 3 (W5 Wed).  
**Week 4 Day 5:** **#202** HashSet of seen n (or Floyd). Sum of squares of digits; cycle → not happy. Not window. Friday = coding only, no LC-SD.  
**Cadence (from Week 3 Day 1):** 2 LCs/weekday through Week 5; **3**/weekday Weeks 6–8. **Top Interview 150 + Blind 75 only.**  
**Do not** call #121 “two pointers.” Compare `Integer` with `intValue()`/`equals`, not `!=`. Do not pour back on every `pop` (#232). Do not write `answer[i]` or push temperatures (#739). **Never name the pattern / HashMap / window / two pointers before Anton names it in the gate** (not in Cover, not in Why). Two pointers **never** reset a worker. `else` binds to the nearest `if`.

*(Same list lives in `leetcode-practice/.cursor/rules/lc-pattern-recognition.mdc`.)*
