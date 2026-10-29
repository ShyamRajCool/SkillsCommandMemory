# ECE 309: Final Project — Skills, Memory, and Command Policy
**Weight:** 45% of project grade | **Duration:** about 5 weeks | **Deadline:** Thursday, December 3, 11:59 PM
**Expected effort:** about 30 hours in total, roughly an hour a day. There is a suggested schedule in §10 that follows the lecture schedule.

> **Start here.**
> 1. Copy your eight P2/P3 files into the starter (§7.1), or configure with `-DUSE_P2_REFERENCE=ON -DUSE_P3_REFERENCE=ON` to use the instructor's versions. Until you do one of these, CMake stops with a "Missing ..." message.
> 2. Build and run the provided tests (§8, "Build & run"). The P2 and P3 tests must pass before you start; the 8 example tests fail until you write your code.
> 3. Work through Parts 1–3 in the order of the schedule (§10). Each part lists exactly what to write, and §12 lists the most common mistakes.

## 1. Project Overview & Learning Goals

Your harness can now hold a conversation (P2) and let the model use tools (P3). The final project adds the three things a real agent needs next:

* **Memory.** The model can remember facts across sessions and find them again later. When a conversation gets too long for the model's context window, the harness decides which messages to drop.
* **Command policy.** The model can ask to run a shell command. Before anything happens, the request passes through a chain of permission rules. Each rule decides *Allow*, *Deny*, or *Ask* (ask you at the terminal first). *Commands are simulated: nothing is ever executed.*
* **Skills.** A skill is a folder of instructions the model can load on demand. Skills can require other skills, so loading one means loading its requirements first, in the right order.

Each feature is built on data structures from lecture, and those data structures are what you write:

```text
                 ┌────────────────────────────────────────────────────────────┐
                 │                          Harness                           │
   user input ──►│   run loop + tool loop (provided)                          │
                 │        │                                                   │
                 │        ├── before every model call: select_context ★ YOU   │
                 │        │      (token budget, MinHeap ★ YOU)                │
                 │        ▼                                                   │
                 │   ToolRegistry (your P3) ──► tools (provided)              │
                 │        │                                                   │
                 │        ├── remember / recall / forget / list_memories      │
                 │        │      MemoryStore ──► MemoryIndex BST ★ YOU         │
                 │        │                      top_k (MinHeap)  ★ YOU        │
                 │        ├── list_skills / load_skill / skill_dependents     │
                 │        │      SkillCatalog ──► DepGraph DFS/BFS ★ YOU       │
                 │        │                       SkillIndex      ★ YOU        │
                 │        │                       LruCache        ★ YOU        │
                 │        └── run_command                                     │
                 │               PolicyChain + 4 policies         ★ YOU        │
                 └────────────────────────────────────────────────────────────┘
```

**Learning goals:**
* Build a **binary heap** and use it for top-k selection in O(n log k) instead of a full sort.
* Build an **LRU cache** from a hash map and a hand-written **doubly linked list**, with O(1) operations.
* Build a **binary search tree** with all three removal cases, and see for yourself why an unbalanced tree can degenerate.
* Use **inheritance and polymorphism** for a family of policies, and the **composite pattern** to combine them.
* Use a **queue** for a sliding-window rate limit.
* Traverse a **directed graph**: depth-first search for a dependency order with cycle detection, breadth-first search for "what depends on this?"
* Build an **inverted index** and measure how much faster it is than scanning every document.

**What's provided vs. what's yours.** As in P3, everything except the data structures is given to you as working code. You implement:

| # | You implement | File(s) | Course topic (week) | Points |
|---|---|---|---|---|
| 1 | `MinHeap<T>` and `top_k()` | `include/util/min_heap.h` | heaps (11) | 9 |
| 2 | `LruCache<K,V>` | `include/util/lru_cache.h` | doubly linked lists, hash tables (6, 9) | 14 |
| 3 | `MemoryIndex`, a binary search tree | `memory_index.h` (private part) + `src/memory/memory_index.cpp` | BSTs (10–11) | 14 |
| 4 | `select_context()` | `src/memory/context_builder.cpp` | heaps (11) | 6 |
| 5 | four `Policy` classes + `PolicyChain` | `src/policy/policies.cpp`, `src/policy/policy_chain.cpp` | inheritance, polymorphism, queues (6, 8–9) | 12 |
| 6 | `DepGraph`: DFS load order, BFS dependents | `dep_graph.h` (private part) + `src/skills/dep_graph.cpp` | graphs, DFS, BFS, topological sort (12, 14) | 12 |
| 7 | `SkillIndex`, an inverted index | `src/skills/skill_index.cpp` | hash tables (9) | 5 |

That is 72 points. The other 28 are integration (8), your test suite (10) and your design log (10), for 100 in total, plus up to 5 bonus points. §11 has the full rubric.

Everything else is provided, including:
* the memory store (save/load to a file), the skill catalog (reads `SKILL.md` files), and the audit log
* all eight new tools, including the simulated `run_command`
* the context budget (in the harness) and the terminal confirmation prompt (in `main.cpp`)
* `main.cpp` with the new command-line flags, a sample skill library in `skills/`, and four demo scripts

**STL rule.** You may use the STL anywhere **except** to replace the data structure you are being graded on. §7.3 lists exactly what is allowed and forbidden in each file.

> **Platform requirement.** As before, build and test on Linux (department machines, WSL2, or a Linux VM). The grader runs on Linux with AddressSanitizer. macOS mostly works, but leak detection is unavailable there, so run your final check on Linux.

> **AI use.** The course AI policy applies (see the syllabus). You may use AI tools to explain concepts and compiler errors and to help you debug code **you wrote**. You may **not** use them to generate the graded data structures listed above. Your design log must say which tools you used and for what. You may be asked to explain or change any part of your code in person (§8).

---

## 2. The Big Picture: Three Short Sessions

The demo scripts in `scripts/` drive the provided `ScriptedModelClient` exactly as in P3. These are the outputs from the reference solution. Long tool results are cut off with `...` in the `[tool ...]` lines, just as in P3. The excerpts show what you see on a terminal, including what you type. In captured output, typed input is not echoed, so `you> ` and the next `assistant> ` share a line.

The **e2e tests** run these sessions and compare your output with the expected output in `tests/e2e/expected/` (the "golden" files). They blank out the text after `-> error]`, so the wording of your policies' reasons is up to you. The exception is `load_skill` errors: the DepGraph messages (`cycle: ...`, `missing: ...`) are specified exactly, so the e2e tests compare them too.

**Memory** (`./build/miniharness --script scripts/memory_demo.script`):

