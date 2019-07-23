---
title: SQL Server Physical Join Operations
layout: post
author: michael_nguyen
---
SQL Server has a logical operation for joining, but has 3 options of achieving the same thing.

These "options" or "physical operations" are:
1. Merge join
2. Hash join
3. Nested loop join

## Merge
This runs through 2 ordered datasets & goes down both datasets, comparing each item.

Both tables are scanned once.

Preferred when the 2 datasets are large.

## Hash
This runs a merge join.

Both tables are scanned once.

But if the 2 datasets cant be fetch and/or ordered efficiently enough, hash join substitutes sorted datasets with a dynamic hash function & hash table.

So its more expensive than a straight-up merge join, but at least it gets the job done.

## Nested Loop
This runs an outer + inner loop semantic.

For each row in 1st dataset, find a matching row in 2nd table.

The difference here is that instead of just scanning both tables once & merge joining them, the outer table is scanned once, but the inner table is SEEKED for every outer row.

You can see this in execution plan "number of executions" for index seek operator on inner table.

### Merge vs nested loop
Merge join is fast with 2 large datasets, because after finding a match it resumes from where it left off for BOTH datasets.

Merge join performance is based on the smallest dataset size.

Nested loop is fast with 1 large 1 small dataset, because it resumes from where it left off for only the OUTER dataset.

Performance of merge and nested loop is based on the smallest dataset size, but the "resume" semantic is what separates them.

Consider big dataset example:
```
Table1	| Table2
---	   	| ---
1      	| 1
2 	   	| 2
... 	| ...
9 		| 9
10 		| 10
		| 11
```

Merge join data flow:

* 1 <-> 1
* 2 <-> 2
* 3 <-> 3
* 4 <-> 4
* 5 <-> 5
* 6 <-> 6
* 7 <-> 7
* 8 <-> 8
* 9 <-> 9
* 10 <-> 10
* END (no point checking Table2, there will never be any matches in Table1 anyways since we hit end of it)

Nested loop join:

* 1 <-> 1
* 1 <-> 2
* 1 <-> 3
* 1 <-> 4
* 1 <-> 5
* 1 <-> 6
* 1 <-> 7
* 1 <-> 8
* 1 <-> 9
* 1 <-> 10
* 2 <-> 1
* 2 <-> 2
* 2 <-> 3
* ...
* 10 <-> 7
* 10 <-> 8
* 10 <-> 9
* 10 <-> 10

For small datasets, not too much performance difference between merge and nested loop join.

But theres no way you can tell me that a nested loop join is faster than a merge join on the above dataset. Especially considering if merge join scans both tables once, and nested loop join scans Table1 once and Table2 10x10=100 times.

## Performance tuning
If you see a hash join, try to upgrade it to a merge join by introducing an index that sorts on the predicate/join condition

If you see a nested loop join with 2 large datasets, ensure table stats are updated by checking the execution plan estimated VS actual
