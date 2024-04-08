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

## File Descriptor Pool
Unix has limits on the number of concurrently open files. PostgreSQL maintains a pool of open file descriptors:
 - to hide this limitation from higher level functions
 - to minimise expensive open() operations

Interface to file descriptor (pool):

```
File FileNameOpenFile(FileName fileName,
                      int fileFlags, int fileMode);
     // open a file in the database directory ($PGDATA/base/...)
File OpenTemporaryFile(bool interXact);
     // open temp file; flag: close at end of transaction?
void FileClose(File file);
void FileUnlink(File file);
int  FileRead(File file, char *buffer, int amount);
int  FileWrite(File file, char *buffer, int amount);
int  FileSync(File file);
long FileSeek(File file, long offset, int whence);
int  FileTruncate(File file, long offset);
```

**Virtual file descriptors (Vfd)**

-> physically stored in dynamically-allocated array
```
VfdCache   [0]        [1]         [2]        ...
            | - , -    |fd, pos... |fd, pos...
                      Vfd         Vfd       
```
-> also arranged into list by recency-of-use
-> VfdCache[0] holds list head/tail pointers.

Virtual file descriptor records (simplified):
```
typedef struct vfd
{
    s_short  fd;              // current FD, or VFD_CLOSED if none
    u_short  fdstate;         // bitflags for VFD's state
    File     nextFree;        // link to next free VFD, if in freelist
    File     lruMoreRecently; // doubly linked recency-of-use list
    File     lruLessRecently;
    long     seekPos;         // current logical file position
    char     *fileName;       // name of file, or NULL for unused VFD
    // NB: fileName is malloc'd, and must be free'd when closing the VFD
    int      fileFlags;       // open(2) flags for (re)opening the file
    int      fileMode;        // mode to pass to open(2)
} Vfd;
```

PostgreSQL stores each table:
 - in the directory PGDATA/pg_database.oid
 - often in multiple data files (aka *forks*)

Possible files for a single PostgreSQL table with *pg_class.relfilename = Oid*:

Oid     | table data pages |

Oid.1   | more table data pages |

Oid_fsm |free space map|

Oid_vm  |visibility map|

Remarks:
 - **Free space map**: -> indicates where free space is in data pages -> "free" space is only free after VACUUM
 - **Visibility map**: -> indicates pages where *all* tuples are "visible" -> such pages can be ignored by VACUUM

PostgreSQL **PageID** values are structured:

*PageID = BufferTag = (rnode, forkNum, blockNum)*
```
typedef struct
{
    RelFileNode rnode;    // which relation/file
    ForkNumber  forkNum;  // which fork (of reln)
    BlockNumber blockNum; // which page/block 
} BufferTag;
```

Access to a block of data proceeds (roughly) as follows:
```
getBlock(BufferTag pageID, Buffer buf)
{
   Vfd vf;  off_t offset;
   (vf, offset) = findBlock(pageID)
   lseek(vf.fd, offset, SEEK_SET)
   vf.seekPos = offset;
   nread = read(vf.fd, buf, BLOCKSIZE)
   if (nread < BLOCKSIZE) ... we have a problem
}
```

The **findBlock** function:
```
findBlock(BufferTag pageID) returns (Vfd, off_t)
{
   offset = pageID.blockNum * BLOCKSIZE
   fileName = relpath(pageID.rnode)
   if (pageID.forkNum > 0)
      fileName = fileName+"."+pageID.forkNum
   if (fileName is not in Vfd pool)
      fd = allocate new Vfd for fileName
   else
      fd = use Vfd from pool
   if (offset > fd.fileSize) {
      fd = allocate new Vfd for next fork
      offset = offset - fd.fileSize
   }
   return (fd, offset)
}
```

# Buffer Pool

Data structures:
 - Page frames[NBUFS] (frames is an array of pages)
 - FrameData directory[NBUFS] (directory stores information / metadata about each frame in the buffer pool)
 - Page is byte[BUFSIZE] (a page is just a sequence of bytes)

For each frame, we need to know:   (FrameData)
 - which Page it contains, or whether empty/free
 - whether it has been modified since loading (*dirty bit*)
 - how many transactions are currently using it (*pin count*)
 - time-stamp for most recent access (assists with replacement)

How scans are performed with Buffer Pool:
```
Buffer buf;
int N = numberOfBlocks(Rel);
for (i = 0; i < N; i++) {
   pageID = makePageID(db,Rel,i);
   bufID = request_page(pageID);
   buf = frames[bufID]
   for (j = 0; j < nTuples(buf); j++)
      process(buf, j)
   release_page(pageID);
}
```
Requires N page reads on the first pass.

If we read it again, 0 ≤ page reads ≤ N (some / all of the pages may still in the buffer pool. That would not need to be read from disk to buffer again.

Implementation of **request_page()**:
 - It first checks if the requested page is already present in the buffer pool. If it is, the function does not need to read the page from the disk again (just return bufID).
 - If the page is found in the pool, the function sets bufID to the index of the frame where the page is located.
```
int request_page(PageID pid)
{
   if (pid in Pool)
      bufID = index for pid in Pool
   else {
      if (no free frames in Pool)
         evict a page // free a frame
      bufID = allocate free frame
      directory[bufID].page = pid
      directory[bufID].pin_count = 0
      directory[bufID].dirty_bit = 0
   }
   directory[bufID].pin_count++
   return bufID
}
```

Evicting a page:
 - find frame(s) preferably satisfying: pin count = 0 and dirty bit = 0
 - if selected frame was modified, flush frame to disk
 - flag directory entry as "frame empty"

## Page Replacement Policies
Several schemes are commonly in use:
 - Least Recently Used (LRU)
 - Most Recently Used (MRU)
 - First in First Out (FIFO)
 - Random

Cost analysis for page replacement:

(a) sequential scan, LRU or MRU, n ≥ b: First scan costs b  reads; subsequent scans are "free" (the data already loaded into buffer and no need to read from disk any more).

(b) sequential scan, MRU, n < b: First scan costs b reads; subsequent scans cost b - n  reads.

(b) sequential scan, LRU, n < b: All scans cost b  reads; known as **sequential flooding**.

Consider a query to find customers who are also employees:
```
select c.name
from   Customer c, Employee e
where  c.ssn = e.ssn;
```
This might be implemented inside the DBMS via nested loops:
```
for each tuple t1 in Customer {
    for each tuple t2 in Employee {
        if (t1.ssn == t2.ssn)
            append (t1.name) to result set
    }
}
```
In terms of page-level operations, the algorithm looks like:
```
Rel rC = openRelation("Customer");
Rel rE = openRelation("Employee");
for (int i = 0; i < nPages(rC); i++) {
    PageID pid1 = makePageID(db,rC,i);
    Page p1 = request_page(pid1);
    for (int j = 0; j < nPages(rE); j++) {
        PageID pid2 = makePageID(db,rE,j);
        Page p2 = request_page(pid2);
        // compare all pairs of tuples from p1,p2
        // construct solution set from matching pairs
        release_page(pid2);
    }
    release_page(pid1);
}
```
















