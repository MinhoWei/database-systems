# Tuples
**Tuple** = collection of attribute values for such a schema, e.g.

(33357462, 'Neil Young', 'Musician', 0277)

**Record** = sequence of bytes, containing data for one tuple, e.g.

| 01101001 | 11001100 | ...

Bytes need to be interpreted relative to schema to get tuple

Common operation on records:

-> access record via RecordId:
```
Record get_record(Relation rel, RecordId rid) {
    (pid,tid) = rid;
    Page *buf = request_page(rel, pid);
    return get_tuple(buf, tid);
}
```

-> Gives a sequence of bytes, which needs to be interpreted, e.g.
```
Relation rel = ... // relation schema
Record rec = get_record(rel,rid)
Tuple t = makeTuple(rel,rec)
```

Operation on tuples:

-> Once we have a record, we need to interpret it as a tuple ...
```
Tuple t = makeTuple(Relation rel, Record rec)
```

-> Once we have a tuple, we want to examine its contents ...
```
Typ   getTypField(Tuple t, int fieldNum)
```

## Scanning
Access methods typically involve *iterators*, e.g.
```
Scan s = start_scan(Relation r, ...)
```
commence a scan of relation r

```
Tuple next_tuple(Scan s)
```
returns next Tuple after last accessed one, returns NULL if no more Tuples left in the relation

For example, a simple scan of the table 'select name from Emplyees' could be implemented as:
```
DB db = openDatabase("myDB");
Relation r = openRel(db,"Employee");
Scan s = start_scan(r);
Tuple t;  // current tuple

while ((t = next_tuple(s)) != NULL)
{
   char *name = getStrField(t,2);
   printf("%s\n", name);
}
```

A possible **Scan** data structure:
```
typedef struct {
   Relation rel;
   Page     *curPage;  // Page buffer
   int      curPID;    // current pid
   int      curTID;    // current tid
} ScanData;
```

A **Scan** object in the context of databases is a data structure that keeps track of the state of a scan operation on a database table (relation).

The Scan data structure (which may be called ScanData in some implementations) usually contains at least the following information:
 - Relation: Reference to the relation (table) being scanned.
 - curPage: A pointer to a buffer holding the current page of data being scanned. This is where actual tuples are stored.
 - curPID: The current page ID within the relation, which indicates which page is currently in the buffer.
 - curTID: The current tuple ID within the current page, which indicates which tuple is currently being processed.

The **start_scan** function:
```
Scan start_scan(Relation rel)
{
	Scan new = malloc(ScanData);
	new->rel = rel;
	new->curPage = get_page(rel,0);
	new->curPID = 0;
	new->curTID = 0;
	return new;
}
```

The **next_tuple** function:
```
Tuple next_tuple(Scan s)
{
	if (s->curTID == nTuples(s->curPage)) {
		// finished cur page, get next
		if (s->curPID == nPages(s->rel))
			return NULL;
		s->curPID++;
		s->curPage = get_page(s->rel, s->curPID);
		s->curTID =0;
	}
	Record r = get_record(s->curPage, s->curTID);
	s->curTID++;
	return makeTuple(s->rel, r);
}
```

Implements iterator data/operations:
 - HeapScanDesc ... struct containing iteration state
 - scan = heap_beginscan(rel,...,nkeys,keys)
 - tup = heap_getnext(scan, direction)
 - heap_endscan(scan) ... frees up scan struct
 - res = HeapKeyTest(tuple,...,nkeys,keys) -> performs ScanKeys tests on tuple ... checks is it a result tuple?

**HeapScanDescData**
```
typedef HeapScanDescData *HeapScanDesc;

typedef struct HeapScanDescData
{
  // scan parameters 
  Relation      rs_rd;        // heap relation descriptor 
  Snapshot      rs_snapshot;  // snapshot ... tuple visibility 
  int           rs_nkeys;     // number of scan keys 
  ScanKey       rs_key;       // array of scan key descriptors 
  ...
  // state set up at initscan time 
  PageNumber    rs_npages;    // number of pages to scan 
  PageNumber    rs_startpage; // page # to start at 
  ...
  // scan current state, initally set to invalid 
  HeapTupleData rs_ctup;      // current tuple in scan
  PageNumber    rs_cpage;     // current page # in scan
  Buffer        rs_cbuf;      // current buffer in scan
   ...
} HeapScanDescData;
```

