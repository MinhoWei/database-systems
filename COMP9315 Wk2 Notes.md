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
 - 






