```text
you> remember three things
assistant> I'll remember those.<|tool_call|>{"id":"m1","name":"remember",...}<|/tool_call|>...
[tool remember m1 -> ok] remembered m1
[tool remember m2 -> ok] remembered m2
[tool remember m3 -> ok] remembered m3
assistant> Saved three memories.
you> when is the demo?
assistant> Let me check.<|tool_call|>{"id":"r1","name":"recall","arguments":{"query":"when is the demo","k":2}}<|/tool_call|>
[tool recall r1 -> ok] m3: The demo needs a laptop with WSL\nm1: Project demo is on...
```

`remember` inserts into **your BST**. `recall` scores every memory by the number of distinct query words it shares, and keeps the best k with **your `top_k`**. Ties go to the **newer** memory, which is why m3 comes before m1 above. The provided `MemoryStore` applies that rule through its comparator; your `top_k` just uses it. `forget` is a **BST remove**, and `list_memories` is an **in-order traversal**.

**Skills** (`--script scripts/skills_demo.script`):

```text
[tool list_skills s1 -> ok] csv-report: Turn a CSV of sales rows into a formatted summar...
[tool load_skill s2 -> ok] Loaded 4 skills (dependencies first): read-csv, number-forma...
[tool skill_dependents s3 -> ok] depend on number-format: csv-report table-format
[tool load_skill s4 -> error] cycle: draft-intro -> draft-outline -> draft-intro
[tool load_skill s5 -> error] missing: changelog-format (required by release-notes)
```

`list_skills` searches **your inverted index**. `load_skill` asks **your DFS** for a load order and reads each skill body through **your LRU cache**. `skill_dependents` is **your BFS**. Three sample skills are broken on purpose so you can see the error messages: `draft-intro` and `draft-outline` require each other (a cycle), and `release-notes` requires a skill that doesn't exist.

**Policy** (`--script scripts/policy_demo.script`):

```text
[tool run_command p1 -> ok] [simulated] $ ls -l\n(exit 0; commands are simulated, nothin...
[tool run_command p2 -> error] denied by allowlist: 'rm' is not on the allowlist
[tool run_command p3 -> error] denied by patterns: command contains forbidden pattern ";"
[confirm] Run `git status`? (y/n) y
[confirm] Run `make clean`? (y/n) n
[tool run_command p4 -> ok] [simulated] $ git status\n(exit 0; commands are simulated, n...
[tool run_command p5 -> error] denied by confirm: the user declined
...
[tool run_command p6 -> ok] [simulated] $ echo hi\n(exit 0; commands are simulated, noth...
[tool run_command p7 -> ok] [simulated] $ wc -l notes.txt\n(exit 0; commands are simulat...
[tool run_command p8 -> error] denied by ratelimit: rate limit: 5 commands per 60.000000 s
```

Every request goes through **your `PolicyChain`**. `git` and `make` return *Ask*, so the harness asks you on the terminal (both prompts come before the two results, because the dispatcher runs every call in the reply first). Each final decision is written to the audit log. The last three commands show the rate limit (5 per minute, the last policy in the chain). Five requests have reached it by then: p1 `ls`, p4 `git`, p5 `make` (the rate limit allowed it before you declined) and p6 `echo`, p7 `wc`. p2 and p3 were denied earlier in the chain and never reached it. p8 is the sixth, so it is denied.

**Context budget** (`--script scripts/context_demo.script --context-budget 1450 --keep-recent 2`; the flags are in §3):

```text
you> [context: sent 7 of 8 messages, 1446 tokens]
```

Before each model call, **your `select_context`** chooses which messages fit in 1450 tokens. The provided harness marks every successful `load_skill` result as *pinned*, so skill instructions are never dropped.

---

## 3. Program Behaviors & Requirements

The provided code implements all of these. They work once your data structures are correct.

**New command-line flags** (all optional):

| Flag | Meaning |
|---|---|
| `--skills DIR` | skill library to scan (default: `./skills` if it exists) |
| `--memory FILE` | load memories from FILE and save after every change (default: memories last only for this run) |
| `--audit FILE` | append every `run_command` decision to FILE |
| `--context-budget N` | send the model at most about N tokens (0 = send everything, the default) |
| `--keep-recent N` | the N newest messages are never dropped (default 4) |

**New tools:**

| Tool | Arguments | Result | Uses your |
|---|---|---|---|
| `remember` | `text` | `remembered m3` | BST insert |
| `recall` | `query`, optional `k` (default 3) | `m3: ...` lines, best first | `top_k`, in-order traversal |
| `forget` | `id` such as `"m3"` | `forgot m3` | BST remove |
| `list_memories` | — | every memory, oldest first | in-order traversal |
| `list_skills` | optional `query`, optional `k` (default 5) | `name: description` lines. Without a query: every skill, sorted by name (`k` is still checked, but otherwise ignored) | `SkillIndex::search` |
| `load_skill` | `name` | bodies, dependencies first | `DepGraph::order_from`, `LruCache` |
| `skill_dependents` | `name` | `depend on X: a b c` | `DepGraph::dependents` |
| `run_command` | `program`, optional `args` (array of strings) | `[simulated] $ ...` or `denied by ...` | `PolicyChain` and the policies |

`k` must be a whole number from 1 to 100; anything else is an error result.

**Required behaviors:**

1. **Earlier behavior is preserved.** All P2 and P3 tests and e2e sessions still pass. `--no-tools` still behaves exactly like P2.
2. **Commands are never executed.** `run_command` only reports what *would* run.
3. **Ask means ask.** If the chain's final decision is Ask, the harness prints `` [confirm] Run `git status`? (y/n) `` and reads your answer. `y`, `Y` or `yes` allows the command; anything else denies it.
4. **Audit before action.** The final decision (allow or deny) is written to the audit log *before* the simulated command "runs".
5. **The transcript keeps everything.** With a context budget, the model sees a trimmed copy, but `--save` still writes the full conversation.
6. **Skill bodies are read lazily.** Scanning reads only the front matter of each `SKILL.md` (the header between its two `---` lines; see §6.1). A body is read from disk the first time it is loaded, and after that it comes from the LRU cache.

---

## 4. Part 1 — Memory (weeks 9–11)

Do Part 1 first. Heaps are covered in week 11, and `top_k` is reused in Parts 1 and 3.

### 4.1 `MinHeap<T, Less>` and `top_k` — `include/util/min_heap.h`

#### What it is
A binary **min-heap** stored in an array: the smallest element (by `less`) is always at index 0. Element `i` has children `2i+1` and `2i+2`, and its parent is `(i-1)/2`. Appendix A walks through push and pop step by step.

