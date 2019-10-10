---
title: Calculating Big O time complexity for SQL Server execution plans
layout: post
author: Michael Nguyen
---
We can specify how an algorithm scales with data size, by expressing its' scalability as a mathematical formula. This is called "Big O notation".

As far as measuring scalability goes, the 3 common weapons of choice are:
* Big Omega: a formula that defines best-case algorithm scalability
* Big O: a formula that defines worst-case algorithm scalability
* Big Theta: a formula that defines both worst-case & best-case algorithm scalability

Additionally, Big O is usually divided into 2 metrics:
* Time complexity: the worst-case time required for the algorithm to complete
* Space complexity: the worst-case space required for the algorithm to complete. This covers either persistent memory such as disk, or volatile memory such as RAM.

This post will run through Big O time complexity by analysing SQL Server execution plans. I'm starting with a fresh db, so if you feel like a fat slob and have cheeseball stains on your shirt, get some exercise by typing in the sql examples.

Speaking of typing things, heres the first example now!:
```sql
CREATE DATABASE big_o;
GO
USE big_o;
GO
```

## O(1)
In the context of database *queries*, O(1) means that 1 read is executed, regardless of the number of rows in a table. E.g.: seeking 1 row by using a predicate against an indexed column:
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

The sql above creates a 100-row table, then runs a query with a predicate that is usable against the implicit clustered index.

![O1](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(1).jpg "O1")

And we can verify the number of reads! The execution plan for this query identifies only 1 "*Number of Rows Read*", even though this table has 100 rows.

Since the database engine only reads 1 row (regardless of the number of rows in the table), we can say that the worst-case performance for this query is O(1).

But what would happen if a table scan was done instead? SQL Server would have to read all rows in our table using an Index Scan operation (100 reads). And if the table had 200 rows, 200 reads would occur. If 400 rows, 400 reads would occur and so forth.

You know what? We can also express this kind of "scalability" as a math formula as well.

## O(n)
A (non-seeked) table scan means that database engine goes to the first leaf node in the clustered binary tree index, then reads all the rows by advancing through all leaf nodes via the "next" pointer (which exists in each leaf node). This semantic is similar to reading a linked list from head to tail.

To execute a table scan on our table, we can do:
```sql
SELECT id, val
FROM dbo.items
```

![O n](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n).jpg "O n")

This scan required reading all 100 rows. And you know what? If there was 1000 rows in the table, the scan would involve 1000 reads.

Since the number of reads is always equal to the number of rows in the table, we could say that `# of reads = # of table rows`.

In other words: O(n)

## O(2n)
If `n` is the number of rows, then `2n` means that the number of reads required is twice the number of rows. This means a query on our 100-row table would have to involve 200 reads.

A common example of this is a self `JOIN`, which is when you do a normal query, but then join to the same source table again. An easy way to visualize this operation is to just imagine 2 copies of 1 table, then visualizing a normal `INNER JOIN` between them:

```sql
SELECT a.id, b.id
FROM dbo.items a JOIN dbo.items b ON a.id = b.id
```

Here, we scan all of our 100 rows, then `JOIN` each row to itself.

* So the number of reads is 100 + 100
* ... or 2 * 100
* ... or 2 * (number of rows in our table)
* ... or 2n

![O 2n](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(2n).jpg "O 2n")

And as we can see, the execution plan verifies our idea. Both scan operations did 100 reads, therefore giving us 200 reads total, which is *double* the number of rows in our table.

## O(n<sup>2</sup>)
If n is 100 rows, then n<sup>2</sup> would be 10000 reads. The `CROSS JOIN` logical operation takes the cartesian product of 2 tables, giving us a mooshy, tasty, salty goo.

```sql
SELECT 1
FROM dbo.items a CROSS JOIN dbo.items b
```

![O n2](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n%5E2).jpg "O n2")

As we can see above, our database engine now scans our table twice, each one raking in 100 rows. Then they're joined using a **nested loop**, giving us a total of 10,000 rows.

Now you may say, "wait a second, but thats 200 reads!". Well, you are technically correct. But now we are dealing with `JOIN`, which is an expensive operation relative to index scans.

> When calculating Big O, you only care about the "meat" of the algorithm - the stuff that actually gobbles up alot of CPU/memory/time. Thus we discard RELATIVELY cheap operations

Comparing Index Scans to nested loop executions is like comparing oranges to apples. The amount of joining we have to do in-memory should be the main focus when calculating big O (rather than focusing on the scan read operations that pull disk pages into memory).

Hence, we will ignore the 200 reads (which are ignorable from a scalability calculation standpoint), and only care about the real "meat" - the number of inner loop executions that produced the cartesian product:

