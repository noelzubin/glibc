# Malloc
`void *malloc( size_t size );`
allocates size bytes of uninitalized storage. 
If allocation succeeds, returns a pointer to the block.
If size is 9, depends on implementation, it may or may not return nulltpr. if it
returns address it should be pass to free to free

Example code
``` cpp
#include <stdio.h>   
#include <stdlib.h> 
 
int main(void) 
{
    int *p1 = malloc(4*sizeof(int));  // allocates enough for an array of 4 int
    int *p2 = malloc(sizeof(int[4])); // same, naming the type directly
    int *p3 = malloc(4*sizeof *p3);   // same, without repeating the type name
 
    if(p1) {
        for(int n=0; n<4; ++n) // populate the array
            p1[n] = n*n;
        for(int n=0; n<4; ++n) // print it back out
            printf("p1[%d] == %d\n", n, p1[n]);
    }
 
    free(p1);
    free(p2);
    free(p3);
}
```


This implementation is a version of malloc/free/realloc that Doug Lea wrote and Wolfram Gloger adapted for multiple threads and arenas. It's important to note that this version, sometimes referred to as ptmalloc2, has undergone substantial changes after being integrated into glibc, the GNU C Library, making it significantly different from the standalone ptmalloc2 version.


Main properties of the algorithm:
  The main properties of the algorithms are:
  * For large (>= 512 bytes) requests, it is a pure best-fit allocator,
    with ties normally decided via FIFO (i.e. least recently used).
  * For small (<= 64 bytes by default) requests, it is a caching
    allocator, that maintains pools of quickly recycled chunks.
  * In between, and for combinations of large and small requests, it does
    the best it can trying to meet both goals at once.
  * For very large requests (>= 128KB by default), it relies on system
    memory mapping facilities, if supported.


`struct malloc_chunk` keeps track of allocated and free memory chunks.

for `malloc(size_t n)` if n is 0. malloc returns a minimum sized chunk which is
16 bytes on most 32 bit systems, and 24 or 32 bytes on 64 bit systems.

`free(void *p)` frees the memory space pointed to by p, If alread freed it can
bad effects.

the main code is in `__libc_malloc(size_t bytes)`

#### Tcache TCache (Thread Cache)
Before the introduction of TCache, operations like malloc and free could become performance bottlenecks in multithreaded programs due to contention for global heap structures. 
. TCache addresses this by providing each thread with a small, private cache for allocating and freeing memory blocks.
: When TCache cannot satisfy an allocation request (either because the requested size is too large for the cache or the cache is depleted), the request is forwarded to the global allocator. Similarly, if the cache for a particular thread becomes full, subsequent freed blocks are returned to the global pool.

### Arenas
If all threads use a single global heap for these operations, the program must ensure that only one thread modifies the heap at any given time, typically using locks. This serialization can significantly hinder performance in applications with heavy dynamic memory usage across multiple threads.
The Concept of Arenas
To mitigate this issue, glibc’s memory allocator, ptmalloc (which is derived from Doug Lea's malloc, or dlmalloc), introduces the concept of arenas. An arena is essentially a separate instance of the heap memory, with its own separate free list and allocation metadata. Each thread in a multithreaded application can allocate memory from its own arena, reducing the need for locking and thereby decreasing contention.


#### Fastbins
Fastbins are used exclusively for small memory allocations. Each fastbin is a singly linked list that holds free blocks of a specific size class. This size-specific strategy allows very fast allocation and deallocation because the allocator can directly go to the fastbin of the appropriate size when processing a request.
Blocks in fastbins are managed in a last-in-first-out (LIFO) order. This means that the most recently freed block is the first one to be reused in a subsequent allocation. The LIFO strategy helps keep recently used memory pages in the cache, potentially improving cache efficiency.
: Unlike the regular free list, blocks in fastbins are not coalesced when they are freed
 To mitigate fragmentation caused by the lack of immediate coalescing in fastbins, the allocator may periodically consolidate fastbins with the regular free list, where coalescing can occur. This consolidation typically happens during certain allocation requests that cannot be satisfied from the fastbins or when explicitly triggered by calling malloc_trim().


## Internal data structures
Bin: An array of bin headers for free chunks. Each bin is doubly linked. The bin at index 1 is used to store unsorted chunks. This list acts as a queue. The chunks here are given a last chance to be used before binning.

Binmap: To help compensate for the large number of bins, a one-level index structure is used for bin-by-bin searching. 

FastBins: Array of lists holding recently freed small chunks.Fastbins
are not doubly linked.  It is faster to single-link them, and since chunks are
never removed from the middles of these lists, double linking is not necessary.
Also, unlike regular bins, they are not even processed in FIFO order (they use
faster LIFO) since ordering doesn't much matter in the transient contexts in
which fastbins are normally used.


struct `malloc_state` is the main structure that holds all the information about the heap.
  * mutex: A lock to protect the data structures in the malloc_state from concurrent access by multiple threads.
  * flags: A bit mask that holds various flags that control the behavior of the allocator.
  * fastbinsY: An array of fastbin lists. Each list holds free blocks of a specific size class.
  * top: A pointer to the top chunk, which is the first chunk in the main free list.
  * last_remainder: A pointer to the last remainder chunk, which is the last chunk that was split to satisfy an allocation request.
  * bins: An array of bin headers for free chunks. Each bin is doubly linked. The bin at index 1 is used to store unsorted chunks. This list acts as a queue. The chunks here are given a last chance to be used before binning.
  * binmap: A one-level index structure to help compensate for the large number of bins. It is used for bin-by-bin searching.
  * next: A pointer to the next malloc_state in the list of all malloc_state instances. This list is used to keep track of all arenas in the process.
  * attached_threads: A list of threads that are attached to this malloc_state. This list is used to keep track of all threads that are using this arena.
  * system_mem: A pointer to the base of the system memory map. This is used to keep track of the memory that is mapped to the process by the system.
  * max_system_mem: The maximum amount of system memory that is mapped to the process. This is used to keep track of the memory that is mapped to the process by the system.
  * n_mmaps: The number of memory mappings that are currently active in the process. This is used to keep track of the memory that is mapped to the process by the system.
  * n_mmaps_max: The maximum number of memory mappings that are allowed in the process. This is used to keep track of the memory that is mapped to the process by the system.
  * max_n_mmaps: The maximum number of memory mappings that are allowed in the process. This is used to keep track of the memory that is mapped to the process by the system.
  * mmapped_mem: The total amount of memory that is currently mapped to the process by the system. This is used to keep track of the memory that is mapped to the process 
