---
title: SQL Server "NOT IN" with 3-valued logic
layout: post
author: Michael Nguyen
---
SQL Server uses booleans. Specifically, the logic system used is Kleene's 3-valued logic.

## 3-valued logic?
SQL Server has 3 possible outcomes for boolean expression: [`true`, `false` and `unknown`](https://en.wikipedia.org/wiki/Three-valued_logic#Kleene_and_Priest_logics).

`Unknown` happens when comparing to unknown values, for example `NULL`.

Example:

```sql
SELECT * FROM dbo.taco WHERE rating NOT IN (1, NULL)
```

## Kleene logic?
Kleenes logic just means that an evaluated result of UNKNOWN is equivalent to false. So this means SQL Server wont return rows with that evaluation.

## In action
`IN` is just shorthand for multiple `OR` statements:

```sql
SELECT * FROM dbo.taco WHERE rating IN (1, 2)
```

is

```sql
SELECT * FROM dbo.taco WHERE (rating = 1 OR rating = 2)
```

and `NOT IN` is just shorthand for multiple `AND` statements:

```sql
SELECT * FROM dbo.taco WHERE rating NOT IN (1, 2)
```

is

```sql
SELECT * FROM dbo.taco WHERE rating <> 1 AND rating <> 2
```

## Where does Kleenes logic differ?
The most important part of Kleenes logic is that `FALSE = UNKNOWN` evaluates to `FALSE`.

This is contradictory to [redgate's article](https://www.red-gate.com/simple-talk/sql/learn-sql-server/sql-and-the-snare-of-three-valued-logic/)), you should instead refer to [Wikipedia's matrixes](https://en.wikipedia.org/wiki/Three-valued_logic#Kleene_and_Priest_logics).

So...

```sql
SELECT * FROM dbo.taco WHERE rating NOT IN (1, 2)
```

is fine and dandy, and will return rows that dont have a rating 1 or 2. But..

```sql
SELECT * FROM dbo.taco WHERE rating NOT IN (1, NULL)
```

is a different game, it returns 0 rows

## 0 rows?
Yep. As noted earlier, the above sql statement is actually

```sql
SELECT * FROM dbo.taco WHERE rating <> 1 AND rating <> NULL
```

the `rating <> NULL` bit is evil. Since comparison to NULL is UNKNOWN, and UNKNOWN evaluations are equivalent to `FALSE` in Kleene logic, no rows will match.

And SQL Server aint gonna save your butt with a compile-time error either. Its just gonna sit back and watch you suffer.

## Correctly using NOT IN in Kleene logic
To get the behaviour you expect, you can:

* Convince the Microsoft CEO to change SQL Server's logic system.
* Or just weed out the NULLs, so that the only possible evaluations are `TRUE` and `FALSE`

Example:

```sql
SELECT *
FROM dbo.taco
WHERE rating NOT IN (
	1
	,NULL
)
```

to

```sql
SELECT *
FROM dbo.taco
WHERE rating NOT IN (
	1
)
```

And a realistic example would be:
```sql
SELECT *
FROM dbo.taco
WHERE rating NOT IN (
	SELECT rating
	FROM dbo.movie_ratings
)
```

to

```sql
SELECT *
FROM dbo.taco
WHERE rating NOT IN (
	SELECT rating
	FROM dbo.movie_ratings
	WHERE rating IS NOT NULL
)
```

## ANSI_NULLS remarks
SQL Server default for [ANSI_NULLS](https://docs.microsoft.com/en-us/sql/t-sql/statements/set-ansi-nulls-transact-sql?view=sql-server-2017#remarks) will change to ON in the future.

If you like living on the edge with ANSI_NULLS still off, you can continue using your precious c#-like `X = NULL` and `X <> NULL` operators (instead of `IN` and `NOT IN`).

But as of this post, the future has alraedy arrived, and Microsoft has laid down the law.

The examples ive given assume you like standards, and that you have either:

* flipped `ANSI_NULLS` to `ON` like a real dev
* or created a new'ish database that already has the `ON` default
