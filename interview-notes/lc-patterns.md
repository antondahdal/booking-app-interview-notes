# LC pattern recognition (every week)

Before coding, say in your own words:

1. **Pattern name**
2. **Why this one** (what state you keep while scanning)
3. **Why not** the 2 closest wrong patterns
4. **Time / extra space** target

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

**Week 1 so far:** HashMap (Two Sum, **#242 counts**), Stack (Parentheses), HashSet (Duplicate), Running min (#121).  
**Week 2:** Stack + min stack (#155). `top` = peek, not pop. Push min when `val <=` current min.  
**Week 2 Day 2:** two stacks as **in + out** (#232 Queue using Stacks). Pour only when out is empty. Amortized O(1). Cousin **#225** Stack using Queues (flip).  
**Week 2 Day 3:** stack as **next greater** (#739 Daily Temperatures). Stack of **indexes** still waiting. Hotter day pops and writes `answer[old] = i - old`. Not two pointers (worker that **resets** is O(n²)). Not #121 (one running min, one answer).  
**Week 2 Day 4:** stack as **unused numbers** (#150 Evaluate RPN). Operator takes last two; first pop is the **right** side. Not one running total. Not “one `*` multiplies the whole pile.” Cousin: infix / #394.  
**Week 3 Day 1:** two pointers from **ends**. **#167** Two Sum II — sorted pair; throw left if sum small, right if sum big; 1-based. Not #1 HashMap. Not index+worker (reset = O(n²)). **#125** Valid Palindrome — skip junk **one** side per step (`else if`); `while (left < right)`. Not stack.  
**Week 3 Day 2:** **#11 parked** (come back this week). **#238** Product Except Self — prefix: left product walk forward, right product walk back; no division. **#26** Remove Duplicates from Sorted Array — two pointers write+read (`counter` writes uniques, `i` scans). Not ends. Not a worker.  
**Cadence (from Week 3 Day 1):** 2 LCs/weekday through Week 5; **3**/weekday Weeks 6–8. Blind 75 / Grind 75 / NeetCode 150 only.  
**Do not** call #121 “two pointers.” Compare `Integer` with `intValue()`/`equals`, not `!=`. Do not pour back on every `pop` (#232). Do not write `answer[i]` or push temperatures (#739). Do not name the pattern family before Anton names it. Two pointers **never** reset a worker. `else` binds to the nearest `if`.

*(Same list lives in `leetcode-practice/.cursor/rules/lc-pattern-recognition.mdc`.)*
