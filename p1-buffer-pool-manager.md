# Project 1 — Buffer Pool Manager

> **Course:** [CMU 15-445/645 — Fall 2025](https://15445.courses.cs.cmu.edu/fall2025/project1/)
> **Component:** Memory management layer of the BusTub RDBMS

---

## High-Level Objective

A database can't keep all its data in memory — that's the whole reason a buffer pool exists. The **Buffer Pool Manager (BPM)** is the component that mediates between the execution engine (which thinks in terms of pages) and the disk (which actually stores them). It decides *which* pages live in memory, *when* to fetch new ones, and *which* to evict when space runs out.

This project required implementing three tightly coupled components:

| Task | Component | Purpose |
|------|-----------|---------|
| 1 | **ARC Replacer** | Eviction policy — decides which page to kick out of memory |
| 2 | **Disk Scheduler** | Asynchronous disk I/O — schedules reads/writes via a thread pool |
| 3 | **Buffer Pool Manager** | The orchestrator — ties replacer + disk I/O + page table together |

```
                      ┌───────────────┐
                      │  Execution    │
                      │   Engine      │
                      └──────┬────────┘
                             │  ReadPage() / WritePage()
                      ┌──────▼────────┐
                      │  Buffer Pool  │ ◄── Page Table (page_id → frame_id)
                      │   Manager     │ ◄── Free List
                      │               │ ◄── ARC Replacer
                      └──────┬────────┘
                             │  Schedule({DiskRequest, ...})
                      ┌──────▼────────┐
                      │    Disk       │ ◄── Thread Pool (4 workers)
                      │  Scheduler    │ ◄── Per-page mutex map
                      └──────┬────────┘
                             │
                      ┌──────▼────────┐
                      │  Disk Manager │
                      │  (filesystem) │
                      └───────────────┘
```

---

## Task 1 — ARC Replacer (Adaptive Replacement Cache)

### What It Does

Standard LRU evicts the *least recently used* page, but it has no way to distinguish between pages accessed once (e.g., a sequential scan) and pages accessed frequently (a hot index page). **ARC** solves this by maintaining *two* active lists and *two* ghost lists that together self-tune the balance between recency and frequency:

- **MRU list** — pages seen recently (accessed once since entering the cache)
- **MFU list** — pages seen frequently (accessed more than once)
- **MRU ghost** — metadata of recently evicted MRU pages (no data, just page IDs)
- **MFU ghost** — metadata of recently evicted MFU pages

When a ghost list gets a "hit" (a page that was evicted comes back), ARC adjusts a target size parameter `p` to shift the balance: more MRU ghost hits → grow the MRU allocation, more MFU ghost hits → shrink it. This makes ARC *scan-resistant* and *self-tuning* — no manual parameter like the "K" in LRU-K.

### Architecture & Design

- **Four doubly-linked lists** (`std::list<page_id_t>`) for MRU, MFU, MRU ghost, and MFU ghost.
- **`page_map_`** (`std::unordered_map<page_id_t, FrameStatus>`) — the central lookup: maps a page ID to its `FrameStatus` struct containing frame ID, evictability, current ARC status, and a **stored list iterator** for O(1) erasure from whichever list it lives in.
- **`frame_to_page_`** (`std::unordered_map<frame_id_t, page_id_t>`) — reverse mapping from frame to page, needed because the BPM identifies frames by frame ID but the ghost lists track by page ID.
- **Target size `mru_target_size_`** (the `p` from the ARC paper) — dynamically adjusted on ghost hits using the delta formula from the original paper.
- Protected by a single `std::mutex` — critical sections are short O(1) operations.

### Toughest Technical Challenges

**Unsigned overflow in target size adjustment.** The ARC algorithm adjusts `mru_target_size_` (parameter `p`) by subtracting a delta on MFU ghost hits. Since `mru_target_size_` is `size_t` (unsigned), a naive `mru_target_size_ -= delta` wraps to a massive value instead of going negative. I had to guard it with `mru_target_size_ = (mru_target_size_ > delta) ? (mru_target_size_ - delta) : 0` — a subtle bug that caused the replacer to massively over-allocate to MRU, destroying the frequency-based eviction half of ARC.

**Stored iterator for O(1) list operations.** Each `FrameStatus` stores a `std::list<page_id_t>::iterator` pointing to its position in whichever list it currently belongs to. This allows O(1) erasure instead of linear scans. The tricky part: every time a page moves between lists (MRU → MFU, active → ghost, etc.), the iterator must be updated atomically with the list mutation. Forgetting to update it in any code path causes use-after-free on the next access.

**Ghost list capacity management.** The ARC paper specifies when to evict entries from the ghost lists themselves (they can't grow unboundedly). Implementing the two conditions — `|MRU| + |MRU ghost| == c` and `|MRU| + |MRU ghost| + |MFU| + |MFU ghost| == 2c` — required careful ordering: evict from the ghost list *before* inserting the new page, and clean up `page_map_` entries for killed ghosts.

### Key Learnings

| Topic | What I Learned |
|-------|---------------|
| **Pointers & ownership** | Gained a deeper understanding of when to use raw pointers vs. references vs. smart pointers. The `page_map_` owns `FrameStatus` by value; the lists store only page IDs; iterators are non-owning handles. Clear ownership prevents leaks and dangling references. |
| **Unsigned overflow** | C++ unsigned integer underflow wraps silently — no exception, no crash, just wrong results. The `mru_target_size_` subtraction bug taught me to *always* guard unsigned subtraction with a lower-bound check. |
| **Stored iterators for O(1) access** | Keeping a `std::list::iterator` inside the struct enables O(1) erasure from any list, but every list mutation must update the corresponding iterator. This is a powerful pattern but demands discipline across all code paths (evict, record access, remove, ghost eviction). |
| **Lock types** | A single `std::mutex` is sufficient here because all operations are O(1). Using `std::shared_mutex` would add overhead without benefit since there's no "read-only" path — even `Size()` could use a simple atomic, but the mutex keeps the implementation straightforward. |

---

## Task 2 — Disk Scheduler

### What It Does

The disk scheduler decouples page I/O from the thread that requested it. Instead of blocking the caller while a page is read from disk, the BPM enqueues a `DiskRequest` and gets back a `std::future<bool>` that it can wait on. A pool of background worker threads dequeues requests and performs the actual reads/writes.

### Architecture & Design

- **Thread pool:** Spawned **4 worker threads** at construction, each running `StartWorkerThread()` in a loop.
- **Request queue:** A thread-safe channel (`Channel<std::optional<DiskRequest>>`) handles producer-consumer communication. Workers block on `Get()` until a request arrives.
- **Graceful shutdown:** The destructor pushes one `std::nullopt` sentinel per worker, signaling each thread to exit, then `join()`s all of them.
- **Per-page locking:** To prevent concurrent reads and writes to the *same page* from corrupting data, I introduced a `std::unordered_map<page_id_t, std::shared_ptr<std::mutex>>` guarded by a separate `map_mutex_`. Each worker acquires the per-page lock before touching the disk manager, allowing parallel I/O to *different* pages while serializing access to the *same* page.

### Toughest Technical Challenges

**Promise lifecycle.** A `DiskRequest` owns a `std::promise<bool>`. If the promise is destroyed without being fulfilled (e.g., due to an exception or early return), the corresponding future throws `std::future_error` — a frustrating crash to debug. I had to ensure every code path calls `callback_.set_value(true)` exactly once, including edge cases.

**Parallel I/O correctness.** With 4 workers, two threads could simultaneously try to write page 5 and read page 5. Without per-page serialization, the read could return partially-written data. The per-page mutex map was the minimal solution that preserves parallelism for independent pages while guaranteeing correctness for concurrent accesses to the same page.

### Key Learnings

| Topic | What I Learned |
|-------|---------------|
| **Promises & futures** | `std::promise`/`std::future` provide a clean one-shot synchronization primitive. The producer sets the value, the consumer blocks on `.get()`. But you must *always* fulfill the promise — an unfulfilled promise is a ticking time bomb. |
| **Parallel I/O** | Disk I/O is inherently slow. A thread pool amortizes latency by overlapping multiple I/O operations. The key insight: parallelism is only safe when requests target different pages. |
| **Fine-grained locking** | The per-page mutex map is a classic "lock striping" pattern. A single global lock would serialize *all* I/O; per-page locks serialize only conflicting operations. The trade-off is memory (one mutex per active page) for throughput. |
| **Shutdown protocol** | Multi-threaded shutdown is subtle. Sentinel values in the queue are a clean pattern — each thread consumes exactly one sentinel, guaranteeing every thread exits. |

---

## Task 3 — Buffer Pool Manager

### What It Does

The BPM is the central coordinator. When the execution engine needs a page, it calls `ReadPage()` or `WritePage()`. The BPM either:

1. **Cache hit:** Finds the page already in a frame → pins it → returns a `PageGuard`.
2. **Cache miss:** Allocates a free frame (or evicts one via the ARC replacer) → reads the page from disk via the disk scheduler → pins it → returns a `PageGuard`.

All returned pages are wrapped in **RAII page guards** (`ReadPageGuard` / `WritePageGuard`) that automatically unpin the page and release locks when they go out of scope.

### Architecture & Design

- **Single BPM latch** (`bpm_latch_`): Protects the page table, free list, and replacer interactions. Held briefly to look up / modify metadata, then released before doing disk I/O.
- **Per-frame reader-writer lock** (`FrameHeader::rwlatch_`): A `std::shared_mutex` that allows concurrent readers but exclusive writers on each frame's data.
- **Page guards (RAII):** The guards acquire the frame's `rwlatch_` (shared for read, exclusive for write) and hold it until destruction. On destruction, they decrement `pin_count_` and, if the page was dirtied, flush it. This guarantees no page is leaked or left locked.
- **Helper function** `GetFrame()`: A private method that consolidates the logic for finding a frame (free list → eviction → disk read) to avoid code duplication between `ReadPage`, `WritePage`, `NewPage`, etc.

### Toughest Technical Challenges

**Deadlock between BPM latch and frame latch.** The most insidious bug. Consider:

```
Thread A: holds bpm_latch_ → tries to acquire frame.rwlatch_ (to evict a page)  
Thread B: holds frame.rwlatch_ (reading a page) → calls Unpin → tries to acquire bpm_latch_
```

Classic lock-ordering violation → deadlock. The fix was to establish a strict lock hierarchy:

1. Acquire `bpm_latch_` to do metadata lookups.
2. Release `bpm_latch_` before acquiring any `rwlatch_`.
3. Never re-acquire `bpm_latch_` while holding a `rwlatch_`.

This required restructuring the fetch/evict logic to separate the "decide what to do" phase (under `bpm_latch_`) from the "do the I/O" phase (under `rwlatch_` only).

**Dirty page writeback on eviction.** When evicting a dirty page, the BPM must write it back to disk *before* reusing the frame. But we can't hold the BPM latch during disk I/O (too slow, blocks all other threads). The solution: mark the frame as "in transit," release the BPM latch, perform the write, then re-acquire the latch to finalize the eviction.

### Key Learnings

| Topic | What I Learned |
|-------|---------------|
| **RAII** | Page guards are a textbook RAII application. Wrapping lock acquisition + pin count management in a guard that cleans up on destruction eliminates entire categories of bugs (forgot to unpin, forgot to unlock). This pattern is fundamental to writing safe C++. |
| **Deadlock prevention** | Deadlocks aren't just a textbook concept — they're a real, silent killer. The only reliable prevention is enforcing a global lock ordering and *never* violating it. Tools like ThreadSanitizer are invaluable for catching violations early. |
| **Lock hierarchy** | In any system with multiple locks, the design must specify which locks can be held simultaneously and in what order. For BusTub: `bpm_latch_` → `frame rwlatch_` → never the reverse. |
| **Separation of concerns** | Breaking the BPM into replacer + scheduler + manager made each piece individually testable and debuggable. A monolithic design would have been a nightmare to debug under concurrency. |

---

## Trade-offs

| Decision | Trade-off |
|----------|-----------|
| **Single BPM latch** | Simpler to reason about, but serializes all metadata operations. Under high concurrency, this becomes a bottleneck. A production system would use a concurrent hash map or partitioned page tables. |
| **4 disk scheduler workers** | Balances I/O parallelism with thread overhead. More workers would help with many concurrent page faults; fewer would reduce context-switching cost on a single-disk system. |
| **Per-page mutex in disk scheduler** | Uses more memory (one mutex per touched page), but allows parallel I/O to different pages. Alternative: a fixed-size array of mutexes indexed by `page_id % N` (lock striping) to bound memory usage at the cost of occasional false conflicts. |
| **Eager dirty page writeback** | Pages are written back on eviction. An alternative is periodic background flushing, which would reduce eviction latency at the cost of increased I/O and complexity. |

---

## Summary

This project was my first deep dive into systems-level C++ with real concurrency. The biggest takeaway: **concurrency bugs don't crash loudly — they corrupt silently.** Building the discipline to reason about lock ordering, use sanitizers religiously, and design for RAII from the start made all the difference. Every subsequent project in BusTub depends on this buffer pool being rock-solid, and the lessons from debugging deadlocks and race conditions here carry forward directly.
