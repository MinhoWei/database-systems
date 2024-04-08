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
```





















