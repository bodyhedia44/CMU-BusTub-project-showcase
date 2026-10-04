# Project 2 — B+ Tree Concurrent Index

> **Course:** [CMU 15-445/645 — Fall 2025](https://15445.courses.cs.cmu.edu/fall2025/project2/)
> **Component:** B+ Tree index layer of the BusTub RDBMS

---

## High-Level Objective

A database without indexes is a database that full-scans every query. The **B+ Tree** is the workhorse index structure of nearly every relational database — it provides $O(\log n)$ point lookups and efficient range scans by keeping all data in sorted order across linked leaf pages, with internal pages acting as a multi-level routing table.

This project required implementing a fully concurrent B+ Tree index that supports thread-safe insertion, deletion, point search, and in-order iteration. The implementation builds directly on top of the Project 1 Buffer Pool Manager — every B+ Tree page lives in a BPM frame, accessed exclusively through `ReadPageGuard` and `WritePageGuard`.

| Task | Component | Purpose |
|------|-----------|---------|
| 1 | **B+ Tree Pages** | Internal and leaf page layouts — key/value storage, split/merge primitives |
| 2 | **B+ Tree Operations** | Insert, Remove, GetValue — the core algorithms with split/merge/redistribute |
| 3 | **Index Iterator** | C++17-style iterator for in-order leaf scans via sibling pointers |
| 4 | **Concurrency Control** | Optimistic latch crabbing — thread-safe operations with minimal lock contention |

```
                   ┌──────────────────┐
                   │  Execution       │
                   │   Engine         │
                   └────────┬─────────┘
                            │  Insert() / Remove() / GetValue()
                   ┌────────▼─────────┐
                   │   B+ Tree Index  │ ◄── Optimistic Latch Crabbing
                   │                  │ ◄── Header Page (root_page_id)
                   │   ┌──────────┐   │
                   │   │ Internal │   │  ◄── Routing (key → child pointer)
                   │   │  Pages   │   │
                   │   └────┬─────┘   │
                   │   ┌────▼─────┐   │
                   │   │  Leaf    │──►│  ◄── Data (key → RID) + sibling ptrs
                   │   │  Pages   │   │
                   │   └──────────┘   │
                   └────────┬─────────┘
                            │  ReadPage() / WritePage()
                   ┌────────▼─────────┐
                   │  Buffer Pool     │
                   │   Manager (P1)   │
                   └──────────────────┘
```

---

## Task 1 — B+ Tree Pages

### What They Do

Every node in the B+ Tree is stored as a page in the buffer pool. There are two types:

- **Internal Pages**: Store $m$ keys and $m+1$ child pointers (page IDs). The first key (`key_array_[0]`) is **always invalid/unused** — lookups start from index 1. This is a critical invariant that caused one of the hardest bugs I encountered (see Concurrency Bug #2).
- **Leaf Pages**: Store $m$ keys and $m$ values (record IDs). Leaves are linked via a `next_page_id_` pointer for efficient range scans. They also include a tombstone buffer for recent deletions (simplified Bε-tree approach).

```
Internal Page Layout:
┌─────────┬────────┬─────────┬────────┬─────────┐
│ Value[0]│ Key[1] │ Value[1]│ Key[2] │ Value[2]│
│ (child) │(valid) │ (child) │(valid) │ (child) │
└─────────┴────────┴─────────┴────────┴─────────┘
  Key[0] = UNUSED GARBAGE — never read by InternalFindChild()

Leaf Page Layout:
┌────────┬────────┬────────┬────────┬──────────────┐
│ Key[0] │ Val[0] │ Key[1] │ Val[1] │ next_page_id │──►
└────────┴────────┴────────┴────────┴──────────────┘
```

### Implementation Details

- **InsertAt / RemoveAt**: Array-shifting operations — insert shifts elements right, remove shifts left. Used binary search (`InternalFindChild`, `LeafFindKey`) to find positions in $O(\log n)$ per page, as required by the project spec.
- **Split invariant**: Leaves split when `size == max_size` (after insertion); internals split when `size > max_size` (before insertion into parent). This follows the recommended approach from the project spec, preventing data overflow and single-child internal nodes.

---

## Task 2 — B+ Tree Operations

### Insert

1. Traverse from root to the correct leaf using `InternalFindChild()` at each level.
2. Insert the key-value pair into the leaf in sorted order.
3. If the leaf is full (`size == max_size`), **split**: move the upper half to a new leaf, push the middle key up to the parent.
4. If the parent overflows too, recursively split upward — potentially creating a new root.

### Remove

1. Traverse from root to the correct leaf.
2. Remove the key-value pair.
3. If the leaf **underflows** (`size < min_size`):
   - Try to **borrow** from a sibling (left first, then right).
   - If no sibling has surplus, **merge** with a sibling and remove the separator key from the parent.
4. If the parent underflows, recursively fix upward — potentially shrinking the tree height.

### GetValue (Point Search)

1. Traverse from root to leaf using comparator at each internal node.
2. Binary search the leaf for the key.
3. Return the associated value if found.

### Toughest Technical Challenges

**Recursive split propagation.** A leaf split pushes a key into the parent, which might overflow and need splitting itself, all the way up to the root. I implemented this using the `Context` class's `write_set_` deque — each level's page guard is pushed on the stack, and `split_Internal()` pops and recurses until the tree stabilizes or a new root is created.

**Balancing borrow vs. merge decisions.** During deletion, the order of checking siblings matters. I check borrow-from-left, borrow-from-right, merge-with-left, merge-with-right — in that priority. Borrowing is preferred because it doesn't cascade underflows upward.

---

## Task 3 — Index Iterator

The iterator enables C++17 for-each loops over the entire index in sorted order:

```cpp
for (auto &[key, value] : tree) {
    // process in sorted order
}
```

- **`begin()`**: Traverses to the leftmost leaf, positions at index 0.
- **`end()`**: Returns a sentinel iterator (invalid page ID).
- **`operator++()`**: Advances within the current leaf; when exhausted, follows `next_page_id_` to the next leaf.
- Holds a `ReadPageGuard` on the current leaf to ensure data consistency during iteration.

---

## Task 4 — Concurrency Control (Optimistic Latch Crabbing)

This is the core of the project and where all three concurrency bugs were discovered. The project spec states: *"You should use the optimistic latch coupling/crabbing technique described in class and in the textbook. The thread traversing the index should acquire latches on B+Tree pages as necessary to ensure safe concurrent operations, and should release latches on parent pages as soon as possible when it is safe to do so."*

### Pessimistic Latch Crabbing (Baseline)

The baseline approach acquires **write latches** from the header page all the way down to the leaf:

```
WriteLatch(header) → WriteLatch(root) → WriteLatch(internal) → WriteLatch(leaf)
```

At each level, if the child is "safe" (won't split/merge), release all ancestor latches. This is correct but slow — **every writer must first acquire the header page write latch**, serializing all concurrent writers at the top of the tree.

### Optimistic Latch Crabbing (My Implementation)

The key insight: most operations don't actually cause splits or merges. We can **speculatively traverse with cheap read latches** and only acquire an expensive write latch on the leaf:

```
ReadLatch(root) → ReadLatch(internal) → WriteLatch(leaf)
```

If the leaf is "safe" (has room for insert / has surplus for delete), we're done with a **single write latch** instead of write-latching the entire path. If unsafe, we fall back to pessimistic.

#### Optimistic Insert

```cpp
// 1. Load cached root (no header page access — eliminates contention)
page_id_t root_id = root_page_id_cache_.load(std::memory_order_relaxed);

// 2. Read-traverse to leaf (shared latches — concurrent readers allowed)
ReadPageGuard guard = bpm_->ReadPage(root_id);
while (!page->IsLeafPage()) {
    child_id = internal->ValueAt(InternalFindChild(internal, key));
    guard = bpm_->ReadPage(child_id);
}

// 3. Snapshot leaf state under read latch
int read_size = leaf->GetSize();
bool safe = (read_size < leaf->GetMaxSize() - 1);

// 4. If safe, upgrade to single write latch + re-verify
if (safe) {
    WritePageGuard write_guard = bpm_->WritePage(leaf_id);
    // Re-verify: size unchanged, next pointer unchanged, first key unchanged
    if (verification_passes) {
        w_leaf->InsertAt(idx, key, value);  // Done! Single write latch.
        return true;
    }
}
// 5. Fall through to pessimistic path
```

#### Optimistic Remove

Same pattern, but "safe" means `size > minSize` (won't underflow after removal):

```cpp
bool safe = (read_size > leaf->GetMinSize());  // surplus entries
```

#### The Re-verification Protocol

Between dropping the read latch and acquiring the write latch, another thread could modify the leaf (split it, merge it, etc.). The re-verification checks three invariants:

1. **Size unchanged**: `w_leaf->GetSize() == read_size`
2. **Next pointer unchanged**: `w_leaf->GetNextPageId() == read_next` (detects splits)
3. **First key unchanged**: `comparator_(w_leaf->KeyAt(0), read_first) == 0` (detects merges/redistributes)

If any check fails, the optimistic attempt is abandoned and we fall back to the safe pessimistic path.

---

## Concurrency Bug #1 — Livelock (Header Page Contention)

### Symptoms

`DeleteTest2` with 2 threads would **hang forever** — both threads alive, CPU at 100%, but zero progress. Not a deadlock (no circular wait), but a **livelock** (both threads repeatedly failing and retrying).

### Root Cause — Step by Step

The original optimistic path called `ReadPage(header_page)` to get the root page ID. The pessimistic path held `WritePage(header_page)` for its entire operation.

```
Time  Thread A (optimistic attempt)        Thread B (pessimistic, holding header)
────  ──────────────────────────────       ──────────────────────────────────────
 T1   ReadPage(header) → get root_id       WritePage(header) → traverse → delete
 T2   Read-traverse to leaf
 T3   Leaf not safe (size == minSize)
 T4   Fall back to pessimistic!
 T5   WritePage(header) → BLOCKED           Still working...
 T6                                         Finishes → releases header
 T7   Got header! Start pessimistic...      Starts new optimistic attempt
 T8   Traversing...                         ReadPage(header) → BLOCKED (A holds it)
 T9   Leaf not safe again!                  Still waiting...
T10   Release header, try optimistic
T11   ReadPage(header) → get root_id        Got header! Start pessimistic...
      ... cycle repeats forever ...
```

With `std::shared_mutex` on macOS (writer-preference), once Thread A requests the write latch on the header, Thread B's read latch request is blocked too. Both threads endlessly cycle between optimistic (hits unsafe leaf) → pessimistic (blocked on header) → optimistic → pessimistic...

### The Fix — Atomic Root Page ID Cache

```cpp
// Added to BPlusTree class:
std::atomic<page_id_t> root_page_id_cache_{INVALID_PAGE_ID};
```

The optimistic path reads the cache instead of touching the header page at all:

```cpp
// Optimistic: ZERO header page access
page_id_t root_id = root_page_id_cache_.load(std::memory_order_relaxed);

// Pessimistic: updates cache whenever it reads the actual header
root_page_id_cache_.store(ctx.root_page_id_, std::memory_order_relaxed);
```

Now optimistic and pessimistic paths **never contend** on the header page. The `relaxed` memory order is sufficient because the worst case (stale root ID) just causes the optimistic attempt to fail and fall back to pessimistic — no correctness issue, just a wasted traversal.

### Key Learnings

| Topic | What I Learned |
|-------|---------------|
| **Livelock vs. deadlock** | A deadlock is when threads wait forever on each other. A livelock is when threads are *active* but endlessly failing and retrying. Livelocks are harder to diagnose because the system appears busy. |
| **Writer-preference rwlocks** | On macOS, `std::shared_mutex` gives priority to writers. A pending write lock request blocks all new read lock requests. This makes the system fair for writers but amplifies contention when mixed read/write paths compete for the same lock. |
| **Lock-free fast paths** | An atomic variable can serve as a "fast path" cache to avoid acquiring any lock at all. The key insight: if the cached value might be stale, design the algorithm so staleness causes a benign retry, not corruption. |

---

## Concurrency Bug #2 — InsertAt(0) (Internal Page Key Corruption)

### Symptoms

`MixTest1` would fail with **"key not found"** errors — keys that were definitely inserted could not be retrieved by `GetValue()`. The tree's structure was silently corrupted.

### Root Cause — The Internal Page key[0] Invariant

Recall the internal page layout from the project spec: *"the first key in `key_array_` is set to be invalid, and lookups should always start from the second key."*

```
Index:    0        1        2        3
Value: [child0] [child1] [child2] [child3]
Key:   [UNUSED] [key1]   [key2]   [key3]
        ^^^^^
        key[0] is GARBAGE — never used in lookups
```

`InternalFindChild()` starts searching from index 1, intentionally skipping index 0. This means `key[0]` can contain **any garbage value** and the tree works correctly — as long as nobody moves it.

In `HandleInternalUnderflow`, when **borrowing from the left sibling**, the original code did:

```cpp
// BUGGY:
node->InsertAt(0, parent->KeyAt(child_idx), node->ValueAt(0));
node->SetValueAt(0, left_sib->ValueAt(last));
```

**What `InsertAt(0)` actually does:**

```
Before InsertAt(0):
  Key:   [garbage] [K1]    [K2]
  Value: [C0]      [C1]    [C2]

After InsertAt(0, separator, C0):
  Key:   [separator] [garbage] [K1]    [K2]     ◄── garbage is now at key[1]!
  Value: [C0]        [C0]      [C1]    [C2]
```

The shift operation moved everything right by one, promoting the garbage from `key[0]` to `key[1]` — the **first position that `InternalFindChild` actually reads**. Now traversal uses a garbage value as a routing decision, sending searches to the wrong subtree.

### The Fix

```cpp
// FIXED:
node->InsertAt(1, parent->KeyAt(child_idx), node->ValueAt(0));
node->SetValueAt(0, left_sib->ValueAt(last));
```

`InsertAt(1)` places the separator key at the first **valid** key slot and only shifts real keys. The garbage at `key[0]` stays in its harmless unused slot.

```
After InsertAt(1, separator, C0):
  Key:   [garbage] [separator] [K1]    [K2]     ◄── correct!
  Value: [C0_new]  [C0_old]    [C1]    [C2]
```

### Why It Only Appeared in Concurrent Tests

Sequential tests used small trees (3–5 keys) where internal nodes rarely underflowed, so borrow-from-left in internals almost never triggered. `MixTest1` with concurrent inserts and deletes across 10,000 keys created deeper trees with frequent internal redistributions, making this code path hot.

### Key Learnings

| Topic | What I Learned |
|-------|---------------|
| **Unused slots are contracts** | Internal page `key[0]` being unused is an *invariant*, not an implementation detail. Every operation that touches the array must preserve this invariant. A single violation corrupts all future traversals silently. |
| **Silent corruption is the worst bug** | Unlike crashes or assertion failures, data corruption manifests as wrong results that may not surface until much later. The tree *appeared* to work for small inputs — it took concurrent stress testing to expose the corruption. |
| **Array-shifting operations need diagrams** | When debugging `InsertAt`/`RemoveAt` on arrays with special-case indices, drawing out the before/after state on paper saved hours of staring at code. |

---

## Concurrency Bug #3 — TOCTOU Race in WritePageGuard::Drop()

### Symptoms

`DeleteTest1` and `MixTest1` would crash with:
```
Assertion failed: (frame_status.evictable_) && ("Remove called on a non-evictable frame")
```

This crash originated inside `ArcReplacer::Remove()`, called from `BufferPoolManager::DeletePage()`. The assertion means: "I'm trying to delete a page that's not evictable, but its pin count is 0 — this should be impossible."

### Root Cause — A Classic TOCTOU (Time-of-Check-Time-of-Use) Race

This bug involves two state variables — `pin_count` (atomic integer) and `evictable` (boolean in the ARC replacer) — that must always be consistent with each other but were updated **non-atomically**.

#### The Original Drop() Code:

```cpp
void WritePageGuard::Drop() {
    is_valid_ = false;
    frame_->is_dirty_ = true;
    frame_->rwlatch_.unlock();              // Step 1: release page latch

    // ⚠️ NO LOCK HELD HERE — DANGER ZONE ⚠️
    auto old_pin = frame_->pin_count_.fetch_sub(1);   // Step 2: atomic decrement
    if (old_pin == 1) {                                //         (pin count is now 0)
        std::lock_guard<std::mutex> lock(*bpm_latch_); // Step 3: acquire BPM lock
        replacer_->SetEvictable(frame_->frame_id_, true);  // Step 4: mark evictable
    }
}
```

#### The DeletePage() Code:

```cpp
auto BufferPoolManager::DeletePage(page_id_t page_id) -> bool {
    std::lock_guard<std::mutex> lock(*bpm_latch_);  // acquire BPM lock
    // ...
    if (frame->pin_count_.load() > 0) return false; // check pin count
    replacer_->Remove(frame_id);  // ASSERTS evictable_ == true
    // ...
}
```

#### The Race — Step by Step:

```
Time  Thread A (Drop)                       Thread B (DeletePage)
────  ─────────────────────────────────     ─────────────────────────────────
 T1   rwlatch_.unlock()
 T2   pin_count_.fetch_sub(1)
      → pin_count becomes 0
      → old_pin was 1
      → about to acquire bpm_latch_
      → BUT HASN'T YET!
                                      ┌─────────────────────────────────────
 T3                                   │ lock(bpm_latch_)  ← succeeds!
                                      │   (Thread A hasn't reached here yet)
 T4                                   │ pin_count_.load() → 0  ✓
                                      │   (Thread A already decremented it)
 T5                                   │ replacer_->Remove(frame_id)
                                      │   → checks evictable_[frame_id]
                                      │   → evictable_ is FALSE  ← 💥 CRASH
                                      │   → assert(evictable_) fails!
                                      └─────────────────────────────────────
 T6   lock(bpm_latch_)  ← BLOCKED
      (would have set evictable_ = true
       here, but Thread B already crashed)
```

#### Visual State Timeline:

```
pin_count:   1 ───────── 0 ────────────────────────── 0
evictable:  false ───── false ───── false ... true ── true
                          ↑                    ↑
                    fetch_sub(1)         SetEvictable(true)
                    [NO LOCK HELD]       [under bpm_latch_]
                          │
                   ┌──────┴──────┐
                   │   DANGER    │ ← DeletePage sees pin=0, evictable=false
                   │   WINDOW    │    → assertion crash
                   └─────────────┘
```

The fundamental problem: **two related state variables are updated in separate critical sections** (or no critical section at all). Between the `pin_count` decrement and the `SetEvictable` call, there's a window where the state is internally inconsistent. Any observer (like `DeletePage`) that checks both variables in this window sees an "impossible" state.

### The Fix — Atomic State Transition Under Lock

Move `pin_count_.fetch_sub(1)` **inside** the `bpm_latch_` critical section:

```cpp
void WritePageGuard::Drop() {
    if (!is_valid_) return;
    is_valid_ = false;
    frame_->is_dirty_ = true;
    frame_->rwlatch_.unlock();

    // ✅ BOTH state changes happen under the SAME lock
    std::lock_guard<std::mutex> lock(*bpm_latch_);
    auto old_pin = frame_->pin_count_.fetch_sub(1);
    if (old_pin == 1) {
        replacer_->SetEvictable(frame_->frame_id_, true);
    }
}
```

The same fix was applied to `ReadPageGuard::Drop()`:

```cpp
void ReadPageGuard::Drop() {
    if (!is_valid_) return;
    is_valid_ = false;
    frame_->rwlatch_.unlock_shared();

    // ✅ BOTH state changes happen under the SAME lock
    std::lock_guard<std::mutex> lock(*bpm_latch_);
    auto old_pin = frame_->pin_count_.fetch_sub(1);
    if (old_pin == 1) {
        replacer_->SetEvictable(frame_->frame_id_, true);
    }
}
```

#### Why This Completely Eliminates the Race:

After the fix, any thread holding `bpm_latch_` (including `DeletePage`) will observe **one of two states**:

| State | pin_count | evictable | Meaning |
|-------|-----------|-----------|---------|
| A | > 0 | false | Page still in use → `DeletePage` returns `false` |
| B | 0 | true | Page fully released → `DeletePage` proceeds safely |

The dangerous intermediate state (`pin_count == 0, evictable == false`) **can never be observed** because both transitions happen atomically under `bpm_latch_`.

#### The Trade-off:

The fix always acquires `bpm_latch_` in `Drop()`, even when `old_pin > 1` (pin count didn't reach 0). Previously, the lock was only acquired when pin count reached 0. This adds slight overhead per `Drop()` call, but it's necessary for correctness. In practice, the lock acquisition is fast (uncontended most of the time) and far cheaper than a crash.

### Key Learnings

| Topic | What I Learned |
|-------|---------------|
| **TOCTOU races** | Time-of-Check-Time-of-Use bugs occur when a condition checked at one point may change before it's relied upon. The classic pattern: "check something without a lock, act on it later" — the check can be stale by the time you act. |
| **Atomic ≠ thread-safe** | `pin_count_` being an `std::atomic` only means individual reads/writes are atomic. It does NOT mean that *sequences of operations* involving `pin_count` and other variables are atomic. Atomicity of individual variables doesn't imply atomicity of composed state. |
| **Related state needs unified synchronization** | When two variables form a logical invariant (`pin_count == 0 ↔ evictable == true`), all transitions must happen under the same lock. Updating them in separate critical sections creates windows where the invariant is violated. |
| **Race windows can be tiny but deadly** | The TOCTOU window in `Drop()` was just 2–3 instructions wide. Under normal testing it might never trigger. Under the aggressive concurrency of `MixTest1` (thousands of operations per second across multiple threads), it triggered reliably. |

---

## Additional Fix — Orphaned Page Stale Reads

### The Problem

After a leaf merge in the pessimistic path, the "dead" leaf is deleted from the BPM. But in the window between merge and deletion, an optimistic reader could:

1. Read-traverse to the dead leaf using a stale internal node pointer
2. Read stale keys that passed re-verification (the old data is still physically in memory)
3. Insert into a merged-away page → **data loss**

### The Fix

After merging, immediately poison the merged-from leaf by setting its size to 0:

```cpp
// In HandleLeafUnderflow, after merge-left:
MergeLeaf(left_sib, leaf);
leaf->SetSize(0);  // ← Poison for optimistic re-verification

// After merge-right:
MergeLeaf(leaf, right_sib);
right_sib->SetSize(0);  // ← Poison for optimistic re-verification
```

Optimistic re-verification checks `w_leaf->GetSize() == read_size`. Since the original `read_size > 0` but is now 0, re-verification **always fails** for poisoned pages, forcing a safe fallback to the pessimistic path.

---

## Files Modified

| File | Changes |
|------|---------|
| `src/storage/index/b_plus_tree.cpp` | Optimistic Insert/Remove paths, atomic root cache updates, `InsertAt(1)` fix, `SetSize(0)` orphan invalidation |
| `src/include/storage/index/b_plus_tree.h` | Added `std::atomic<page_id_t> root_page_id_cache_`, updated `RemoveDeadPage` signature, `dead_pages_` vector in Context |
| `src/storage/page/page_guard.cpp` | Fixed TOCTOU race in `ReadPageGuard::Drop()` and `WritePageGuard::Drop()` |

---

## Test Results

All 14 tests pass consistently (verified over 5 consecutive runs with no failures):

### Sequential Tests (8/8)

| Test | Status |
|------|--------|
| BasicInsertTest | ✅ PASSED |
| OptimisticInsertTest | ✅ PASSED |
| InsertTest1NoIterator | ✅ PASSED |
| InsertTest2 | ✅ PASSED |
| DeleteTestNoIterator | ✅ PASSED |
| OptimisticDeleteTest | ✅ PASSED |
| SequentialEdgeMixTest | ✅ PASSED |
| BasicScaleTest | ✅ PASSED |

### Concurrent Tests (6/6)

| Test | Status |
|------|--------|
| InsertTest1 | ✅ PASSED |
| InsertTest2 | ✅ PASSED |
| DeleteTest1 | ✅ PASSED |
| DeleteTest2 | ✅ PASSED |
| MixTest1 | ✅ PASSED |
| MixTest2 | ✅ PASSED |

---

## Benchmark Results (Leaderboard)

Configuration: 100K keys, BPM size 1024, 4 read threads + 2 write threads, 10-second duration.

| Metric | Throughput (ops/sec) |
|--------|---------------------|
| **Read** | **~90,969** |
| **Write** | **~36,033** |

The optimistic path delivers high read throughput because readers never contend on the header page (atomic cache) and only acquire cheap shared latches during traversal. Write throughput reflects the cost of pessimistic fallback for splits/merges, which is the expected trade-off of optimistic latch crabbing.

---

## Trade-offs

| Decision | Trade-off |
|----------|-----------|
| **Always acquire `bpm_latch_` in Drop()** | Fixes the TOCTOU race but adds lock overhead to every page guard destruction. Acceptable because the critical section is tiny (one atomic decrement + conditional SetEvictable call). |
| **Atomic root cache with relaxed ordering** | Zero-overhead reads but may return a stale root page ID. Staleness only causes a wasted optimistic traversal + safe fallback — never corruption. |
| **`SetSize(0)` poisoning on merged leaves** | Prevents optimistic readers from using orphaned pages, but requires matching changes in the re-verification protocol. An alternative would be epoch-based reclamation, which is more general but far more complex. |
| **Depth limit (100/20) in optimistic path** | Guards against infinite loops from stale cache, but might prematurely abort legitimate deep traversals. In practice, B+ Trees with 100K keys have depth ≤ 4, so limits of 20–100 are extremely generous. |

---

## Summary

This project taught me that concurrent data structures are fundamentally harder than their single-threaded counterparts — not because the algorithms are more complex, but because **every assumption about "what other threads are doing" must be explicitly verified**. The three bugs found here represent three distinct categories of concurrency failures:

1. **Livelock** (Bug #1) — threads making progress individually but starving each other globally. Fixed by eliminating the shared resource (header page) from the fast path.
2. **Silent data corruption** (Bug #2) — a violated invariant (`key[0]` is unused) that produced wrong results without any crash or assertion. Fixed by understanding the array layout contract.
3. **TOCTOU race** (Bug #3) — two related state variables updated non-atomically, creating a window of inconsistency. Fixed by unifying the state transition under a single lock.

Each bug reinforced the same lesson: **concurrency bugs don't announce themselves.** They hide behind nondeterminism, surface only under specific thread interleavings, and often manifest far from their root cause. The only reliable defense is rigorous reasoning about invariants + aggressive stress testing.

---

## How to Build Experience with Concurrency & Systems Debugging

### 1. Understand the Theory First

- Read **"The Little Book of Semaphores"** (free PDF) — builds intuition for synchronization patterns through progressively harder puzzles
- Read chapters 26–33 of **OSTEP** (Operating Systems: Three Easy Pieces, free online) — covers locks, condition variables, semaphores, and common concurrency bugs
- Know the classic bugs by name: **deadlock** (circular wait), **livelock** (threads make progress but undo each other's work), **priority inversion**, **ABA problem**, **thundering herd**, **convoy effect**

### 2. Practice Identifying Bug Patterns

The bugs I fixed required recognizing three distinct patterns:

| Pattern | How to Spot It | How I Spotted It Here |
|---------|---------------|----------------------|
| **Deadlock** | Two threads each hold a lock the other needs | Initial hypothesis — turned out wrong |
| **Livelock** | Threads are running but no useful work happens | Debug log showed `safe=0` every iteration, threads spinning forever |
| **Lock convoy** | Threads serialize on a hot lock, throughput collapses | `ReadPage(header)` in optimistic path contended with `WritePage(header)` in pessimistic path |

**Key skill:** Adding `fprintf(stderr, ...)` at lock acquire/release points and reading the interleaving. This is how most real-world concurrency bugs are diagnosed — not with fancy tools, but with timestamped logs showing which thread holds what.

### 3. Deliberate Practice Projects

**Beginner:**
- Implement a thread-safe queue with `std::mutex` + `std::condition_variable`
- Write a readers-writer lock from scratch
- Implement a thread pool

**Intermediate:**
- Build a concurrent hash map (start with coarse-grained lock, then striped locks, then lock-free)
- Implement the dining philosophers with different strategies (resource ordering, Chandy/Misra)
- Do CMU 15-445's Buffer Pool Manager project

**Advanced:**
- Implement lock-free data structures (Michael-Scott queue, Harris linked list)
- Build a simple database transaction manager with 2PL
- Read and modify real concurrent code (RocksDB, LevelDB, SQLite)

### 4. Debugging Methodology

The process I followed in this project is repeatable for any concurrency bug:

1. **Reproduce** — find a test that fails reliably (or run it in a loop until it does)
2. **Isolate** — binary-search for the cause by disabling/enabling components until you find the minimal reproduction
3. **Instrument** — add `fprintf(stderr, ...)` at critical points (lock acquire/release, state transitions)
4. **Reason about interleavings** — for every pair of consecutive lines, ask: "What can another thread do between these two lines?"
5. **Fix and verify** — make the minimal fix, run the failing test 5+ times to confirm

### 5. Tools to Learn

| Tool | What For |
|------|---------|
| **ThreadSanitizer** (`-fsanitize=thread`) | Detects data races automatically |
| **Helgrind** (Valgrind) | Detects lock ordering violations |
| **GDB/LLDB** `thread apply all bt` | See where all threads are stuck during a deadlock |
| **`perf lock`** (Linux) | Shows lock contention hotspots |
| **`fprintf` to stderr** | The simplest and often most effective — what I used here |

### 6. Conceptual Framework

Every concurrency bug comes down to one question:

> **"What can another thread do between these two lines?"**

Train yourself to look at any two consecutive lines and imagine an adversarial scheduler inserting another thread's operations between them. In the TOCTOU bug:

```cpp
auto old_pin = frame_->pin_count_.fetch_sub(1);  // pin_count is now 0
// ← What can another thread do HERE? Answer: DeletePage, which sees pin=0 but evictable=false → crash
if (old_pin == 1) {
    std::lock_guard<std::mutex> lock(*bpm_latch_);
    replacer_->SetEvictable(frame_->frame_id_, true);
}
```

This "what-if" thinking is the core skill. Practice it by code-reviewing concurrent code and trying to find bugs before running it.

### 7. Recommended Reading Order

1. **OSTEP concurrency chapters** (free) — foundations
2. **"C++ Concurrency in Action"** by Anthony Williams — C++ specific patterns
3. **"The Art of Multiprocessor Programming"** by Herlihy & Shavit — advanced, lock-free algorithms
4. **CMU 15-445 lecture notes** on concurrency control — database-specific (B+ tree latching, 2PL)

The biggest takeaway from this debugging session: **concurrency bugs are rarely about code being "wrong" — they're about assumptions being violated by interleavings you didn't consider.** The fix is always to either eliminate the shared resource (atomic cache) or make the critical section atomic (move pin_count decrement under `bpm_latch_`).
