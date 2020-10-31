---
title: "Malloclab Reflections"
lang: en
toc: true
permalink: /posts/malloclab
tags:
  - study
---

# MallocLab - A lab from CMU 18-613

## Before we begin
The malloc lab is, by far, the most challenging and rewarding lab that I've ever had in CMU18-613. The writeup is 25 pages long, and it took me 5 days to get full mark (100/100.) Mine implementation is not the best (KOPS=9443 and UTIL=74.2%), but it suffices to help me build a solid understanding on this matter. If you, by any chances, are working on the lab with a due date. My suggestion is to sit back, prepare to embrace seg_fault, and have fun. Oh, as always, start early! I finished it 11 days earlier than the due date. Unless you are a genius, do not wait till the last minute. But do not feel discouraged by above descriptions. 

To be honest, after finishing the lab and looking back, it seems much less difficult than I first read the spec. So, do not be too panic. Following are my reflections on this lab. Feel free to refer to my post, but please hold yourself from conducting AIV. OK! Buckle up and let's get started!

*Just shoot me an email if you find it useful, or you have suggestions. Your reaching out to me makes me feel less lonely in the world of challenging HW and assignments.*

## Short Introduction
The lab requires us to implement a **dynamic memory allocator**. In English, we need to implement `init`, `malloc`, `free`, `realloc` and `calloc` functions in C. We also need to implement a debug_func called `mm_checkheap`. So on the bright side, we only need to implement 6 functions! Easy, right? 
~~right your head right~~. If you are not familiar with linux malloc, my advice for you is to read the manual `man malloc` before you start.

Also, the lab comes with 2 pages of programming rules. Read through them line by line to avoid rewriting your implementation. For your convenience, I've summarized them,

1. Your allocator must not explicitly detect which traces are running. ~~(but you can write an adaptive allocator if you have time.)~~
2. You should not change the interface of `mm.h`
3. global data should sum to at most 128 bytes
4. You should not write macros that has parameters. eg, `#define GET(p) (*(unsigned int *)(p))`
5. Your allocator must return pointers that are aligned to 16-byte boundaries
6. NO AIV, NO warnings. WARNINGS are ERRORS in this course

## Phase 0: Road clearing
**Caveat: Everything in this phase are for conceptual understanding. In real life and this lab, things are much complicated!**

Hours of thinking can save you from days of debugging. So make sure that you have understood everything before coding. Here, I want to highlight some of the key points that I personally think is of great importance. If you are familiar with the material, feel free to skip this section.
### Malloc
Malloc is a function to allocate memory during runtime. It is super useful for data structures whose size are unknown until runtime. The malloced blocks are stored in heap, a segment of memory addressses reserved almost exclusiviely for malloc to use. 

```
Heap: | prologue | free | d | free | free | e | e | e | ... | epilogue |
```

Now, assuming byte-addressible, it's easy to explain what `a = malloc(2);` does:

1. Go to the heap
2. Find 5 consecutive free bytes in the heap (*this is wrong and I will explain why*)
3. occupy it and let a point to the start of the chosen byte

```
Heap: | prologue | free | d | a | a | e | e | e | ... | epilogue |
```

Easy, right? However, this is the abstraction we made for the programmers. There are actually a lot things happening behind the scene. Do not believe me? Try answering the following questions

1. how should we tell whether a block is free or allocated? They are 0/1s, not free/a/b/c/d, ...
2. how should we find free blocks?
3. what if we consider the alignment requirement?

Hence, in addition to the spaces that the programmer requests [*payload*], we actually need more space to store some information of the blocks [*overhead*]. The hero behind the scene, `malloc` actually does the three things:

1. organizes all blocks and stores information about them in a structured way
2. uses the structure made to choose an appropriate location to allocate new memory
3. updates the structure when the user frees a block of memory

In English, `malloc` is not only a waiter/waitress, it is also a manager.

### Implicit-list no-coalesce allocator
I strongly recommend you to understand the provided implicit-list no-coalesce allocator. Implicit list connects and organizes all blocks and stores information about them in a structured way, typically implemented as a **singly linked list**. To fulfill it, the structure of each component is

* block
  * free block       `|header|padding|footer|`
  * allocated block  `|header|payload|(padding)|footer|`
* overhead
  * header          `|size, is_alloc|`
  * footer           `|size, is_alloc|`

The actually implementaion in the code is a little bit different,

```c
typedef struct block {
    /** @brief Header contains size + allocation flag */
    word_t header;     // the overhead we are talking about
    char payload[0];   // allocated space to store data given by the programmer
} block_t;
```

You may tell that footer excluded form the `struct block`. Hence, the real game is this,
```
abstraction:   |         real block            |
malloc's view: |header|payload+(padding)|footer|
code's view:   |         block_t        |footer|  
```
You can use the provided helper functions to move among header, payload, and footer.

### Double-word alignment
This is one of my first obstacles. What does double-word alignment mean? 

