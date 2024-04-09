# Sorted Files
Records stored in file in order of some field k (the sort key). This will make searching more efficient and insertion less efficient.

In order to mitigate insertion costs, use overflow blocks.

-> Total number of overflow blocks = bov

-> Average overflow chain length (average # of overflow blocks per page) = Ov = bov / b

-> **Bucket** = data page + its overflow page(s)

## Selection in Sorted Files
For one queries on sort key, use binary search.
```
// select * from R where k = val  (sorted on R.k)
lo = 0; hi = b-1
while (lo <= hi) {
    mid = (lo+hi) / 2;  // int division with truncation
    (tup,loVal,hiVal) = searchBucket(f,mid,k,val);
    if (tup != NULL) return tup;
    else if (val < loVal) hi = mid - 1;
    else if (val > hiVal) lo = mid + 1;
    else return NOT_FOUND;
}
return NOT_FOUND;
```

Implementation of **searchBucket** function (search a page and its overflow chain for a key value):
```
searchBucket(f,pid,k,val)
{
    buf = getPage(f,pid);
    (tup,min,max) = searchPage(buf,k,val,+INF,-INF)
    if (tup != NULL) return(tup,min,max);
    ovf = openOvFile(f);
    ovp = ovflow(buf);
    while (tup == NULL && ovp != NO_PAGE) {
        buf = getPage(ovf,ovp);
        (tup,min,max) = searchPage(buf,k,val,min,max)
        ovp = ovflow(buf);
    }     
    return (tup,min,max);
}
```
(Assumes each page contains index of next page in Ov chain, and getPage(f,pid) = { read_page(f,pid,buf); return buf; })

Implementation of **searchPage** function:
```
searchPage(buf,k,val,min,max)
{
    res = NULL;
    for (i = 0; i < nTuples(buf); i++) {
        tup = getTuple(buf,i);
        if (tup.k == val) res = tup;
        if (tup.k < min) min = tup.k;
        if (tup.k > max) max = tup.k;
    }
    return (res,min,max);
}
```
(This function also updates the min and max values to track the lowest and highest key values seen in the page, which is crucial for narrowing down the binary search range in the main algorithm.)

Cost analysis:
 - best: find tuple in first data page we read (1)
 - worst: full binary search, and not found, this costs examine log2b data pages plus examine all of their overflow pages (log2 b + bov)
 - average: examine some data pages + their overflow pages

## Search with PMR Queries
For pmr query, on *non-unique attribute k*, where file is sorted on k, tuples containing k may span several pages (can be multiple records (tuples) with the same key value).

For example, 'select * from R where k = 2' may have the following situation:
```
[0]          [1]          [2]
 | 1, 1, 2, 2 | 2, 2, 2, 2 | 2, 3, ...
                             |
               Binary search will land here
```
In this case, we do the following:
 - Begin by locating a page p containing k=val   (as for one query).
 - Scan backwards and forwards from p to find matches.
 - Costpmr  =  Costone + (bq-1).(1+Ov)

## Search with Ranged Queries
For range queries on *unique* sort key (e.g. primary key):
 - use binary search to find lower bound
 - read sequentially until reach upper bound

For example, consider the query: 'select * from R where k >= 5 and k <= 13', we may have the following situation:
```
[0]          [1]          [2]         [3]
 | 1, 2, 3, 4 | 5, 6, 7, 8 | 9, 10, 11 | 12, 13, 14|
                |                             |
             starts here                  ends here
```
Costrange  =  Costone + (bq-1).(1+Ov) (same as pmr query)

For range queries on non-unique sort key, similar method to pmr.
 - binary search to find lower bound
 - then go backwards to start of run
 - then go forwards to last occurence of upper-bound

Costrange = Costone + (bq-1).(1+Ov)

Note: If condition contains attribute j, not the sort key

-> file is unlikely to be sorted by j as well
-> sortedness gives no searching benefits

Costone,   Costrange,   Costpmr   as for heap files

## Insertion into Sorted Files
Insertion approach:
 - find appropriate page for tuple (via binary search)
 - if page not full, insert into page
 - otherwise, insert into next overflow block with space

Costinsert  =  Costone + δw         (where δw = 1 or 2)

## Deletion from Sorted Files
E.G. Consider the query: 'delete from R where k = 2'

Deletion strategy:
 - find matching tuple(s) and mark them as deleted
 - Cost depends on selectivity of selection condition, and selectivity determines bq   (# pages with matches): Costdelete  =  Costselect + bqw

# Hashed Files
## Hashing
Basic idea: use *key value* to compute *page address* of tuple.

Requires: hash function h(v) that maps KeyDomain → [0..b-1].
 - hashing converts key value (any type) into *integer value*
 - *integer value* is then mapped to *page index*
 - can view *integer value* as a *bit-string*

PostgreSQL hash function (simplified):

This function is designed to convert a key of arbitrary length into a *32-bit integer* value that can be used as an index to map the key to a page in a hash table.
```
Datum hash_any(unsigned char *k, register int keylen)
// k is a pointer to the key to be hashed, treated as an array of bytes
{
   register uint32 a, b, c, len;
   /* Set up the internal state */
   len = keylen;                           // assigned the length of the key
   a = b = c = 0x9e3779b9 + len + 3923095; // these are working variables

   /* handle most of the key */
   // processing the key in chunks of 12 bytes at a time
   while (len >= 12) {
      a += ka[0]; b += ka[1]; c += ka[2];
      mix(a, b, c);
      ka += 3;  len -= 12;
   }
   /* collect any data from last 11 bytes into a,b,c */
   mix(a, b, c);
   // UInt32GetDatum() function (which is part of the PostgreSQL internals) converts this 32-bit integer into a Datum
   return UInt32GetDatum(c);
}
```

hash_any() gives hash value as 32-bit quantity (uint32).

Two ways to map raw hash value into a page address:

-> if b = 2^k, bitwise AND with *k low-order bits* set to one
```
uint32 hashToPageNum(uint32 hval) {
    uint32 mask = 0xFFFFFFFF;
    return (hval & (mask >> (32-k)));
}
```

-> otherwise, use mod  to produce value in range 0..b-1
```
uint32 hashToPageNum(uint32 hval) {
    return (hval % b);
}
```

**Best case**: every bucket contains same number of tuples.

**Worst case**: every tuple hashes to same bucket.

**Average case**: some buckets have more tuples than others.

Two important measures for hash files:
 - **load factor**:   L  =  r / bc (to achieve average case, aim for   0.75  ≤  L  ≤  0.9)
 - **average overflow chain length**:   Ov  =  bov / b

### Selection with Hashing
Select via hashing on *unique* key k (one)
```
// select * from R where k = val
(pid,P) = getPageViaHash(val,R)
for each tuple t in page P {
    if (t.k == val) return t
}
for each overflow page Q of P {
    for each tuple t in page Q {
        if (t.k == val) return t
}   }
```
Costone :   Best = 1,   Avg = 1+Ov/2   Worst = 1+max(OvLen)

Select via hashing on *non-unique* hash key nk (pmr)
```
// select * from R where nk = val
(pid,P) = getPageViaHash(val,R) 
for each tuple t in page P {
    if (t.nk == val) add t to results
}
for each overflow page Q of P {
    for each tuple t in page Q {
        if (t.nk == val) add t to results
}   }
return results
```
We always have to check the overflow page. Costpmr  =  1 + Ov

Remarks:

-> Hashing does not help with range queries: Costrange = b + bov

-> Selection on attribute j which is not hash key: Costone,   Costrange,   Costpmr  =  b + bov

### Insertion with Hashing
```
// insert tuple t with key=val into rel R
(pid,P) = getPageViaHash(val,R) 
if room in page P {
    insert t into P; return
}
for each overflow page Q of P {
    if room in page Q {
        insert t into Q; return
}   }
add new overflow page Q
link Q to previous page
insert t into Q
```

Costinsert :    Best: 1r + 1w    Worst: 1+max(OvLen))r + 2w

### Deletion with Hashing
```
// delete from R where k = val
// f = data file ... ovf = ovflow file
(pid,P) = getPageViaHash(val,R)
ndel = delTuples(P,k,val)
if (ndel > 0) putPage(f,P,pid)
for each overflow page qid,Q of P {
    ndel = delTuples(Q,k,val)
    if (ndel > 0) putPage(ovf,Q,qid)
}
```
Extra cost over select is cost of writing back modified blocks. Method works for both unique and non-unique hash keys.

## Page Splitting
Important concept for flexible hashing: *splitting*
 - consider one page (all tuples have same hash value)
 - recompute page numbers by considering one extra bit
 - if current page is 101, new pages have hashes 0101 and 1101
 - some tuples stay in page 0101 (was 101)
 - some tuples move to page 1101 (new page)
 - also, rehash any tuples in overflow pages of page 101

## Linear Hashing
File organisation:
 - file of primary data blocks
 - file of overflow data blocks
 - a "register" called the **split pointer** (sp). It indicates the position within the hash table **where the next split (expansion) will occur**.

File grows linearly (one block at a time, at regular intervals). Has "phases" of expansion; over each phase, b doubles. For example, at the start of expansion phase k, the # of pages is 2^d, then at the start of expansion phase k+1, the # of pages is 2^(d+1) (doubles).

### Selection with Lin.Hashing
If b=2^d, the file behaves exactly like standard hashing. Use d  bits of hash to compute block address.
```
// select * from R where k = val
h = hash(val);
pid = bits(d,h);  // lower-order bits
P = getPage(f, pid)
for each tuple t in page P
         and its overflow pages {
    if (t.k == val) add t to Result;
}
```
Average Costone  =  1+Ov

If b != 2^d, treat different parts of the file differently.
```
 0                         2^d-1          b-1         2^(d+1)-1
 |      A      |      B      |      C      |      D      |
               |
              sp
```
-> Parts A and C are treated as if part of a file of size 2d+1.

-> Part B is treated as if part of a file of size 2d.

-> Part D does not yet exist (tuples in B may eventually move into it).

Remarks:
 - **Part A**: These are the buckets before the split pointer. They are treated as part of the new, larger hash table size of 2^(d+1). When hashing keys, we consider the hash table as if it has already doubled from size 2^d to 2^(d+1).

 - **Part B**: This represents the bucket at the split pointer, which is being split. Before splitting, it is considered part of the old table size 2^d. After splitting, one of the resulting buckets will become part of the new segment C.

 - **Part C**: These are the buckets that have been added during the current expansion phase. They are new buckets that resulted from splitting the buckets in part A and are treated as part of the hash table of size 2^(d+1).

 - **Part D**: This part does not yet exist. It represents future buckets that will be added as the expansion continues. Tuples from part B might be moved into these new buckets once they are created.

Modified search algorithm:
```
// select * from R where k = val
h = hash(val);
pid = bits(d,h);
if (pid < sp) { pid = bits(d+1,h); }
P = getPage(f, pid)
for each tuple t in page P
         and its overflow blocks {
    if (t.k == val) add t to Result;
}
```
