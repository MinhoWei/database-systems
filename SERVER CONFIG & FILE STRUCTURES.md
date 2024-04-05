# SERVER CONFIG

Most configuration parameters can be set by modifying the *$PGDATA/postgresql.conf* file and restarting the server. The standard PostgreSQL mechanism for starting the server is yet another script, called *pg_ctl*:
```
pg_ctl start
pg_ctl stop
pg_ctl status
```

The *pgs* script simply invokes *pg_ctl* to start a server, with some extra options:
```
pg_ctl -w start -l /localstorage/$USER/pgsql/data/log
```

Several options:
 - The **-l** option tells the PostgreSQL server which file to use to write its log messages.
 - The **-w** option tells pg_ctl to wait until the server has actually started properly before returning.
 - To pass configuration parameters, you use the **-o** option and a single string containing all the server parameters.
 - **-B** parameter to postgres lets you say how many shared memory buffers the server should use.

For example, you could start postgres and get it to use just 16 buffers as follows:
```
pg_ctl start -o '-B 16' -l /localstorage/$USER/pgsql/log
```





For the queries below, think about and describe the patterns of access to the data in the tables that would be required to answer them.

(a)
```
select max(id) from People;
```
Requires all People.id values to be accessed; potentially this would need a scan over all tuples in the relation, hence all pages would need to be read.

PostgreSQL makes a B-tree index on all primary keys. We could use **EXPLAIN** to show the execution plan that the PostgreSQL optimizer has chosen for the query. (1) If the plan includes **Index Scan** or **Index Only Scan** on your primary key index, PostgreSQL is using the B-tree index to efficiently find the largest value. (2) If the plan shows **Seq Scan** (sequential scan), it means PostgreSQL is planning to scan the entire table to find the largest value.

(b) 
```
select min(birthday) from People;
```
Requires all People.birthday values to be accessed. Because there is no index on this attribute.

(c)
```
select max(maxmark) from items;
```
Requires all Items.maxmark values to be accessed. Because there is no index on this attribute.

(d)
```
select c.code, i.name, i.maxmark
from   Courses c, Items i
where  c.id = i.course;
```
Requires a join on the Courses and Items table; each tuple in each table will need to be accessed, possibly multiple times; in the worst-case scenario, we would read the Courses table once and read the entire Items table for each page in the Courses table.

_Note: You can compare the time taken for the first two queries by turning on psql's timing mechanism, using the command \timing on. If you run the first two queries a few times you will observe that (a) the time value is different each time (thanks to the way times are computed), (b) that the first query generally takes less time than the second._

# How Data is Stored in the File System

All data is for a given database is stored under the directory (folder):

**$PGDATA/base/OID**

Where:
- PGDATA=/localstorage/$USER/pgsql/data (the location of the PostgreSQL data directory as set in your env file)
- OID is the unique internal id of the database from the pg_database table

The following SQL query will help you work out what is the OID for your database:
```
select oid, datname from pg_database;
```
Write a query to print the number of data pages in each relation. ("Data pages" refers to the basic unit of data storage on disk. A "page" is a fixed-length block of this storage)
```
select c.relname,c.relpages
from   pg_class c, pg_namespace n
where  c.relkind='r' and c.relnamespace=n.oid and n.nspname='public';
```
Once you've got the page counts in the catalog, check that they're consistent with the file sizes in the directory for the uni database (assuming an 8KB page size).

Below is an example of how to do this:
```
select oid,relpages from pg_class where relname='courses';
\q
ls -l NNNNN -- NNNNN is the oid of the course table
bc -l -- this is a calculator
```