Since we are working on 64-bit machines, a word (`wsize`) is 8B, then double word (`dsize`) is 16B. Hence, the pointer to each payload should be divisible by 16 (4'b10000). Now let's add the size of each component,

* prologue (wsize)
* block (??)
  * header (wsize)          `|size, is_alloc|`
  * payload/padding/payload+padding (??)
  * footer (wsize)           `|size, is_alloc|`
* block ...
* epilogue (wsize)

Now can you tell me how large the payload should be? It has to be divisible by `dsize` in order to achieve double alignment. Only in this way can the total block size (8+8+16N) be divisible by `dsize`. Hence, a side product is that the `min_block_size` for each block should be 32B.

Now comes the most important part! How should we align payload? 

**!!!Remember: we are aligning payloads, not headers!!!**.

The answer is surpriseingly easy. The structure has automated it for us. There is nothing we need to explicitly do to preserve double alignment w.r.t. payload. Why?

```
          |<- double aligned place 0x...0    
-----------------------------------------------------------------------
|         |         |         |         |         |         |         |
|prol| h  | p  | p  | p  | p  | f  | h  | p  | p  | -  | -  | f  |epil|
|         |         |         |         |         |         |         |
-----------------------------------------------------------------------
               |<-- 0x...8
* prol->prologue, h->header, p->payload, --> padding, f->footer, epil->epilogue
* |    | -> word is 8 bytes
* |    |    | -> doubleword is 16 bytes
```
You can see that if all invariants are followed, headers are always at one word before double aligned place, and footer is always at double aligned place. 
```
The only possible stuff in a double alinged double word:

   |<- double aligned place 0x...0    
   -----------
1. |prol| h  |
2. | p  | p  |
3. | f  | h  |
4. | f  |epil|
   -----------
        |<-- 0x...8
```

Hence, the payload is guaranteed to start at a double aligned place. It's so elegant that you do not need to do a thing...

### Trick
Remember that the size of block (not `block_t`) should be divisible by 16 (`dsize`)? This tells us that we can use the last four bits of the header/footer however we like. Now, can you understand the following helper functions?
```c
static const word_t alloc_mask = 0x1;
static const word_t size_mask = ~(word_t)0xF;

static size_t extract_size(word_t word) {
    return (word & size_mask);
}

static bool extract_alloc(word_t word) {
    return (bool)(word & alloc_mask);
}

static word_t pack(size_t size, bool alloc) {
    word_t word = size;
    if (alloc) {
        word |= alloc_mask;
    }
    return word;
}
```

**To be continued...**
Since I have more dues coming, I will jump to the key points. I may update more background info when available.

## Phase 1: Checkpoint
Welcome to the actual coding part! From this section, I will organize the rest according to how I developed. There are numerous other implmenetations for you to try out. So do not restrain your minds.

### Implicit-list with-coalesce allocator


### Explicit-list with-coalesce allocator

### Seg-list with-coalesce allocator

### find_fit algorithm

### Parameter tunning



## Phase 2: Final submission

### Footer-removed Seg-list allocator
After a second look, we may find out that the footer for allocated blocks are useless. By removing it, we can hold larger payload in an allocated block.
* prologue (wsize)
* block (32N)
  * allocated block (32N)
    * header (wsize) `|size, prev_alloc, is_alloc|`
    * payload (can be larger than 16N)
  * free block (32)
    * header (wsize) `|size, prev_alloc, is_alloc|`
    * next   (wsize) 
    * prev   (wsize)
    * footer (wsize) `|size, prev_alloc, is_alloc|`
* block ...
* epilogue (wsize)





### Miniblock Footer-removed Seg-list allocator
Above this section, we assume the each block is at least 32B. But do we actually need that amount of spaces for any blocks? Can we reduce `min_block_size`?
Well yes, you may see that the not all free blocks need to be recorded as a doubly linked-list. In old times when we have`16 <= size - asize < 32` in `split` function, we refuse to further split it, but this precondition is too tight. We can allow free blocks to have only 16 Bytes. In other words, we can further improve the allocator to have `min_block_size=16`. This can further reduce fragmentations.
* prologue (wsize)
* block (32N)
  * allocated block (32N)
    * header (wsize) `|size, prev_alloc, is_alloc|`
    * payload (can be larger than 16N)
  * free block (16)
    * header (wsize) `|size, prev_alloc, is_alloc|`
    * next   (wsize)
* block ...
* epilogue (wsize)

#### One modification SAVES your throughput
Instead of traversing the 16B_freeBlock free list to find out whether the previous block is a 16B freeBlock or not, why do not we mimic the "is_prev" method and use a bit in header to indicate whether the previous block is a 16B freeBlock or not?

* prologue (wsize)
* block (32N)
  * allocated block (32N)
    * header (wsize) `|size, is_prev_16B_freeBlock, prev_alloc, is_alloc|`
    * payload (can be larger than 16N)
  * free block (16)
    * header (wsize) `|size, is_prev_16B_freeBlock, prev_alloc, is_alloc|`
    * next   (wsize)
* block ...
* epilogue (wsize)

At this point, you can get >98. Great job! Cheers! But what is missing?


### Final Push: find_fit and structure optimization
If you can not get full mark, then it's a good time to take a rest, and come back later.

OK, I assume you've taken plenty of rest. Let's continue.

I am a fan of "find_first_fit" and "LIFO-list" because of it is absolute simplicity to implement. Pretty normal routine. But the ease to implement is currently backfiring. We should turn to a wiser yet not-that-too-complicated stragety. The very opposite is "find_best_fit", which requires traversing the whole list in worst case (O(N)), can we find a middle ground between them?

The answer is yes. Why do not we adopt "find_best-ish_fit"? or, why do not we do some sorting-ish when maintaining the free list?

Yea, yea, that's right! Now you are guaranteed to have a full mark.

Congrats!!!!

### Parameter tuning