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