The **HeapScanDescData** structure is used to keep track of the state and configuration of a scan operation on a heap-organized table within a database system. Its purpose is to maintain all necessary information for the scan so that the database system can efficiently continue the operation from one call to the next without reevaluating the initial conditions.

A **tuple** can be described as (C struct):

(ushort is unsigned short integer)
```
typedef struct {
  ushort    nfields;   // number of fields/attrs
  ushort    data_off;  // offset in struct for data
  FieldDesc fields[];  // field descriptions
  Record    data;      // pointer to record in buffer
} Tuple;
```
Where: fields[]: An array of FieldDesc structs (not shown in full here), where each FieldDesc gives the details (offset, length, type) of a field within the tuple.
 - offset = offset of field within record data
 - length = length (in bytes) of field
 - type = data type of field

For example:
```
   nfields   data_off   fields = FieldDesc[4]
|     4    |    16    | (0, 4, int) | (6, 10, char) | (18, 8, char) | (28, 2, int) | Pointer -> elsewhere
```

Tuple data could be: 
 - a pointer to bytes stored elsewhere in memory
 - appended to Tuple struct   (used widely in PostgreSQL)

Tuple-related data types:
```
// representation of a data value
typedef uintptr_t Datum;
```
Datum is a type that represents a generic data value. uintptr_t is an unsigned integer type that is guaranteed to be the same size as a pointer, which makes Datum flexible enough to store various data types, from integers to pointers to actual data structures.

```
// TupleDescData: schema-related information for HeapTuples
typedef struct tupleDescData
{
  int         natts;          // number of attributes in the tuple 
  Oid         tdtypeid;       // composite type ID for tuple type 
  int32       tdtypmod;       // typmod for tuple type 
  int         tdrefcount;     // reference count (-1 if not counting)
  TupleConstr *constr;        // constraints, or NULL if none 
  FormData_pg_attribute attrs[FLEXIBLE_ARRAY_MEMBER];
  // attrs[N] is description of attribute number N+1 
} *TupleDesc;
```
**TupleDescData**: Defines the *schema* for a tuple, including the number of attributes, type information, constraints, and a flexible array member for attribute descriptors.

```
// HeapTupleData contains information about a stored tuple
typedef HeapTupleData *HeapTuple;

typedef struct HeapTupleData
{
  uint32           t_len;  // length of *t_data 
  ItemPointerData t_self;  // SelfItemPointer 
  Oid         t_tableOid;  // table the tuple came from 
  HeapTupleHeader t_data;  // -> tuple header and data 
} HeapTupleData;
// HeapTupleHeader is a pointer to a location in a buffer
```
**HeapTupleData**: Represents a tuple within a heap table, including its length, location, originating table OID, and a pointer to its header and data.

```
typedef struct HeapTupleHeaderData // simplified
{
  HeapTupleFields t_heap;
  ItemPointerData t_ctid;      // TID of this tuple or newer version
  uint16          t_infomask2; // #attributes + flags
  uint16          t_infomask;  // flags e.g. has_null, has_varwidth
  uint8           t_hoff;      // sizeof header incl. bitmap+padding
  // above is fixed size (23 bytes) for all heap tuples
  bits8           t_bits[1];   // bitmap of NULLs, variable length
  // OID goes here if HEAP_HASOID is set in t_infomask
  // actual data follows at end of struct
} HeapTupleHeaderData;
```
**HeapTupleHeaderData**: Contains metadata for a heap tuple, such as transaction information, a bitmap for NULL values, and the size and flags of the header.