`Less` works like the comparator you pass to `std::sort`: `less(a, b)` is true when `a` should come first. With the default `std::less<T>` you get a min-heap. With `std::greater<T>` you get a max-heap.

#### Function by function

| Function | Behavior | Cost |
|---|---|---|
| `push(value)` | append at the end, then **sift up** (swap with the parent while smaller) | O(log n) |
| `top()` | the smallest element; throws `std::out_of_range` if empty | O(1) |
| `pop()` | save the root, move the last element to the root, shrink, **sift down** (swap with the smaller child while larger), return the saved root; throws `std::out_of_range` if empty | O(log n) |
| `size()`, `empty()`, `clear()` | provided | O(1) |

**`top_k(items, k, less)`** returns the **k largest** items, **largest first**.
* `k == 0` or no items: empty result.
* `k >= items.size()`: every item, largest first.

It must keep a `MinHeap` of **at most k** items: the k largest seen so far, with the *smallest* of them on top. Two details trip people up. First, build the heap **with** the comparator, `MinHeap<T, Less> heap(less);`, and compare only with `less(a, b)`, never `<`: some callers pass a lambda, and some item types have no `operator<`. Second, return early when `k == 0`, before anything touches `heap.top()`. For each item:
* if the heap holds fewer than k items, push the item;
* else if the heap's top is less than the item, pop the top and push the item;
* otherwise skip it.

At the end, pop everything (smallest first) and reverse. That is O(n log k) time and O(k) extra space. The hidden tests count comparisons, so a full sort fails them.

**Example.** `top_k({5, 1, 9, 3, 7}, 2)` returns `{9, 7}`. The heap holds `[5]`, then `[1,5]`, then 9 replaces 1 to give `[5,9]`, then 3 is skipped, then 7 replaces 5 to give `[7,9]`.

#### Rules
* Write sift-up and sift-down yourself. `std::vector` is fine as the storage array, and `std::swap` and `std::reverse` are fine.
* No library heap, sorting or sorted container (`std::priority_queue`, `std::make_heap`, `std::sort`, `std::set`, ...). §7.3 has the full list.

### 4.2 `LruCache<Key, Value>` — `include/util/lru_cache.h`

#### What it is
A cache that holds at most `capacity` entries. When it is full and a new key arrives, it throws out the **least recently used** entry. Both `get` and `put` must be **O(1)**, so the cache combines two structures that point to the same nodes:

```text
 map_  (std::unordered_map<Key, Node*>):   "a" ──► node A    "b" ──► node B    "c" ──► node C

 doubly linked list (YOU):     head_ ──► [A] ⇄ [C] ⇄ [B] ◄── tail_
                                       most recent        least recent (evicted next)
```

The map finds a key's node in O(1). The list keeps recency order. Moving a node to the front, or removing the tail, is a constant number of pointer updates. Appendix B lists every pointer update.

#### Function by function

| Function | Behavior |
|---|---|
| `LruCache(capacity)` | empty cache. Capacity 0 is legal: it never stores anything |
| `get(key)` | **hit:** move the node to the front, count a hit, return a pointer to its value. **Miss:** count a miss, return `nullptr` |
| `put(key, value)` | **key present:** replace the value and move it to the front (no eviction, no hit or miss counted). **Key absent:** if full, remove the tail node (count one eviction), then insert the new node at the front. Capacity 0: do nothing |
| `contains(key)` | true if cached; changes **neither** recency **nor** stats |
| `erase(key)` | remove if present (not an eviction); returns whether it was present |
| `clear()` | free every node; stats are kept |
| `stats()` | provided: returns `{hits, misses, evictions}`; you update `stats_` in `get` and `put` |
| `keys_mru_to_lru()` | provided; walks the list from `head_` (the tests use it to check your order) |
| `~LruCache()` | free every node |

`size()` and `capacity()` are provided. The pointer returned by `get()` stays valid until the next `put`, `erase` or `clear`. The LRU cache is in Part 1 because linked lists and hash tables come early in the course, but its only user is Part 3. The provided `SkillCatalog` caches skill bodies with your `LruCache<std::string, std::string>`.

