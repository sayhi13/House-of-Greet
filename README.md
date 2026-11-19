# House of Greet

An analysis of `glibc` `ptmalloc2` **unsorted bin** doubly-linked list behavior, examining how forged chunk metadata and `fd`/`bk` pointer manipulation interact with allocator unlinking and `unsorted_chunks` address exposure while preserving bin list integrity.

---

## Overview

The technique focuses on how `ptmalloc2` traverses and unlinks chunks from the **unsorted bin** (starting from the tail) and how controlled metadata/pointers in a writable region can produce an overlapping allocation and expose an `unsorted_chunks` pointer while leaving the cyclic list usable for subsequent steps.

### Key Characteristics
* **Target Allocator:** `glibc` `ptmalloc2` unsorted bin doubly-linked list.
* **Core Mechanics:** Forged chunk metadata, `fd`/`bk` pointer wiring, tail/head updates during unlinking, and `unsorted_chunks` address exposure.
* **Post-Condition:** Preserves cyclic unsorted bin integrity so further heap operations can proceed cleanly.

---

## `glibc` Version Regimes

| Regime | Key Constraints & Checks | Behavioral Impact |
| :--- | :--- | :--- |
| **`glibc <= 2.27`** | Fewer unlink integrity checks; requires a heap leak, controlled fake chunk location, and `bk` control on an unsorted bin chunk. | Broader allocation size flexibility and fewer structural constraints during unlinking. |
| **`glibc >= 2.28`** | Adds unsorted bin integrity validation (commit `b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c`): next-chunk size/`prev_size`/`PREV_INUSE` checks and `bck->fd == victim` / `victim->fd == unsorted_chunks(av)` verification. | Requires consistent metadata at the adjacent chunk offset and pointer alignment across both `victim` chunks in the list. |

---

## Repository Structure

```text
.
├── README.md            # Technique overview, version comparison, and usage notes
├── house_of_greet.md    # translated to English
└── house_of_greet_ru.md # original explanation
```

---

## Lab Environment & Reproducing State Transitions

1. **Build in an isolated test environment** matching the target `glibc` version (`<= 2.27` or `>= 2.28`), using a debugger (`GDB` with `pwndbg` or `gef`) to inspect `unsorted_chunks`, chunk headers, and `fd`/`bk` pointers step by step.
2. **Observe heap state transitions** across `malloc`/`free` sequences, metadata placement, and unlink checks rather than relying on hardcoded addresses.

---

## Defensive Takeaway

Strict metadata validation on unlink—such as verifying `fd`/`bk` reciprocity and adjacent chunk `size`/`prev_size`/`PREV_INUSE` invariants introduced in `glibc 2.28`—alongside eliminating Use-After-Free and out-of-bounds write primitives prevents forged bin entries from passing allocator consistency checks.
