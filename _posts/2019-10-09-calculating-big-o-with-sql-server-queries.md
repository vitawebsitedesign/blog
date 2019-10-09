---
title: Calculating Big O notation for SQL Server queries
layout: post
author: Michael Nguyen
---
The best way to verify algorithm scalability is by expressing the performance through a mathematical formulae. The 3 common weapons of choice are:

* Big Omega: a formulae that defines both the lower bound of scalability
* Big O: a formulae that defines both the upper bound of scalability
* Big Theta: a formulae that defines both the lower & upper bounds of scalability

This post will run through Big O by analysing SQL Server execution plans. I'm starting with a fresh db, so if you feel like a fat slob and have cheeseball stains on your shirt, get some finger exercise by following along:
```sql
CREATE DATABASE big_o;
GO
USE big_o;
GO
```

## O(1)
In the context of database queries, O(1) means that 1 read is executed, regardless of the number of rows in a table. E.g.: seeking 1 row by using a predicate against an indexed column:
```sql
CREATE TABLE dbo.items (
	id INT IDENTITY PRIMARY KEY
	,val INT NOT NULL
);
GO

INSERT INTO dbo.items (val) VALUES (1);
GO 100

SELECT id, val
FROM dbo.items
WHERE id = 10
```

The sql above creates a blank table with 100 rows, then runs a query with a predicate usable by the implicit clustered index.

![O1](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(1).jpg "O1")

And we can prove this! The execution plan for this query identifies only 1 "Number of Rows Read", & this table has 100 rows.

Since the database engine only reads 1 row (regardless of the number of rows in the table), we can say that the worst-case performance for this query is O(1).

But what would happen if a table scan was done instead? The db would have to read all rows in our table (100 reads). If there was 200 rows, 200 reads would occur. If 400 rows, 400 reads would occur and so forth.

And we can express such table scans as a math formulae as well.

## O(n)
A (non-seeked) table scan means that database goes to the first leaf node in the clustered binary tree index, then reads all the rows by advancing through all leaf nodes via the "next" pointer (which exists in each leaf node). This semantic is similar to reading a linked list from head to tail.

To execute a table scan on our table, we can do:
```sql
SELECT id, val
FROM dbo.items
```

![O n](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n).jpg "O n")

This scan involved reading 100 rows. And you know what? If there was 1000 rows in the table, there would be 1000 reads.

Since the number of reads is always equal to the number of rows in the table, we could say that `# of reads = # of rows`. And we can express this as a math formulae too: O(n)

## O(2n)
If n is the number of rows, then 2n means that the number of reads required is 2 times the number of rows. This means a query on our 100-row table would involve 200 reads.

A common example is a self-join, which is when you do a normal query, but then join it to the same table. An easy way to visualize the operation is to just imagine 2 copies of 1 table, then doing a join between them:

```sql
SELECT a.id, b.id
FROM dbo.items a JOIN dbo.items b ON a.id = b.id
```

Here, we scan all of our 100 rows, then join each row to itself.

* So the number of reads is 100 + 100
* ... or 2 * 100
* ... or 2 * (number of rows in our table)
* ... or 2n

![O 2n](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(2n).jpg "O 2n")

And we can verify this through the execution plan. Both scan operations did 100 reads, therefore giving us 200 reads total, or "double the number of rows in our table".

## O(n<sup>2</sup>)
The `CROSS JOIN` takes the cartesian products of 2 tables, giving us a mooshy, tasty, salty goo.

```sql
SELECT 1
FROM dbo.items a CROSS JOIN dbo.items b
```

![O n2](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n%5E2).jpg "O n2")

Now the database engine scans our table twice, each one rakin in 100 rows. Then they're joined using a NESTED loop, giving us a total of 10000 rows.

Now you may say, "wait a second, but thats 200 reads!". Well, you are technically correct. But now we are dealing with `JOIN`, which is an expensive operation relative to index scans.

Following the BIG O rule of "care about expensive operations relative to the rest", we now need to throw in join operations into the mix.

In fact, table scans to nested loop joins is like comparing oranges to apples - now the amount of joining we have to do should be the main focus when calculating big O (instead of calculating scan reads).

Hence, we will ignore the 200 reads (which are ignorable from a scalability calculation standpoint), and only care about the real "meat" - the number of inner loop executions to produce the cartesian product.

![O n2](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n%5E2).jpg "O n2")

Looking at the execution plan again, we wanna focus on "Actual Rows" & "Number of executions". Since the inner join operation is executed once start-to-finish, this leaves us with the 10000 inner loop operations to produce our cartestian product. So:

* 10000 inner loop operations
* ...or 100 * 100
* ...or 100<sup>2</sup>