```
typedef struct HeapTupleFields  // simplified
{
  TransactionId t_xmin;  // inserting xact ID
  TransactionId t_xmax;  // deleting or locking xact ID
  union {
    CommandId   t_cid;   // inserting or deleting command ID
    TransactionId t_xvac;// old-style VACUUM FULL xact ID 
  } t_field3;
} HeapTupleFields;
```
**HeapTupleFields**: Holds transaction control information for a tuple, detailing the transactions responsible for inserting and potentially deleting the tuple.

## Cost analysis
In counting reads and writes, assume minimal buffering
 - each *request_page()* causes a read
 - each *release_page()* causes a write (if page is dirty)

Consider three simple file structures:
 - **heap file** ... tuples added to any page which has space
 - **sorted file** ... tuples arranged in file in key order
 - **hash file** ... tuples placed in pages using hash function
 - Note: Some records in each page may be marked as "deleted".

## Cost analysis 1
Consider the query: 'select * from Rel;' (scanning)

Operational view:
```
for each page P in file of relation Rel {
   for each tuple t in page P {
      add tuple t to result set
   }
}
```
Cost: read every data page once

Cost = b     (b page reads + ≤b page writes)

## Cost analysis 2

Scanning with overflow pages:
```
for each page P in file of relation T {
    for each tuple t in page P {
        add tuple t to result set
    }
    for each overflow page V of page P {
        for each tuple t in page V {
            add tuple t to result set
}   }   }
```
Cost: read each data and overflow page once

Cost = b + b<sub>Ov</sub>, where b<sub>Ov</sub> = total number of overflow pages

