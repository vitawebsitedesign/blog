---
title: T-SQL offset & fetch IO performance
layout: post
author: Michael Nguyen
toc: true
---
When using offset/fetch in query, execution plan shows query optimizer leveraging that info to minimise rows read.

Example:

```sql
SELECT 1
FROM dbo.TableA a JOIN dbo.TableB b ON a.id = b.id
OFFSET 0 ROWS FETCH NEXT 100 ROWS ONLY;
```

Execution plan will verify 100 actual rows read from both tables, then doing join.

But how smart is optimizer if we throw in a predicate, such that actual rows is larger than FETCH NEXT?

```sql
SELECT 1
FROM dbo.TableA a JOIN dbo.TableB b ON a.id = b.id
WHERE a.col = 1
OFFSET 0 ROWS FETCH NEXT 100 ROWS ONLY;
```

Assuming row count of "TableA with predicate" is more than 100, execution plan will show optimizer reading more rows than necessary.

* 100+ rows from TableA
* N rows from TableB
* Where N = rows read from TableA

So if 16000 rows from TableA satisfy predicate, then 16000 will also be read from TableB prior to join

I always hoped query optimizer was a magical unicorn able to make all code fast, but my hopes have been dashed here.

## Rows read VS actual rows
Execution plan shows these 2 metrics - important to know difference prior to tuning our query:

* Rows read = rows scanned/seeked prior to predicate (i.e.: seek predicate operation)
* Actual rows = rows returned by operator (i.e.: predicate operation)

This means actual rows will be less or equal to rows read.

So in above situation, TableB rows read = TableA rows read

We can do better: TableB rows read = TableA actual rows

## Tuning the query IO performance
We can step in and minimize rows read from TableB by splitting the query to:

Below example use table variable, but you can use temp table based on your situation.

```sql
DECLARE @rowsA TABLE (
	id INT PRIMARY KEY
);
INSERT INTO @rowsA (id)
	SELECT id
	FROM dbo.TableA
	WHERE col = 1;

SELECT 1
FROM @rowsA a JOIN dbo.TableB b ON a.id = b.id
OFFSET 0 ROWS FETCH NEXT 100 ROWS ONLY;
```

This will result in 100+ rows from TableA, but only <actual rows read from tableA> rows read from TableB.

2 important things to note here:

* We use table variable instead of temp table to avoid stats calculation overhead
* The table variable means an extra insert operation, which means more overhead

But if large datasets, overheads are worth it.