### Bonus example
To help hammer in the horrifying effects of quadratic scalability, heres another O(n<sup>2</sup>) example thats a little more... "real". This example below uses a recursive CTE to print out all the nodes in a perfectly-balanced binary tree.
```sql
DROP TABLE IF EXISTS dbo.tree;
GO

CREATE TABLE dbo.tree (
	id INT IDENTITY PRIMARY KEY
	,label CHAR(1) NOT NULL
	,parent_label CHAR(1)
	,is_root BIT NOT NULL
);
CREATE INDEX IX_tree_is_root ON dbo.tree (is_root)
INCLUDE (label, parent_label);
GO

INSERT INTO dbo.tree (label, parent_label, is_root) VALUES
	('A', NULL, 1)
	,('B', 'A', 0)
	,('C', 'A', 0)
	,('D', 'B', 0)
	,('E', 'B', 0)
	,('F', 'C', 0)
	,('G', 'C', 0);
GO

WITH cte_tree AS (
	SELECT label, parent_label
	FROM dbo.tree
	WHERE is_root = 1
	UNION ALL
	SELECT child.label, child.parent_label
	FROM cte_tree JOIN dbo.tree child ON cte_tree.label = child.parent_label
)
SELECT *
FROM cte_tree;
```

Before we dive into the good stuff, here's the result set:

![O n2 result](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/recursive-tree-result.jpg "O n2 result")

And NOW, BEHOLD THE ALL MIGHTY PLAN!

![O n2 execution plan](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/recursive-tree-execution-plan.jpg "O n2 execution plan")

The 2 parts to focus on are:

* The Index Seek to get the root node of the tree. This is the represented by the row with `is_root = 1`
* And the Index Scan, which gets the remaining rows in the tree for our recursive CTE

Lets start with the Index Seek - it will always be 1, assuming our table has at least 1 row. Since we are calculating Big O, we are therefore only interested in the worst case, and therefore we can just assume there will be at least 1 row (would there even be a point of calculating big O that can only be applied for empty tables anyways?).

> When calculating Big O, we ignore numeric constants. 1 is a constant number, so we will ignore the "1 read" from the Index Seek operation

And now for the main course. The Index Scan has 49 "Number of Rows Read" & 6 "Actual Number of Rows".

> "Number of Rows Read" means the rows read from your storage medium, whereas "Actual Number of Rows" is the rows output by the operation. The number can differ when applying predicate, which you can identify underneath the "Predicate" header

Since were calculating scalability, the number of rows returned is of no interest to us, but rather the work REQUIRED to get those rows. So this leaves us with the 49 "Number of Rows Read".

* 49 rows read
* ...or 7 * 7 rows read
* ...or (number of nodes in tree) * (number of nodes in tree)
* ...or (number of nodes in tree) squared
* ...or n squared
* ...or n<sup>2</sup>

As you can see, this is for a table with only 49 rows. Imagine with millions of rows!

O(n<sup>2</sup>) algorithms are the kinds of things the SQL devil gives us when we skip church.

Enough about bad things - what kind of scalability should we be aiming for in our query tuning?

## O(log n)
Logarithmic math operations (which in the context of Big O, is to the base of 2) are equivalent to square root. To put it simply, database queries that half the amount of work required in each recursion step, exhibit logarithmic scalability characteristics.

An example usually used is searching through some tree. Fortunately for us, the SQL Server optimizer is so good at it's job that the execution plan it generates for recursive CTE's & the HIERARCHYID data type are usually BETTER than O(log n).

So instead, our example below is about doubling a number (until the result no longer exists in a table). Essentially, we have a recursive function that terminates when a INNER JOIN doesnt find a match.

For example, if we have a table with rows with values `1`, `2`, `4` & `8`, we start with 1 and end at 8.

```sql
DROP TABLE IF EXISTS dbo.scale;
CREATE TABLE dbo.scale (
	val INT IDENTITY PRIMARY KEY
	,tag INT NOT NULL
);
GO

INSERT INTO dbo.scale (tag) VALUES (1);
GO 64

WITH cte_scale AS (
	SELECT TOP 1 val
	FROM dbo.scale
	UNION ALL
	SELECT child.val
	FROM cte_scale JOIN dbo.scale child ON child.val = cte_scale.val * 2
)
SELECT val
FROM cte_scale;
```

Here we chuck in values between 1 & 64. Then let the recursive CTE roll!

![O log n result](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o-logn-result.jpg "O log n result")

![O log n execution plan](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o-logn-execution-plan.jpg "O log n execution plan")

Firstly, lets rule out constants again. Since we always start with 1 row read (for tables with at least 1 row), we can ignore the Index Scan that retrieves 1 row for the anchor portion of our CTE.

Secondly, we are dealing with `JOIN`s (in our recursive CTE), so we can rule out the Index Seek in the bottom-right. Its not used for a key lookup either, so its not really an expense compared to the `JOIN`.