![O n2](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o(n%5E2).jpg "O n2")

Furthermore, looking at the execution plan again, we wanna focus on "*Actual Number of Rows*" & "*Number of executions*". The execution plan identifies that our `INNER JOIN` operation executed once start-to-finish. This leaves us with the 10,000 inner loop operations to produce our cartestian product. So:

* 10000 inner loop operations
* ...or 100 * 100
* ...or 100<sup>2</sup>

### Bonus example
To help hammer in the horrifying effects of quadratic scalability, heres another O(n<sup>2</sup>) example thats a little more... "real". This example below uses a recursive CTE to print out all the nodes in a [Perfect Binary Tree](https://en.wikipedia.org/wiki/Binary_tree#Types_of_binary_trees).
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

AND NOW, BEHOLD OUR ALL MIGHTY PLAN!

![O n2 execution plan](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/recursive-tree-execution-plan.jpg "O n2 execution plan")

The 2 parts to focus on are:

* The **Index Seek**, which gets the root node of the tree. This is represented by the row we inserted with `is_root` set to `1`
* The **Index Scan**, which gets the remaining rows in the tree for our recursive CTE

Lets start with the **Index Seek**.

It will always be 1, assuming our table has at least 1 row. Since we are calculating Big O, we are therefore only interested in the worst case, and therefore we should assume there will be at least 1 row (would there even be a point of calculating big O that can only be reliably used for [empty tables](https://blog.codinghorror.com/everything-is-fast-for-small-n/) anyways?).

> When calculating Big O, we ignore numeric constants. 1 is a constant number, so we ignore the "1 read" from the Index Seek operation

And now for the main course. The Index Scan has 49 "Number of Rows Read" & 6 "Actual Number of Rows".

> "Number of Rows Read" is the number of rows read from your storage medium, whereas "Actual Number of Rows" is the rows output by the plan operation. These numbers can differ in queries with predicates, which you can identify underneath the "Predicate" header

Again, since we only care about the "meat", the "*Number of Rows Returned*" is of no interest to us, but the work REQUIRED to get those rows IS of interest to us. So this leaves us with the 49 "*Number of Rows Read*".

* 49 rows read
* ...or 7 * 7 rows read
* ...or `# of nodes in tree` * `# of nodes in tree`
* ...or `# of nodes in tree` squared
* ...or `n` squared
* ...or n<sup>2</sup>

Awesome work! We were able to calculate the worst-case time complexity for this algorithm, and express it as a convenient math formula - O(n<sup>2</sup>).

Our example table only had 49 rows. Imagine running this kind of unscalable algorithm with millions of rows! O(n<sup>2</sup>) algorithms are the truly the kinds of horrors the SQL devil injects into our dreams after we skip church.

Enough about the bad. Now to identify the kind of scalability we should be aiming for in our query tuning...

## O(log n)
Logarithmic math operations (which in the context of Big O, is to the base of 2) are equivalent to square root operations. To put it simply, database queries that halve the amount of work required after each step, exhibit logarithmic scalability characteristics.

A common example is searching through some tree. Fortunately for us, the SQL Server optimizer is so good at it's job that the execution plans it generates for recursive CTE's & the [HierarchyId](https://docs.microsoft.com/en-us/sql/t-sql/data-types/hierarchyid-data-type-method-reference?view=sql-server-2017) data type often run *better* than `O(log n)`.

So for maximum clarity, the example we will run through this time just recursively doubles a number. Essentially, we will have a recursive function that terminates when a INNER JOIN doesnt find the next value in our table.

For example, if we have a table with rows with values `1`, `2`, `4` & `8`, we start doubling from 1 & keep doing until we hit 8.

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

Here we chuck in values between 1 & 64, then we sit back & let the recursive CTE roll:

![O log n result](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o-logn-result.jpg "O log n result")

![O log n execution plan](https://media.githubusercontent.com/media/vitawebsitedesign/blog/master/assets/o-logn-execution-plan.jpg "O log n execution plan")

### Let the calculations begin!
Firstly, lets rule out constants again. Since we always start with 1 row read (for tables with at least 1 row), we can ignore the `Index Scan` operation that just retrieves 1 row for the anchor portion of our `cte_scale` CTE.

Secondly, we are dealing with `JOIN`s (in our recursive CTE), so we can rule out the `Index Seek` operation (shown in the bottom-right of the pic). Its not used for a key lookup either, so its not really an expense when relatively compared to the `JOIN`.

And now, the `JOIN` operation. With our nested loop operation outputting 6 rows, we can identify that the nested loop ran 6 times. We can cross-verify this with our output of `1, 2, 4, 8, 16, 32, 64`. Ignoring the initial step (which will always be constant, as noted before), this gives us:

* 6 steps for our 64 row table  
* ...or log<sub>2</sub>(64)
* ...or log<sub>2</sub>(rows in table)
* ...or log<sub>2</sub>(n)

Oh yes, this is what dreams are made of. This is what true happiness feels like. I have tears of joy right now as we speak.

And this is the "end game" for your tuning - taking factorial/exponential/quadratic algorithms, and making them either `O(1)`, `O(n)` or `O(log n)`.

So far we have covered:
* O(1), what dreams are made of
* O(log n), the king we love most
* O(n), not bad, but must be tolerated in most cases when you need alot of unpaginated data
* O(2n), an unscalable algorithm that makes you gag, but not puke your guts out
* O(n<sup>2</sup>), which is the most evil of all. To me, O(n<sup>2</sup>) is just as bad as exponential or factorial scalabilities, because these kinds of algorithms all just get terminated halfway through anyways, either by the user exiting, or by the server in order to open resources for other connections.

There is 1 more "common" Big O complexity that fits somewhere between `O(2n)` and O(n<sup>2</sup>).

Its possible to get away with this kind of scalability, but sooner or later, they strike production from the shadows, and you'll regret the day you slouched in your chair, lit a cigar & proclaimed yourself as the king of database tuning.

## O(n log n)
This is like `O(log n)`, but you run the algorithm `n` times instead of just once.

Below is an example that extends our "keep squaring a number" example from before. But instead of running the recursive CTE for 1 anchor row, we run the whole algorithm for each table row.

To simplify this example to work with our previous example table (which was a small 64-row table), i'm:

* Limiting the run to **8** executions (i.e.: the 1st 8 rows) via a `TOP 8`
* Limiting each execution to only recurse **3 times**

This means the `O(log n)` algorithm will run 8 times, with each execution doing 3 recursion steps. So we end up with `8 * 3 = 24` executions:

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

### Calculating Big O for this algorithm
Firstly, we ignore constants when calculating Big O. So let's weed out the constant 8 rows we use as our recursive CTE anchor.

Secondly, we only focus on the "meat" when calculating Big O. Relatively speaking, index seeks (that dont lead to key lookups) are cheap compared to nested `INNER LOOP` operations.

Thirdly, we only care about the worst-case scenario when calculating Big O. The worst case for this algorithm is starting with "1", because that starting point involves the most operations (1 will keep squaring until it hits the biggest value in our table - 64)

This leaves us with the nested loop operation, which outputs 24 rows (i.e.: the outer loop used an inner loop 24 times). So this means **24** executions (**8** algorithm executions, multiplied by the **3** recursive steps occurring in each run).

Whilst we have put limits on our algorithm for demonstration sakes, we have to theoretically remove these limits in order to calculate Big O. The limits we applied were:
* A max number of algorithm executions limited to 8
* A max recursion step of 3 within each run

### Theoretical asymptotic limits
With the limits removed, our table would realistically have a much larger max value (much larger than 64). From a pragmatic perspective, real-world tables usually have millions/billions of rows, so think about a max value of say... **100,000,000** instead of **64**.

Additionally, the worst-case time-complexity for our algorithm is when the "keep squaring" operation starts from the smallest number (i.e.: 1).

So now we have an `O(log n)` operation - the "recursive squaring" CTE, which is run for each row in the table - i.e.: `n` times.

Therefore, the scalability (in the form of a math formula) is:

* `n` multiplied by `O(log n)`
* ...or `n * log(n)`
* ...or `n log(n)`
* ...or `O(n log n)`

## The dangers of ignoring scalability
A characteristic of startup companies is that they start with a small amount of customers, which then (hopefully) grows over time.

### Data growth + low detectability = danger
This means they start with a small amount of data, but then that data size grows significantly over time. And it KEEPS growing - in fact, there's usually more data coming IN from customer use, than the number of rows going OUT (e.g.: archiving or permanent deletion).

But thats not the problem - the ACTUAL problem is that database stuff is expensive to fix - its a long process, sql is less malleable than UI or API code, and the most important queries (which run against the most data) are complex, because programmers need to continually trade understandability for performance to maintain uptime against the ever-increasing customer load.

This is a huge expense for a business.

And it doesnt help that everything runs fast with small datasets, therefore making these kinds of scalability issues fly under the radar, and make a surprise appearance in prod when the issue has already affected customers.

### Conclusion
By making database operations scalable early in the development cycle (rather than later when its already in production), we can minimise the long-term business cost for our companies, as well as managing the technical debt for our software team.

You'd be surprised how much money, time & effort you save down the line, & all it takes is a couple of extra hours tuning execution plans.
