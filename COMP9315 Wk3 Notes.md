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
