And now, the `JOIN` operation. With our nested loop operation outputting 6 rows, we can see that the nested loop runs 6 times. So it goes from 1, 2, 4, 8, 16, 32, 64. Ignoring the initial step (which will always be constant, as noted before), this gives us:

* 6 steps for our 64 row table  
* ...or log<sub>2</sub>(64)

Oh yes, this is what dreams are made of. This is what true happiness feels like. I have tears of joy right now as we speak.

And this is the "end game" for your tuning - taking factorial/exponential/quadratic algorithms, and making them either O(1), O(n) or O(log n).

So far we have covered:

* O(1), what dreams are made of
* O(log n), the king we love most
* O(n), not bad, but must be tolerated in most cases when you need alot of unpaginated data
* O(2n), a pretty bad algorithm that makes you gag, but not puke your guts out
* O(n<sup>2</sup>), which is pretty much the devil. To me, O(n<sup>2</sup>) is just as bad as exponential or factorial, because these kinds of algorithms all just get cancelled in the form of a timeout anyway.

There is 1 more "common" Big O complexity, that fits in between O(2n) and O(n<sup>2</sup>). Its possible to get away with this kind of scalability, but sooner or later, you'll regret not tuning them once they start causing problems in production.

## O(n log n)
This is like `O(log n)`, but you run the algorithm `n` times instead of just once.

Below is an example that extends our "keep squaring a number" example from before. But instead of running the recursive CTE for 1 anchor row, we run it for each row in our table.

To simplify this example to work with our previous example table (which was a small 64-row table), i'm:

* Limiting the run to 8 executions (i.e.: the 1st 8 rows) via a `TOP 8`
* Limiting each execution to only recurse 3 times

So we run the O(log n) algorithm 8 times, with each execution doing 3 recursion steps. So we expect 8 * 3 = 24 executions:

```sql
TRUNCATE TABLE dbo.scale;
GO

INSERT INTO dbo.scale (tag) VALUES (1);
GO 64

WITH cte_scale AS (
	SELECT TOP 8 val, val AS 'initial', 1 AS 'step'
	FROM dbo.scale
	UNION ALL
	SELECT child.val, cte_scale.initial, cte_scale.step + 1
	FROM cte_scale JOIN dbo.scale child ON child.val = cte_scale.val * 2
	WHERE cte_scale.step < 4
)
SELECT val, initial, step
FROM cte_scale;
```

![O n log n execution plan](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n%20log%20n).jpg "O n log n execution plan")

Firstly, we ignored constants when calculating Big O. So let's weed out the constant 8 rows we use as our recursive CTE anchor.

Secondly, we only focus on the "meat" when calculating Big O. Relatively speaking, index seeks (that dont lead to key lookups) are cheap compared to nested inner loop operations.

Thirdly, we only care about the worst-case scenario when calculating Big O. The worst case for this algorithm is starting with "1", because it will involve the most operations (1 will keep squaring until it hits the biggest value in our table - 64)

This leaves us with the nested loop operation, that outputs 24 rows (i.e.: the outer loop used the inner loop 24 times). So this means 24 executions (8 runs each run * 3 recursive steps occurring each run).

Assuming we only care about the worst case, we should remove the limits we put on our algorithm (which was just used to simplify this example):

* The max number of executions limited to 8
* The max recursion step of 3 in each execution

Thus, our table would have a larger max value (i.e.: the largest number in our table would be much larger than 64). From a realistic standpoint, tables usually have millions or billions of rows, so think about a max value of 100 million instead of 64.

And the worse case for the algorithm is when the "keep squaring" operations starts from the smallest number (i.e.: 1).

So what we have here is:

* an O(log n) operation - the "keep squaring" recursive CTE
* and which is run for each row in the table - i.e.: `n` times

Therefore, the scalability (in math formulae form) is:

* `n` multiplied by `O(log n)`
* ...or `n * log(n)
* ...or `n log(n)`
* ...or `O(n log n)`

## Why you should care?
A characteristic of startup companies is that they start with a small amount of customers, which then (hopefully) grows over time.

This means they start with a small amount of data, but then that data size grows significantly over time. And it KEEPS growing, there's usually more data coming in from customer use, than the number of rows that get archived or even permanently deleted.

But thats not the problem - the ACTUAL problem is that database stuff is expensive to fix - its a long process, sql is less malleable than UI or API code, and the most important queries (which run against lots of data) are complex, because programmers need to continually trade understandability for performance to keep up with the ever-increasing customer load.

This is a huge expense for a business.

And this doesnt help that since everything runs fast with small datasets, these kinds of scalability issues go under the radar, and only appears when the problem has already taken effect.

By making database operations scalable early in development cycle (rather than when its already in production), we can minimise the business cost for our company, and the technical debt for our software team.