## Cost analysis 3
Selection via scanning: 'select * from Employee where id = 762288;'
```
for each page P in relation Employee {
    for each tuple t in page P {
        if (t.id == 762288) return t
}   }
```
Cost analysis for one searching in unordered file
 - best case: read one page, find tuple (1)
 - worst case: read all b pages, find in last (or don't find) (b)
 - average case: read half of the pages (b/2)

## Cost analysis 4
Relation copying: 'create table T as (select * from S);'
```
s = start scan of S
make empty relation T
while (t = next_tuple(s)) {
    insert tuple t into relation T
}
```
Note: Possible that T is smaller than S -> may be unused free space in S where tuples were removed -> if T is built by simple append, will be compact

In terms of existing relation/page/tuple operations:
```
Relation in;       // relation handle (incl. files)
Relation out;      // relation handle (incl. files)
int ipid,opid,tid; // page and record indexes
Record rec;        // current record (tuple)
Page ibuf,obuf;    // input/output file buffers

in = openRelation("S", READ);
out = openRelation("T", NEW|WRITE);
clear(obuf);  opid = 0;
for (ipid = 0; ipid < nPages(in); ipid++) {
    get_page(in, ipid, ibuf);
    for (tid = 0; tid < nTuples(ibuf); tid++) {
        rec = get_record(ibuf, tid);
        if (!hasSpace(obuf,rec)) {
            put_page(out, opid++, obuf);
            clear(obuf);
        }
        insert_record(obuf,rec);
}   }
if (nTuples(obuf) > 0) put_page(out, opid, obuf);
```

# The Sort Operation
## Two-Way Merge Sort
```
input file:  |7, 3| |1, 2| |3, 5| |6, 4|
1-page runs: |3, 7| |1, 2| |3, 5| |4, 6|
2-page runs: |1, 2|-|3, 7| |3, 4|-|5, 6|
Output file: |1, 2|-|3, 3| |4, 5|-|6, 7|
```

Remarks:
 - Two-Way Merge Sort requires 3 in-memory buffers (2 for input and 1 for output)
 - Need a function **tupCompare(r1,r2,f)**:
```
int tupCompare(r1,r2,f)
{
   if (r1.f < r2.f) return -1;
   if (r1.f > r2.f) return 1;
   return 0;
}
```

Sketch of multi-attribute sorting function:
```
int tupCompare(r1,r2,criteria)
{
   foreach (f,ord) in criteria {
      if (ord == ASC) {
         if (r1.f < r2.f) return -1;
         if (r1.f > r2.f) return 1;
      }
      else {
         if (r1.f > r2.f) return -1;
         if (r1.f < r2.f) return 1;
      }
   }
   return 0;
}
```

Cost of Two-way Merge Sort:

For a file containing b data pages:
 - require ceil(log2b) passes to sort,
 - each pass requires b page reads, b page writes
 - Gives total cost:   2b*ceil(log2b)

## n-Way Merge Sort
Suppose we use B total buffers, that is, n = B - 1 input buffers, 1 output buffer

Method:
```
// Produce B-page-long runs
for each group of B pages in Rel {
    read B pages into memory buffers
    sort group in memory
    write B pages out to Temp
}
// Merge runs until everything sorted
numberOfRuns = ⌈b/B⌉
while (numberOfRuns > 1) {
    // n-way merge, where n=B-1
    for each group of n runs in Temp {
        merge into a single run via input buffers
        write run to newTemp via output buffer
    }
    numberOfRuns = ⌈numberOfRuns/n⌉
    Temp = newTemp // swap input/output files
}
```

Cost of n-way Merge Sort:

For b data pages and B buffers:
 - first pass: read/writes b pages, gives b0 = ⌈b/B⌉ runs
 - then need ⌈lognb0⌉ passes until sorted, where n = B-1
 - each pass reads and writes b pages   (i.e. 2*b page accesses)
 - Cost = 2b*(1 + ⌈lognb0⌉),   where b0 = ⌈b/B⌉ and n = B-1

Disk-based sort has phases:
 - divide input into sorted runs using HeapSort
 - merge using N buffers, one output buffer
 - N = as many buffers as workMem allows

Sorting comparison operators are obtained via catalog (in Type.o):
```
// gets pointer to function via pg_operator
struct Tuplesortstate { ... SortTupleComparator ... };

// returns negative, zero, positive
ApplySortComparator(Datum datum1, bool isnull1,
                    Datum datum2, bool isnull2,
                    SortSupport sort_helper);
```

# Implementing Projection
(Note: duplicate tuples are eliminated during projection)

## Sort-based Projection
Requires a temporary file/relation (Temp):
```
for each tuple T in Rel {
    T' = mkTuple([attrs],T)
    write T' to Temp
}

sort Temp on [attrs]

for each tuple T in Temp {
    if (T == Prev) continue
    write T to Result
    Prev = T
}
```

Cost of Sort-based Projection:

The costs involved are (assuming B=n+1 buffers for sort):

 - scanning original relation Rel:   bR   (with cR)
 - writing Temp relation:  bT     (smaller tuples, cT > cR, sorted)
 - sorting Temp relation: 2.bT.(1+ceil(lognb0)) where b0 = ceil(bT/B)
 - scanning Temp, removing duplicates:   bT
 - writing the result relation:   bOut     (maybe less tuples)
 - Cost = sum of above = bR + bT + 2.bT.(1+ceil(lognb0)) + bT + bOut

## Hash-based Projection
Two phases:
 - Partitioning phase
 - Duplicate elimination phase

Algorithm for both phases:
```
for each tuple T in relation Rel {
    T' = mkTuple([attrs],T)
    H = h1(T', n)
    B = buffer for partition[H]
    if (B full) write and clear B
    insert T' into B
}
for each partition P in 0..n-1 {
    for each tuple T in partition P {
        H = h2(T, n)
        B = buffer for hash value H
        if (T not in B) insert T into B
        // assumes B never gets full
    }
    write and clear all buffers
}
```

Cost of Hash-based Projection

The total cost is the sum of the following:
 - scanning original relation R:   bR
 - writing partitions:   bP   (bR vs bP ?)
 - re-reading partitions:   bP
 - writing the result relation:   bOut
 - Cost = bR + 2bP + bOut

## Projection on Primary Key
No duplicates, so the above approaches are not required.

Method:
```
bR = nPages(Rel)
for i in 0 .. bR-1 {
   P = read page i
   for j in 0 .. nTuples(P) {
      T = getTuple(P,j)
      T' = mkTuple(pk, T)
      if (outBuf is full) write and clear
      append T' to outBuf
   }
}
if (nTuples(outBuf) > 0) write
```

(Comparing to previous methods, no need to remove duplicates)

## Index-only Projection





























