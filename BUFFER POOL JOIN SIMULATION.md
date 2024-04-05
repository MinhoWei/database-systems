# Buffer Replacement Strategies
Buffer replacement strategies in Database Management Systems (DBMS) are crucial for managing the cache of disk blocks in memory. These strategies determine which blocks to remove from the buffer pool (a cache in the system's main memory) to make room for new blocks when the pool is full and a new block needs to be read from disk. Some common buffer replacement strategies:
 - **Least Recently Used (LRU)**: LRU is a popular strategy that removes the least recently used block from the buffer. The idea is that blocks accessed recently are more likely to be accessed again soon.
 - **Most Recently Used (MRU)**: Contrary to LRU, the MRU strategy removes the most recently used block.
 - **First In First Out (FIFO)**: FIFO is a straightforward strategy where the oldest block in the buffer, i.e., the block that has been in the buffer the longest, is removed first.
 - **Random**: When the buffer manager needs to replace a block, instead of using complex heuristics or algorithms to determine which block to evict, it simply selects a block at random.
 - **Clock (Second Chance Algorithm)**: It arranges the buffer blocks in a circular manner (like the hands of a clock) and maintains a pointer to the oldest block. Each block has a use bit that is set to 1 when the block is accessed. When deciding which block to replace, the algorithm scans from the pointer forward, turning off the use bit and only replacing the block when it finds a block whose use bit is 0.
 - **Cycling**: Cycles through the slots and picks the next available.

# The Nested Loop Join
The nested-loop join is a particular method for executing the following SQL query:
```
select * from R join S;
```

It can be described algorithmically as:

![](https://github.com/MinhoWei/database-systems/blob/main/buffer1.png)

# Using Buffer Pool

Remarks:
- When using a buffer pool, each page is obtained via a call to the **request_page()** function.
- If the page is already in the pool, it can be used from there. If the page is not in the pool, it will need to be read from disk, most likely replacing some page that is currently in the pool if there are no free slots (using the buffer replacement strategies from above).
- If no buffer pool is used (i.e. one input buffer per table), the number of pages that will need to be read from disk is b<sub>R</sub> + b<sub>R</sub>b<sub>S</sub>, where b<sub>R</sub> and b<sub>S</sub> are the number of pages in tables R and S respectively (nested loop join; very inefficient).

## bufpool.c
(Needs to implement removeFromUsedList(), grabNextSlot() and makeAvailable().
```
// bufpool.c ... buffer pool implementation

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>
#include "bufpool.h"

#define MAXID 4

// for Cycle strategy, we simply re-use the nused counter
// since we don't actually need to maintain a usedList
#define currSlot nused

/*
id:    buffer id (used to uniquely identify a buffer)
pin:   pin count, determines how many processes are currently rely on this buffer
       (a buffer can only be removed when pin = 0)
dirty: indicates whether the buffer has been modified after being loaded into memory
*data: holds the actual data in the buffer
*/
struct buffer {
	char  id[MAXID];
	int   pin;
	int   dirty;
	char  *data;
};

struct bufPool {
	int   nbufs;         // how many buffers
	char  strategy;      // LRU, MRU, Cycle
	int   nrequests;     // number of requests made to the buffer pool (read or write in a buffer)
	int   nreleases;     // how many times buffers are released back to the pool (typically the pin count is decreased)
	int   nreads;        // number of read operations
	int   nwrites;       // number of write operations
	int   *freeList;     // list of completely unused bufs
	int   nfree;         // number of currently free buffers
	int   *usedList;     // implements replacement strategy
	int   nused;         // number of used buffers (pinned or dirty)
	struct buffer *bufs; // array of buffers
};

static unsigned int clock = 0;

// Helper Functions (private)

// - return the name (id) of the page (buffer) current stored in specified slot
static
char *pageInBuf(BufPool pool, int slot)
{
	char *pname;
	pname = pool->bufs[slot].id;
	if (pname[0] == '\0')
		return "_";
	else
		return pname;
}

// - check whether page from rel is already in the pool
static
int pageInPool(BufPool pool, char rel, int page)
{
	int i;  
	char id[MAXID];
	sprintf(id,"%c%d",rel,page); // construct the id
	for (i = 0; i < pool->nbufs; i++) {
		if (strcmp(id,pool->bufs[i].id) == 0) { // strcmp returns 0 if two strings are identical
			return i;
		}
	}
	return -1;
}

// - use the first slot on the free list
// - the slot is removed from the free list
//   by moving all later elements down
static
int removeFirstFree(BufPool pool)
{
	int first_free_idx, i;
	assert(pool->nfree > 0);
	first_free_idx = pool->freeList[0];
	for (i = 0; i < pool->nfree-1; i++)
		pool->freeList[i] = pool->freeList[i+1];
	pool->nfree--;
	return first_free_idx;
}

// - search for a slot in the usedList and remove it
// - depends on how usedList managed, so is strategy-dependent
static
void removeFromUsedList(BufPool pool, int slot)
{
	int i, j;
	switch (pool->strategy) {
	case 'L':
		// remove from LRU usedList 
		break;
	case 'M':
		// remove from MRU usedList
		break;
	case 'C':
		// remove from cycled usedList
		break;
	}
}

//getNextSlot(pool)
// - finds the "best" unused buffer pool slot
// - "best" is determined by the replacement strategy
// - if the replaced page is dirty, write it out
// - initialise the chosen slot to hold the new page
// - if there are no available slots, return -1

static
int grabNextSlot(BufPool pool)
{
	int slot;
	switch (pool->strategy) {
	case 'L':
		// get least recently used slot from used list
		break;
	case 'M':
		// get most recently used slot from used list
		break;
	case 'C':
		// get next available according to cycle counter
		break;
	}
	if (slot >= 0 && pool->bufs[slot].dirty) {
		pool->nwrites++;
		pool->bufs[slot].dirty = 0;
	}
	return slot;
}


//makeAvailable(pool,slot)
// - add the specified slot to the used list
// - where to add depends on strategy

static
void makeAvailable(BufPool pool, int slot)
{
	switch (pool->strategy) {
	case 'L':
		// slot become most recently used
		break;
	case 'M':
		// slot become most recently used
		break;
	case 'C':
		// slot becomes available
		break;
	}
}

// Interface Functions

// - initialise a buffer pool with nbufs
// - buffer pool uses supplied replacement strategy
BufPool initBufPool(int nbufs, char strategy)
{
	BufPool newPool;
	struct buffer *bufs;

    // initialize parameters
	newPool = malloc(sizeof(struct bufPool));
	assert(newPool != NULL);
	newPool->nbufs = nbufs;
	newPool->strategy = strategy;
	newPool->nrequests = 0;
	newPool->nreleases = 0;
	newPool->nreads = 0;
	newPool->nwrites = 0;
	newPool->nfree = nbufs;
	newPool->nused = 0;
	newPool->freeList = malloc(nbufs * sizeof(int));
	assert(newPool->freeList != NULL);
	newPool->usedList = malloc(nbufs * sizeof(int));
	assert(newPool->usedList != NULL);
	newPool->bufs = malloc(nbufs * sizeof(struct buffer));
	assert(newPool->bufs != NULL);

	int i;
	for (i = 0; i < nbufs; i++) {
		newPool->bufs[i].id[0] = '\0';
		newPool->bufs[i].pin = 0;
		newPool->bufs[i].dirty = 0;
		newPool->freeList[i] = i;
		newPool->usedList[i] = -1;
	}
	return newPool;
}

// request from relation 'rel', page number 'page'
int request_page(BufPool pool, char rel, int page)
{
	int slot;
	printf("Request %c%d\n", rel, page);
	pool->nrequests++;
	slot = pageInPool(pool,rel,page);
	if (slot < 0) { // page is not already in pool
		if (pool->nfree > 0) {
			slot = removeFirstFree(pool);
		}
		else {
			slot = grabNextSlot(pool);
		}
		if (slot < 0) {
			fprintf(stderr, "Failed to find slot for %c%d\n",rel,page);
			exit(1);
		}
		pool->nreads++;
		sprintf(pool->bufs[slot].id,"%c%d",rel,page);
		pool->bufs[slot].pin = 0;
		pool->bufs[slot].dirty = 0;
	}
	// have a slot
	pool->bufs[slot].pin++;
	removeFromUsedList(pool,slot);
	showPoolState(pool);  // for debugging
	return slot;
}

// page release
// marks the page as no longer being used by the caller
// potentially freeing it up for replacement or reuse within the pool
void release_page(BufPool pool, char rel, int page)
{
	printf("Release %c%d\n", rel, page);
	pool->nreleases++;

	int i;
	i = pageInPool(pool,rel,page);
	assert(i >= 0);
	// last user of page is about to release
	if (pool->bufs[i].pin == 1) {
		makeAvailable(pool, i);
	}
	pool->bufs[i].pin--;
	showPoolState(pool);  // for debugging
}

// - prints statistics counters for buffer pool
void showPoolUsage(BufPool pool)
{
	assert(pool != NULL);
	printf("#requests: %d\n",pool->nrequests);
	printf("#releases: %d\n",pool->nreleases);
	printf("#reads   : %d\n",pool->nreads);
	printf("#writes  : %d\n",pool->nwrites);
}

// - display printable representation of buffer pool on stdout
void showPoolState(BufPool pool)
{
	int i, j; char *p; struct buffer b;

	printf("%4s %6s %6s %6s\n","Slot","Page","Pin","Dirty");
	for (i = 0; i < pool->nbufs; i++) {
		b = pool->bufs[i];
		p = pageInBuf(pool,i);
		printf("[%02d] %6s %6d %6d\n", i, p, b.pin, b.dirty);
	}
	printf("FreeList:");
	for (i = 0; i < pool->nfree; i++) {
		j = pool->freeList[i];
		printf(" [%02d]%s", j, pageInBuf(pool,j));
	}
	printf("\n");
	printf("UsedList:");
	for (i = 0; i < pool->nused; i++) {
		j = pool->usedList[i];
		printf(" [%02d]%s", j, pageInBuf(pool,j));
	}
	printf("\n");
}
```
