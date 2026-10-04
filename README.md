# BusTub — CMU 15-445/645 Database Systems

## Overview

This repository contains my implementation of [Carnegie Mellon University's 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/fall2025/) course projects. BusTub is a relational database management system built from scratch in C++, designed to teach the internals of modern database engines.

> **Note:** Per CMU's academic integrity policy, the source code cannot be made public. This showcase directory documents the components I implemented, the design decisions I made, and the lessons I learned — without exposing the implementation itself.

> **AI Disclaimer:** To gain the best learning outcomes, all code in this project was written entirely by me, without the use of any AI coding assistants. AI tooling was used **only** for writing this documentation/showcase — not for any part of the implementation.

---

## Course Projects

| # | Project | Component | Status | Writeup |
|---|---------|-----------|--------|---------|
| 1 | [Buffer Pool Manager](p1-buffer-pool-manager.md) | Memory management, caching, disk I/O | ✅ Complete | [Details →](p1-buffer-pool-manager.md) |
| 2 | B+ Tree Index | Concurrent index structure | 🔲 Upcoming | — |
| 3 | Query Execution | Volcano-model operators, joins, aggregations | 🔲 Upcoming | — |
| 4 | Concurrency Control | MVCC, transaction isolation | 🔲 Upcoming | — |

---

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│                   SQL Layer                         │
│         Parser → Binder → Planner → Optimizer       │
├─────────────────────────────────────────────────────┤
│               Execution Engine                      │
│     Sequential Scan, Hash Join, Aggregation, ...    │
├─────────────────────────────────────────────────────┤
│                 Index Layer                          │
│              B+ Tree (concurrent)                   │
├─────────────────────────────────────────────────────┤
│             Buffer Pool Manager          ← P1       │
│         ARC Replacer · Page Guards · Disk I/O       │
├─────────────────────────────────────────────────────┤
│               Storage Layer                         │
│          Disk Manager · Disk Scheduler              │
└─────────────────────────────────────────────────────┘
```

Each project builds on the previous one — the buffer pool manager from P1 is used by every layer above it, making correctness and thread safety critical from the start.

---

## Tech Stack

- **Language:** C++17
- **Build System:** CMake
- **Testing:** Google Test
- **Sanitizers:** AddressSanitizer, ThreadSanitizer
- **Tools:** clang-tidy, clang-format

---

## Key Skills Demonstrated

- **Systems Programming:** Low-level memory management, raw pointers, RAII patterns
- **Concurrency:** Mutexes, shared mutexes (reader-writer locks), lock-free data structures, deadlock prevention
- **Data Structures:** ARC (Adaptive Replacement Cache) eviction policy, hash tables, B+ trees
- **Software Engineering:** Test-driven development, debugging with sanitizers, performance benchmarking

---

## How to Read This Showcase

Each project writeup follows a consistent structure:

1. **High-Level Objective** — What the component does and why it matters
2. **Architecture & Design Choices** — How I approached the problem
3. **Toughest Technical Challenges** — Hardest bugs and how I resolved them
4. **Trade-offs** — Performance vs. correctness decisions
5. **Key Learnings** — What the project taught me about systems programming
