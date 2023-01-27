---
title: SQL Server seek predicate precedence
layout: post
author: Michael Nguyen
toc: true
---
When a table seek occurs, you may see something called "seek predicate" in execution plan:

Seek predicate is the predicate applied to the index in question.
You may also see predicate alongside it - that just means the seek predicate couldnt apply the full predicate using the index alone.
This happens because:
- predicate columns are not in the index
- the index column order is inefficient, meaning that the earlier columns arent cardinal enough
- the predicate does not use equality (e.g. myTable.myColumn <> 1)

So if the predicate is applied after the seek predicate, what order does the SQL Server optimizer choose for seek predicate?

It'll likely be based on predicate specificity. Consider below sql:

```sql
SELECT TOP 1 1
FROM dbo.apple a JOIN dbo.banana b ON a.col = b.col
WHERE a.col = 1
```
vs
```sql
SELECT TOP 1 1
FROM dbo.apple a JOIN dbo.banana b ON a.col = b.col
WHERE b.col = 1
```
Former case, seek predicate will be applied to dbo.apple, for a total of 1 row read. Then 0+ rows read from dbo.banana.
Later case, opposite.