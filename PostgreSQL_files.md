For the queries below, think about and describe the patterns of access to the data in the tables that would be required to answer them.

(a)
```
select max(id) from People;
```
Requires all People.id values to be accessed; potentially this would need a scan over all tuples in the relation, hence all pages would need to be read.

