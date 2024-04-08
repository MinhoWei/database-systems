 # Cost Models
Cost

**Time cost**: total time taken to execute a method (we do not consider this)

**Page cost**: # of pages read / written

**Page** := fixed-size block of data -> the size is determined by storage medium

Terminologies:

r: # of tuples in a relation

b: # of pages

B: size of page

c: max # of tuples in a page

The tuple which answer query q are contained in b<sub>q</sub> pages

T<sub>r</sub>: page read time (high)

T<sub>w</sub>: page write time (high)

# Single-File DBMS (e.g. SQLite)

Objects are allocated to regions (segments) of a single file:

|params|catalog|update logs|table1| ... |table2| ... |catalog (cont.)|index for table1| ... |table1 (cont.)|...

Remarks:
 - If an object grows too large for allocated segments, allocate an extension
 - When objects are removed, the allocated space will: -> be reused for new data -> left vacant -> compacted (reclaim the space and reduce the size)

An example single-file DBMS layout:
```
| SpaceMap | TableMap | Employee Data Pages | Project Data Pages | ...
[0]       [10]       [20]                 [620]                 [720]
```
**SpaceMap** = [(offset, #pages, status), ...] =  [ (0,10,U), (10,10,U), (20,600,U), (620,100,U), (720,20,F) ]

E.G. (20,600,U) means that Employee Data Pages starts at page index 20, has 600 pages, the status is U (used)

**TableMap** = [(name, offset, #pages)...] = [ ("employee",20,500), ("project",620,40) ]

E.G. ("employee",20,500) means that "employee" relation starts at page index 20, 500 represents the number of pages that the employee table occupies in the file

Note: *page offset = page id * PAGESIZE*

## Storage manager (single file) data structures
```
typedef struct DBrec {
   char *dbname;    // copy of database name
   int fd;          // the database file
   SpaceMap map;    // map of free/used areas 
   NameTable names; // map names to areas + sizes
} *DB;

typedef struct Relrec {
   char *relname;   // copy of table name
   int   start;     // page index of start of table data
   int   npages;    // number of pages of table data
   ...
} *Rel;
```

With the above definition, the query 'select name from Employee;' might be inplemented as:
```
DB db = openDatabase("myDB");
Rel r = openRelation(db,"Employee");
Page buffer = malloc(PAGESIZE*sizeof(char));

for (int i = 0; i < r->npages; i++) {
   PageId pid = r->start+i;            // page id = file id + offset, start is the page index at the start of table data
   get_page(db, pid, buffer);          // read the page from DB to buffer
   for each tuple in buffer {
      get tuple data and extract name
      add (name) to result tuples
   }
}
```

The **openDatabase** implementation:
```
DB openDatabase(char *name) { 
   DB db = new(struct DBrec);
   db->dbname = strdup(name);
   db->fd = name,open(O_RDWR);
   db->map = readSpaceTable(db->fd);
   db->names = readNameTable(db->fd);
   return db;
}
```

The **closeDatabase** implementation:
```
void closeDatabase(DB db) {
   writeSpaceTable(db->fd,db->map);
   writeNameTable(db->fd,db->map);
   fsync(db->fd);   // fsync() flushes contents of file buffers to disk
   close(db->fd);
   free(db->dbname);
   free(db);
}
```

The **openRelation** implementation:
```
Rel openRelation(DB db, char *rname) {
   Rel r = new(struct Relrec);
   r->relname = strdup(rname);
   // get relation data from map tables
   r->start = ...;
   r->npages = ...;
   return r;
}
```

The **closeDatabase** implementation:
```
void closeRelation(Rel r) {
   free(r->relname);
   free(r);
}
```

The **get_page** (read page from file to buffer) implementation:
```
void get_page(DB db, PageId p, Page buf) {
   lseek(db->fd, p*PAGESIZE, SEEK_SET);    // moves the file descripter (fd) to the correct position
                                           // p*PAGESIZE = page offset
                                           // SEEK_SET indicates the offset to be measured from the beginning of the file
   read(db->fd, buf, PAGESIZE);
}
```

The **put_page** (write page from buffer to file) implementation:
```
void put_page(DB db, PageId p, Page buf) {
   lseek(db->fd, p*PAGESIZE, SEEK_SET);
   write(db->fd, buf, PAGESIZE);
}
```

The **allocate_pages** (allocate n new pages) implementation:
```
PageId allocate_pages(DB db, int n) {
   if (no existing free chunks are large enough) {
      int endfile = lseek(db->fd, 0, SEEK_END);  // SEEK_END: file descriptor should be moved to a position relative to the end of a file
      addNewEntry(db->map, endfile, n);
   } else {
      grab "worst fit" chunk
      split off unused section as new chunk
   }
}
```

The **deallocate_pages** (drop n pages starting from p) implementation:

(changes take effect when closeDatabase() executed)
```
void deallocate_pages(DB db, PageId p, int n) {
   if (no adjacent free chunks) {
      markUnused(db->map, p, n);
   } else {
      merge adjacent free chunks
      compress mapping table
   }
}
```

# Multi-File DBMS
| NameMap |

| Table 1 pages | ...

| Table 2 pages | ...

| Table 3 pages | ...

*page[i] offset = i * pagesize*

Two basic kinds of files:
 - **heap files**: containing data (tuples)
 - **index files**: containing index entries

## Relations as Files
PostgreSQL identifies relation files via their OIDs. The core data structure for this is **RelFileNode**:

```
typedef struct RelFileNode {
    Oid  spcNode;  // tablespace
    Oid  dbNode;   // database
    Oid  relNode;  // relation
} RelFileNode;
```

Note: Global (shared) tables (e.g. pg_database) have: -> spcNode == GLOBALTABLESPACE_OID -> dbNode == 0

The **relpath** function maps **RelFileNode** to file:

```
char *relpath(RelFileNode r)  // simplified
{
   char *path = malloc(ENOUGH_SPACE);

   if (r.spcNode == GLOBALTABLESPACE_OID) {
      /* Shared system relations live in PGDATA/global */
      Assert(r.dbNode == 0);
      sprintf(path, "%s/global/%u",
              DataDir, r.relNode);
   }
   else if (r.spcNode == DEFAULTTABLESPACE_OID) {
      /* The default tablespace is PGDATA/base */
      sprintf(path, "%s/base/%u/%u",
              DataDir, r.dbNode, r.relNode);
   }
   else {
      /* All other tablespaces accessed via symlinks */
      sprintf(path, "%s/pg_tblspc/%u/%u/%u", DataDir
              r.spcNode, r.dbNode, r.relNode);
   }
   return path;
}
```

## 