#### Rules
* Write the doubly linked list yourself, with `new`/`delete` only inside this class. Forbidden: `std::list`, `std::deque`, `std::forward_list` (don't even include their headers), or a `std::vector` you shift around. §7.3 has the full list.
* `std::unordered_map` is expected for the key-to-node index.
* No leaks and no double frees: ASan checks every test, including those that erase the head, the middle, and the tail.

### 4.3 `MemoryIndex`, a binary search tree — `include/memory/memory_index.h`, `src/memory/memory_index.cpp`

#### What it is
The provided `MemoryStore` keeps every memory in your BST, keyed by its numeric `id`. `remember` assigns ids 1, 2, 3, … and calls `insert`. `forget` calls `remove`. `list_memories` and `recall` walk the tree with `for_each_in_order`.

#### Function by function

| Function | Behavior |
|---|---|
| `insert(rec)` | add it; returns `false` (and changes nothing) if the id is already present |
| `find(id)` | pointer to the record, or `nullptr`; valid until the next insert/remove/clear |
| `remove(id)` | `false` if absent. Otherwise handle all three cases: **leaf** (just delete it), **one child** (splice the child into its place), **two children** (copy in the in-order successor *or* predecessor, then delete that node) |
| `size()` | provided: returns `count_`, which you keep up to date |
| `height()` | nodes on the longest root-to-leaf path: 0 for empty, 1 for a single node |
| `for_each_in_order(fn)` | call `fn` on every record in ascending id order |
| `clear()`, `~MemoryIndex()` | free every node |

Appendix C draws the three removal cases.

#### Design requirements
* Write the tree with node pointers. Choose your own private members (the starter suggests a layout).
* Forbidden in these two files: `std::map`, `std::set`, `std::unordered_map`, `std::unordered_set`, and any other associative container. Do not keep the records in a sorted `std::vector` either (§7.3).
* **Do not balance the tree** for the core grade. Ids arrive in increasing order, so your tree will degenerate into a linked list (height = n). It must still work with 5,000 sequential ids. In your design log you will measure this and explain the cause. Fixing it with AVL balancing is the bonus (§9).
* Recursion is fine at the sizes the tests use: at 5,000 nodes deep, a recursive `insert` or destructor still fits on the stack. Much deeper unbalanced trees (tens of thousands of sequential ids) can overflow the stack under ASan, so an iterative `insert` is a nice touch.

### 4.4 `select_context()` — `src/memory/context_builder.cpp`

#### What it is
Before each model call, the harness describes every message as a `ContextItem {role, tokens, pinned}` and asks `select_context(items, budget, keep_recent)` which ones to send. You return the kept indices (ascending), their total tokens, and whether you are still over budget.

#### Exact behavior
1. **Protected items are never dropped:** every `Role::System` item, every `pinned` item, and the last `keep_recent` items (all of them if `keep_recent >= size`).
2. If the total of **all** items is `<= budget`, keep everything.
3. Otherwise, drop unprotected items one at a time, in this order:
   * lowest `drop_rank(role)` first (provided in `context_builder.h`): Tool (0), then Assistant (1), then User (2);
   * among equal ranks, the **oldest** (smallest index) first.

   Stop as soon as the total is `<= budget`.
4. If every unprotected item is gone and the total is still too big, keep the protected items anyway. `over_budget` is true exactly when `total_tokens > budget`.

**Required:** pick the next item to drop with **your `MinHeap`** of `(drop_rank, index)` pairs (`std::pair` compares the first element, then the second). Do not sort.

**Example** (`keep_recent = 2`, `budget = 50`, ten tokens each):

| index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| role | System | User | Assistant | Tool | Assistant | Tool | User | Assistant |
| | protected | kept | dropped 3rd | dropped 1st | kept | dropped 2nd | protected (recent) | protected (recent) |

The total is 80, so three items must go. The tools go first (3, then 5). Then the oldest assistant reply, 2. That leaves 50, which fits, so the result is `keep = {0, 1, 4, 6, 7}`.

---

## 5. Part 2 — Command Policy (weeks 6, 8–9)

### 5.1 Provided types — `include/policy/policy.h`
* `CommandRequest {program, args, tool_call_id}`, plus `command_line()`, which gives `"git status"`.
* `Decision {kind, reason, policy}`. `Kind` is ordered **Allow < Ask < Deny**, so "most restrictive" means the larger kind. The helpers `Decision::allow(why, who)`, `ask(...)` and `deny(...)` build one. `reason` must never be empty, and `policy` is the `name()` of the policy that decided.
* `Policy`, the abstract base: `name()` and `evaluate(req)`. `evaluate` is not `const`, because a rate limit has to remember past requests.
* `Clock` with two implementations: `SteadyClock` (real time) and `ManualClock` (for tests; time moves only when you call `advance()`).
* `Confirmer` answers Ask decisions. The terminal version is in `main.cpp`. For tests, `ScriptedConfirmer` is in `tests/test_helpers.h`.

### 5.2 The four policies — `src/policy/policies.cpp`

| Class | `name()` | Decision |
|---|---|---|
| `AllowlistPolicy(programs)` | `allowlist` | **Allow** if `req.program` exactly matches one of `programs`, else **Deny** |
| `ArgumentPatternPolicy(patterns)` | `patterns` | **Deny** if `req.command_line()` contains any non-empty pattern as a substring, else **Allow**. `";"` catches `cat a; rm b`, and `"rm -rf"` catches `{"rm", {"-rf", "/"}}` |
| `ConfirmPolicy(programs)` | `confirm` | **Ask** if `req.program` is listed, else **Allow** |
| `RateLimitPolicy(max, window, clock)` | `ratelimit` | **sliding window:** at time t, count the earlier requests that **this policy** allowed (recorded) at times s with `t - s < window`, whatever happened to them later in the chain or at the confirm prompt. If there are already `max`, **Deny** and do not record this one. Otherwise record t and **Allow**. `max == 0` denies everything. A request exactly `window` seconds after an allowed one no longer counts it: with a 10 s window, a request at t = 110 ignores one at t = 100 |

`RateLimitPolicy` must keep its timestamps in a **queue** (`std::queue<double>` is fine). Pop timestamps from the front once they leave the window, so memory stays O(max). `tracked()` (provided) reports how many it holds.

The public interfaces in `policies.h` are fixed. The private members shown are suggestions: you may change them.

### 5.3 `PolicyChain` — the composite pattern — `src/policy/policy_chain.cpp`

A `PolicyChain` **is a** `Policy` that **has** child policies, so chains can nest, and the harness never knows whether it holds one rule or forty.

| Function | Behavior |
|---|---|
| `add(child)` | append; returns `false` (and adds nothing) for `nullptr` |
| `size()` | provided: number of direct children |
| `evaluate(req)` | **empty chain:** Deny with policy `"chain"` (deny by default). Otherwise evaluate children **in order**. On the **first Deny, stop immediately** (later children are not evaluated) and return that child's `Decision` unchanged. Otherwise, if any child said Ask, return the **first** Ask unchanged. Otherwise return Allow with policy `"chain"` and a non-empty reason |

"Unchanged" means a Deny from a chain nested inside another chain still names the **leaf** policy (`"allowlist"`, not `"chain"`).

### 5.4 How `run_command` uses it (provided)

`main.cpp` builds this default chain. Order matters: the rate limit goes last, so a request already denied by an earlier policy never uses up the rate budget. An Ask request does use a slot, even if you then decline it.

```cpp
ArgumentPatternPolicy({";", "|", "&", "`", "$(", ">", "<", "rm -rf", "sudo"})
AllowlistPolicy({"ls", "cat", "echo", "grep", "wc", "head", "tail", "git", "make", "g++", "python3"})
ConfirmPolicy({"git", "make", "g++", "python3"})
RateLimitPolicy(5, 60.0, clock)
```

`RunCommandTool::invoke` evaluates the chain and resolves Ask with the confirmer. It writes the final decision to the audit log and only then returns `[simulated] $ ...`. Read `src/tools/agent_tools.cpp` to see the whole flow.

---

## 6. Part 3 — Skills (weeks 12–14)

Graphs, DFS and BFS are covered in week 12, and directed graphs and topological sort in week 14. Do this part last.

### 6.1 Skills and the catalog (provided)

A skill is a folder containing `SKILL.md`:

```text
---
name: csv-report
description: Turn a CSV of sales rows into a formatted summary report.
requires: [read-csv, table-format, number-format]
---
# CSV report
1. Load the rows (read-csv).
...
```

`SkillCatalog::scan()` reads only the front matter of every `skills/*/SKILL.md`. It sorts the skills by name, so a skill's position is its `doc_id`. Then it builds your `DepGraph` (one node per skill, one edge per `requires` entry) and your `SkillIndex` (name plus description). Malformed skills are reported and skipped.

### 6.2 `DepGraph` — `include/skills/dep_graph.h`, `src/skills/dep_graph.cpp`

An edge `from -> to` means "**from requires to**": `to` must be loaded before `from`. Any STL container is fine for adjacency lists and marks. The traversals are yours.

| Function | Behavior |
|---|---|
| `add_node(name)` | `false` if it already exists |
| `add_edge(from, to)` | `false` (adds nothing) if `from` is not a node or the edge already exists. `to` does **not** have to exist yet |
| `has_node(name)` | whether `name` is a node |
| `node_count()` | number of nodes |
| `requirements(name)` | its requirements, in the order the edges were added; empty for an unknown name |
| `order_from(root, &out, &error)` | **DFS** (below) |
| `dependents(name)` | **BFS** (below) |

**`order_from`: depth-first search with three colors** (Appendix D). Visit `root`'s requirements in edge order, recursively. A node is **white** (not visited yet), **gray** (on the current DFS path) or **black** (finished). Append a node to `out` when it turns black. That is the DFS **post-order**: every skill comes after everything it requires, each skill appears exactly once, and `root` is last.

When the DFS follows an edge `y -> x`, it checks in this order:
1. `x` is not a node: fail with `missing: <x> (required by <y>)`.
2. `x` is gray: it is on the current path, so this is a **cycle**. Fail with `cycle: ` followed by the path from `x` to the current node, then `x` again, joined by ` -> `.
3. `x` is white: visit it.
4. `x` is black: skip it. It is already finished, and this is *not* a cycle; diamonds reach the same node twice.

If `root` itself is not a node, fail with `unknown: <root>`. Report the **first** problem the DFS meets. On success return `true` and fill `*out`. On failure return `false` and set `*error`; `*out` may contain anything. Error strings must match exactly (the hidden tests and the `skills_demo` e2e session both check them):

| Graph (`a>b` = a requires b) | `order_from("a")` |
|---|---|
| `a>b, b>c` | `out = {c, b, a}` |
| `a>b, a>c, b>d, c>d` (diamond) | `out = {d, b, c, a}` |
| `a>a` | `cycle: a -> a` |
| `a>b, b>c, c>a` | `cycle: a -> b -> c -> a` |
| `a>b, b>c, c>b` | `cycle: b -> c -> b` |
| `a>b, b>gone` | `missing: gone (required by b)` |

**`dependents(name)`: breadth-first search on the reversed edges.** Every skill that requires `name` directly or indirectly ("what breaks if I delete this skill?"), nearest first: sorted by BFS distance, and alphabetically within the same distance. Do not include `name` itself. It must work when `name` is not a node (a missing requirement) and when the graph has cycles; use a visited set. Use a queue (`std::queue` is fine) or process the graph level by level.

### 6.3 `SkillIndex`, an inverted index — `src/skills/skill_index.cpp`

An **inverted index** maps each word to the list of documents containing it (its *posting list*). A search then looks only at the posting lists of the query's words, not at every document. Appendix E has a worked example.

| Function | Behavior |
|---|---|
| `add(doc_id, text)` | for every distinct word in `unique_tokens(text)`, append `doc_id` to that word's posting list (once per document) |
| `postings(word)` | the posting list for a lowercase word, or `nullptr` |
| `word_count()` | number of distinct words indexed |
| `search(query, k)` | score = number of **distinct** query words the document contains. Return up to k doc ids with score ≥ 1, highest score first, ties broken by **smaller doc_id** first. Empty query, no matches, or `k == 0`: empty |

Rank with **your `top_k`**, not a sort. `std::unordered_map` for the postings and the per-document scores is expected. The provided `linear_search()` computes the same ranking by re-reading every document; it is your baseline for the timing table in your design log.

---

## 7. Putting It Together

### 7.1 Carrying P2 and P3 forward

Copy eight files from your P3 repository into the starter:

| From P3 | To the final project |
|---|---|
| `include/core/conversation.h`, `include/core/sentinel_scanner.h` | `include/core/` |
| `src/core/conversation.cpp`, `src/core/sentinel_scanner.cpp` | `src/core/` |
| `include/tools/tool_registry.h` | `include/tools/` |
| `src/tools/tool_registry.cpp`, `src/tools/workspace.cpp`, `src/tools/calculator_eval.cpp` | `src/tools/` |

Do not copy `message.h`, `tool.h`, `builtin_tools.*`, `harness.*` or `main.cpp`. The starter's versions are newer.

**If your P2 or P3 code didn't work,** configure with `-DUSE_P2_REFERENCE=ON` and/or `-DUSE_P3_REFERENCE=ON` to use the instructor's versions. You lose no final-project points for this, but say so at the top of your design log.

Your P3 registry must hold **12** tools now: the four from P3 plus the eight new ones. That is well within any correct hash table.

### 7.2 Where your code is called

| Provided file | Calls your |
|---|---|
| `src/memory/memory_store.cpp` | `MemoryIndex` (all of it), `top_k` |
| `src/harness/harness.cpp` | `select_context` |
| `src/skills/skill_catalog.cpp` | `DepGraph`, `SkillIndex`, `LruCache` |
| `src/tools/agent_tools.cpp` | everything above through the store and catalog, plus `PolicyChain` via `ctx.policy` |
| `src/main.cpp` | constructs the four policies and the chain |

### 7.3 What you may and may not use

This table is the complete list of forbidden shortcuts. The grader scans for them, and the per-part "Rules" above summarize it. The "Allowed" column gives examples, not a complete list: anything not forbidden is fine (`std::pair`, `std::less`, `std::function`, ...).

| File | Allowed (examples) | Forbidden (that row scores 0) |
|---|---|---|
| `min_heap.h` | `std::vector`, `std::swap`, `std::reverse`, `std::pair` | `priority_queue`, `make_heap`/`push_heap`/`pop_heap`/`sort_heap`, `sort`, `stable_sort`, `partial_sort`, `nth_element`, `set`/`multiset`, `map`/`multimap` |
| `lru_cache.h` | `std::unordered_map` | `std::list`, `std::deque`, `std::forward_list` (even just including their headers); a `std::vector` you shift around to keep recency order |
| `memory_index.*` | `std::function`, `std::vector` for a traversal stack | `map`, `set`, `unordered_map`, `unordered_set` (and multi- versions, or their headers); a sorted `std::vector` of records |
| `context_builder.cpp` | your `MinHeap`, `std::vector` | `sort`, `stable_sort`, `partial_sort`, `nth_element`, `priority_queue`, `make_heap`, `set`/`multiset`, `map`/`multimap` |
| `policies.cpp`, `policy_chain.cpp` | any STL | — |
| `dep_graph.*` | any STL container (`unordered_map`, `vector`, `queue`, ...) and `std::sort` for BFS ties | graph libraries |
| `skill_index.cpp` | `std::unordered_map`, `std::vector`, your `top_k` | `sort`, `stable_sort`, `partial_sort`, `nth_element`, `priority_queue`, `make_heap` |

---

## 8. Deliverables & Testing

The repository must build with CMake, compile with **zero warnings** under `-Wall -Wextra -Wpedantic` (`-DMH_WERROR=ON` checks this), and run clean under AddressSanitizer + UndefinedBehaviorSanitizer (on by default).

**Repository structure** (★ = you write or modify; ◆ = copied from your P2/P3; everything else is provided):

```text
├── CMakeLists.txt · README.md · Final_spec.md
├── include/
│   ├── core/       message.h · conversation.h ◆ · sentinel_scanner.h ◆
│   ├── tools/      tool.h · tool_registry.h ◆ · workspace.h · agent_tools.h · ...
│   ├── util/       min_heap.h ★ · lru_cache.h ★ · text.h · number_format.h
│   ├── memory/     memory_index.h ★ (private section) · context_builder.h · memory_store.h
│   ├── policy/     policy.h · policies.h ★ (private sections, optional) · policy_chain.h · audit_log.h
│   ├── skills/     dep_graph.h ★ (private section) · skill_index.h ★ (private section, optional) · skill_catalog.h
│   └── harness/, json/, model/   (as in P3)
├── src/
│   ├── core/       conversation.cpp ◆ · sentinel_scanner.cpp ◆
│   ├── tools/      tool_registry.cpp ◆ · workspace.cpp ◆ · calculator_eval.cpp ◆ · agent_tools.cpp · ...
│   ├── memory/     memory_index.cpp ★ · context_builder.cpp ★ · memory_store.cpp
│   ├── policy/     policies.cpp ★ · policy_chain.cpp ★ · audit_log.cpp
│   ├── skills/     dep_graph.cpp ★ · skill_index.cpp ★ · skill_catalog.cpp · skill_index_linear.cpp
│   ├── util/       text.cpp
│   └── harness/, json/, model/, main.cpp
├── reference/p2/, reference/p3/   instructor references (only with -DUSE_P2/P3_REFERENCE=ON)
├── skills/         sample skill library (three skills are broken on purpose)
├── scripts/        demo sessions: memory_demo, skills_demo, policy_demo, context_demo, + P3's
├── tests/
│   ├── mini_test.h · test_helpers.h   (new helpers: FixedPolicy, ScriptedConfirmer, write_skill)
│   ├── p2/test_p2.cpp · p3/test_loop.cpp   (provided regression tests)
│   ├── final/test_final.cpp ★          your suite
│   └── e2e/                             golden sessions, run by ctest
└── docs/design-log-final.md ★
```

**Build & run** (copy your P2 and P3 files in first, §7.1):

```bash
cmake -S . -B build && cmake --build build -j
cd build && ctest --output-on-failure && cd ..        # p2, p3 tool loop, your tests, e2e
./build/test_final lru_                                # only your tests starting with lru_
./build/miniharness --script scripts/skills_demo.script
./build/miniharness --script scripts/memory_demo.script --memory my_memories.txt
```

**Your test suite (`tests/final/test_final.cpp`):** at least **20** test cases using `mini_test.h`, named `<area>_<what>` with the areas `heap`, `lru`, `bst`, `ctx`, `pol`, `graph`, `index`. The starter gives you **8 examples**, and they count toward the 20:

| Part | Given examples |
|---|---|
| Memory (4) | `heap_pops_in_order`, `lru_evicts_least_recently_used`, `bst_remove_two_children`, `ctx_spec_example` |
| Policy (2) | `pol_rate_limit_window_boundary`, `pol_chain_first_deny_stops_first_ask_wins` |
| Skills (2) | `graph_order_and_cycle_message`, `index_ranking_and_ties` |

Write **at least 12 more**. Read the examples first: they show how to build each structure, how to use `ManualClock` and `FixedPolicy`, and how to compare vectors with `CHECK_EQ` (it prints both sides on failure). If a sanitizer error aborts the run, rerun just that test with `./build/test_final =test_name` (the `=` means "exactly this name"). One test may cover several of the items below. Together, the examples and your tests must cover:

1. **Heap:** pops come out in order, including duplicates; `top()`/`pop()` on an empty heap throw; `top_k` with `k = 0`, `k > n`, and a check against a full sort on random data. **Print** the time for `top_k` with k = 10 and for `std::sort` on 100,000 random numbers.
2. **LRU:** the eviction order; `get` refreshes recency; `put` on an existing key updates without evicting; capacity 0 and capacity 1; `erase` of the head, a middle node, and the tail (check `keys_mru_to_lru()` after each); the hit, miss and eviction counts.
3. **BST:** all three `remove` cases, including removing the root; a duplicate insert is rejected; in-order is sorted; the `height()` of a shape you draw by hand. **Print** the height after inserting ids 1…1000 in order and after inserting them shuffled.
4. **Context:** system, pinned and recent items survive; tool results go first, then the oldest; `over_budget` is set correctly.
5. **Policy:** every policy's allow / deny / ask cases; the rate limit across a window boundary using `ManualClock`; chain precedence; short-circuit on Deny (use `FixedPolicy`'s `calls` counter); a nested chain.
6. **Graph:** a chain, a diamond (each skill exactly once), a self-loop, a three-node cycle (exact message), a missing requirement (exact message), and `dependents` ordering.
7. **Index:** postings once per document; ranking and tie-breaking. **Print** the average time per query for your `search` and for `linear_search` on 100, 1,000 and 5,000 generated documents (use `std::chrono::steady_clock`).

**Design log (`docs/design-log-final.md`):** 700–1,000 words in Markdown. A template is in the starter. Cover:
* **At the top:** whether you used your own P2/P3 or the references, and which tools (including AI assistants) you used and for what.
* **LRU cache:** list every pointer change in (a) a `get` hit on a middle node and (b) a `put` that evicts. Use them to argue that both operations are O(1).
* **Heap:** why `top_k` is O(n log k), not O(n log n). Include one measurement against a full sort.
* **BST:** your two printed heights. Why do sequential ids make the tree degenerate, and what does that do to `find`? What would fix it?
* **Graph:** why DFS needs three colors. Give a small graph where a two-color version (visited / not visited) would report a cycle that isn't there.
* **Index:** your timing table (index vs. linear scan at 100 / 1,000 / 5,000 documents) and what the trend shows.
* **Threat model:** in 3–5 sentences, what the default policy chain defends against and what it does *not*. Hint: `python3` is on the allowlist.
* **One thing you would design differently** in hindsight.

**Oral check-in.** You may be asked to explain any part of your code, or to change a small piece of it on the spot (for example, "make the cache evict the *most* recently used entry"). "The AI wrote it" is not a valid answer.

**Submission.** Upload both files to the Moodle assignment before the deadline:
* **github.txt:** the URL of your GitHub repository. Make sure the course staff can open it.
* **github.zip:** a ZIP of the same repository. Leave out `build/`.

The two must contain the same code. Either one is enough for grading if the other has a problem.

---

## 9. Bonus (up to +5)

Choose **one**. It is graded only if the core project builds and passes its own tests.

* **AVL balancing (recommended).** Make `MemoryIndex` an AVL tree: store heights, rotate after insert and remove, keep every public behavior the same. Add tests showing the height stays below about 1.44 log₂ n (for example, ≤ 20 for 10,000 nodes) after 10,000 sequential inserts and after random removes. Put before/after heights and timings in your design log. The hidden BST tests accept any valid height, so an AVL tree passes them unchanged.
* **Performance study.** Generate a 10,000-message session and time `select_context`, `recall` with 10,000 memories, and `list_skills` with 5,000 skills. Find the slowest step, improve it, and report the before and after numbers with your method. A well-argued "I tried X and it did not help" earns full credit.

---

## 10. Suggested Schedule

About six hours a week. Each part starts after its topic is covered in lecture.

| Week | Lecture | Goal | Checkpoint |
|---|---|---|---|
| Oct 27 – Nov 2 | BST remove, AVL, heaps | read §1–4; port P2/P3; `MinHeap` + `top_k`; `LruCache` | everything builds; the P2/P3 tests and the heap and LRU example tests pass |
| Nov 3 – Nov 9 | graphs, DFS, BFS | `MemoryIndex` (all three removes); `select_context` | the `memory_demo` e2e session passes; your BST and context tests pass |
| Nov 10 – Nov 16 | review, **Exam 2** | lighter week: four policies + `PolicyChain` | the `policy_demo` e2e session passes; your policy tests pass |
| Nov 17 – Nov 23 | directed graphs, topological sort | `DepGraph` (DFS order, errors, BFS); `SkillIndex` | `skills_demo` and `context_demo` (which loads a skill) pass; all 11 e2e sessions pass (7 from P3 + 4 new) |
| Nov 24 – Dec 3 | Thanksgiving | finish your 20+ tests; measurements; design log; final review | zero warnings, ASan clean, submission files ready |

---

## 11. Grading Rubric

| Criterion | Points | Evaluation |
|---|---|---|
| **`MinHeap` + `top_k`** | 9 | Hidden tests: ordering, duplicates, custom comparators, empty-heap exceptions, `top_k` edge cases, agreement with a full sort, and a comparison count that rules out sorting. |
| **`LruCache`** | 14 | Hidden tests: hits and misses, eviction order, update-in-place, capacity 0 and 1, erase anywhere, `clear`, a 20,000-operation comparison against a reference model, a 300,000-operation workload that must finish quickly (O(1)), and keys and values of different types. |
| **`MemoryIndex` (BST)** | 14 | Hidden tests: insert/find, duplicates, in-order, height, all three removal cases, emptying the tree through the root, 6,000 random operations checked against `std::map`, 5,000 sequential ids, and `MemoryStore` end to end. |
| **`select_context`** | 6 | Hidden tests: the §4.4 rules, 500 random cases against a reference, and the harness sending the trimmed conversation. |
| **Policies + `PolicyChain`** | 12 | Hidden tests: every policy, the sliding-window boundary, bounded memory, all 27 three-child combinations, short-circuiting, nesting, and `run_command` with the audit log. |
| **`DepGraph`** | 12 | Hidden tests: exact orders and error strings, diamonds, cycles not through the root, a 5,000-node chain, random DAGs, `dependents` ordering with cycles and missing nodes, and the catalog's `load_skill` (bodies read lazily through your `LruCache`). |
| **`SkillIndex`** | 5 | Hidden tests: postings, ranking, ties, empty cases, and agreement with `linear_search` on 3,000 documents. |
| **Integration** | 8 | 2 points: earlier projects still work (P2 tests, P3 tool-loop tests and the 7 P3 e2e sessions). 6 points: the 4 new e2e sessions. |
| **Test suite quality** | 10 | Manual review against §8 items 1–7. |
| **Design log** | 10 | Quality of the arguments, measurements, and threat model. |
| **Bonus** | +5 | §9. |

Every automated test runs in its own process under ASan + UBSan; a crash, a leak or a sanitizer report fails that test. If one of your parts does not compile or link, the grader replaces **that part** with the starter stub and grades the rest. That part scores 0, and so may parts built on it: `select_context` and `SkillIndex` use your heap. **Always submit code that compiles**, even if some functions are still stubs. A build with warnings loses up to 5 points. Using a forbidden library shortcut (§7.3) scores 0 for that row.

---

## 12. Common Pitfalls

1. **Heap: comparing with the wrong child.** In sift-down, swap with the *smaller* of the two children, and check that the right child exists (`2i+2 < size`).
2. **`top_k`: comparing the wrong way.** Replace the heap's top only when the top is *less than* the new item. Reversing that keeps the k smallest.
3. **LRU: forgetting `tail_`.** When you unlink the last node, `tail_` must move to its `prev`. When you unlink the only node, both `head_` and `tail_` become `nullptr`.
4. **LRU: deleting the node before erasing its key from the map.** Read `victim->key` first, then `delete victim`. ASan reports the reverse order as heap-use-after-free.
5. **BST: two-children removal that loses a subtree.** The successor can still have a *right* child. Splice that child into the successor's old place.
6. **BST: wrong `count_`.** Change the count exactly once per successful insert or remove, and never on a failed one.
7. **DFS: two colors instead of three.** A diamond (`a>b, a>c, b>d, c>d`) reaches `d` twice. That is not a cycle; only an edge to a *gray* node is.
8. **DFS: pre-order instead of post-order.** Append a node *after* visiting its requirements, not before.
9. **Rate limit: recording denied requests.** Record a timestamp only when the rate limit itself allows the request.
10. **Chain: returning the last Ask.** Return the *first* Ask, and stop at the first Deny.
11. **`map[key]` to look something up.** On a miss, `operator[]` *inserts* the key. In the LRU this silently grows `size()` and crashes later. In a `const` function it doesn't even compile ("discards qualifiers"). Use `find(key)` and compare with `end()`.
12. **Unsigned subtraction.** `items.size() - keep_recent` wraps around to a huge number when `keep_recent` is bigger. Check first.
13. **`if (s.find(p))`.** `find` returns `std::string::npos` when nothing is found, not 0. Compare with `npos`.
14. **Copying a `unique_ptr`.** `push_back(child)` and `for (auto child : children_)` both try to copy. Use `std::move(child)` and `for (const auto& child : children_)`.
15. **DFS path not popped.** Pop a node off your path when it turns black. Otherwise a cycle message includes finished nodes: for `a>b, a>c, c>a` the answer is `cycle: a -> c -> a`, with no `b`.
16. **BFS sorted by name only.** The order is by distance first, then by name. Sort by (distance, name), or sort each level before appending it. Sorting the whole result by name loses the nearest-first order.
17. **`Node*` as a return type outside the class.** In the `.cpp`, write `MemoryIndex::Node* MemoryIndex::helper(...)`. Plain `Node*` gives "'Node' does not name a type".
18. **A string_view key.** `postings_.find(word)` with a `std::string_view` does not compile in C++17. Use `find(std::string(word))`.

---

## Appendix A — Primer: Binary Heaps

A heap is a complete binary tree stored level by level in an array. For index `i`: left child `2i+1`, right child `2i+2`, parent `(i-1)/2`. **Min-heap rule:** every node is ≤ its children, so the minimum is at index 0.

**push(2)** into `[1, 3, 5, 4, 6]`:

```text
append:     [1, 3, 5, 4, 6, 2]      2 is at index 5, parent (5-1)/2 = 2 holds 5
2 < 5 swap: [1, 3, 2, 4, 6, 5]      2 is at index 2, parent 0 holds 1
2 > 1 stop
```

**pop()** from `[1, 3, 2, 4, 6, 5]`:

```text
save 1; move last to root:  [5, 3, 2, 4, 6]
children of 0 are 3 and 2; smaller is 2 < 5, swap:  [2, 3, 5, 4, 6]
5 is at index 2, children 5 and 6 don't exist: stop.  Return 1.
```

Each step moves one level, so both operations are O(log n).

**top-k with a size-k min-heap.** To keep the 3 largest of a stream, hold the best 3 so far with the *worst of them* on top. A new item only enters if it beats that worst one, which costs O(log k). Every item costs at most O(log k), so the total is O(n log k). For n = 100,000 and k = 10, the reference `top_k` takes about 0.1 ms, and `std::sort` of the whole array takes about 6 ms.

## Appendix B — Primer: LRU Cache Pointer Updates

Nodes have `prev` and `next`. `head_` is the most recent and `tail_` is the least recent.

**`get` hit on a middle node X** (list `H ⇄ P ⇄ X ⇄ N ⇄ ... ⇄ T`):

```text
unlink X:        P->next = N;  N->prev = P
push X to front: X->prev = nullptr;  X->next = H;  H->prev = X;  head_ = X
```

That is six pointer writes no matter how long the list is, plus one O(1) hash lookup to find X.

**`put` that evicts** (cache full, new key K):

```text
victim = tail_;  tail_ = victim->prev;  tail_->next = nullptr
map_.erase(victim->key);  delete victim;  ++evictions
node = new Node{K, V};  node->next = head_;  head_->prev = node;  head_ = node
map_[K] = node
```

Edge cases to handle: unlinking the head (`head_` moves), unlinking the tail (`tail_` moves), the only node (both become `nullptr`), and capacity 1.

## Appendix C — Primer: BST Removal

```text
         50                    remove 20 (leaf): just delete it
       /    \                  then remove 30 (now one child, 40): 50's left becomes 40
     30      70                remove 70 (two children):
    /  \    /  \                  successor = leftmost node of 70's right subtree = 80
  20   40  60   80                 copy 80's record into 70's node, then delete the
                                   old 80 node (its right child, if any, takes its place)
```

If 80 had a left child 75, then 75 would be the successor instead. Using the **predecessor** (the rightmost node of the left subtree) is equally correct. Either way, the in-order sequence of the remaining ids is unchanged.

**Why sequential ids hurt.** Inserting 1, 2, 3, …, n makes every new node the right child of the previous one. The "tree" is a linked list of height n, so `find` is O(n). Inserting the same ids shuffled gives a height near 2–3 × log₂ n. With 10,000 ids, the reference measures height 10,000 vs. 29, and 10,000 `find`s take 142 ms vs. 1 ms. An AVL tree (lecture, week 11) keeps the height below about 1.44 log₂ n.

## Appendix D — Primer: DFS with Three Colors, and BFS

**Three colors.** White = not visited yet. Gray = visit started and still on the current path. Black = finished. An edge to a **gray** node means you have come back to an ancestor: a cycle. An edge to a **black** node is fine. In the diamond `a>b, a>c, b>d, c>d`, the DFS finishes `d` under `b`. Later `c` reaches `d` again, but `d` is black, not gray.

**Trace of `order_from("a")` on the diamond:**

```text
visit a (gray)                          path: a
  visit b (gray)                        path: a b
    visit d (gray) → no edges → black   out: d
  b → black                             out: d b
  visit c (gray)                        path: a c
    d is black → skip
  c → black                             out: d b c
a → black                               out: d b c a
```

**Reporting a cycle.** Keep the current path (for example, a `std::vector<std::string>` you push and pop). When an edge reaches a gray node x, the cycle is the part of the path from x to the end, followed by x again: path `a b c`, edge `c -> b`, gives `cycle: b -> c -> b`.

**BFS for dependents.** Build or keep the reversed edges (`to -> from`). Start a queue at `name` with distance 0, and give each newly seen node the distance of its parent + 1. Use a visited set, so cycles terminate. Finally, sort by (distance, name).

## Appendix E — Primer: Inverted Index

```text
doc 0: "format numbers"           format  → [0, 2]
doc 1: "csv report table numbers" numbers → [0, 1]
doc 2: "table format"             csv     → [1]
                                  report  → [1]
                                  table   → [1, 2]
```

Query `"csv table numbers"`: walk the posting lists of `csv`, `table` and `numbers` only, adding 1 per hit. Doc 1 scores 3, docs 0 and 2 score 1 each, so the result is `[1, 0, 2]`. `top_k` itself does not break ties (equal items may come out in any order). To make the smaller id win, rank `(score, doc_id)` pairs with a comparator that calls a pair "less" when its score is lower **or** its score is equal and its id is **larger**.

A linear scan re-reads every document on every query: its cost grows with the whole collection. The index's cost grows only with the lengths of the query words' posting lists. With 5,000 documents, the reference index answers a two-word query about 1,500× faster than `linear_search` (0.004 ms vs. 7.3 ms).

## Appendix F — Script and Transcript Formats

Unchanged from P3 (P3 spec, Appendix D). The new demo scripts in `scripts/` are good examples of multi-call assistant blocks.
