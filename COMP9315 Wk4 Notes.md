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















































