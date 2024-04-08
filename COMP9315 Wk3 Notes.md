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












