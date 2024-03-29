---
title: Situational concurrency control with SQL Server locks
layout: post
author: Michael Nguyen
toc: true
---

# Lock control
Locks are a trade between concurrency and performance. Locks take up *space* (depending on the table's primary key), and they have an *efficiency* overhead.

SQL Server allows you to control this "trade", by determining lock behaviour through a "setting" called [isolation levels](https://docs.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql?view=sql-server-ver15).

So depending on existing application requirements & future requirements volatility, choosing the **appropriate** level could be useful.

TLDR:
- `Read uncommited`: no shared or exclusive locks are used
- `Read committed`: shared and exclusive locks are used
- `Repeatable Read`: shared locks are held until end of transaction
- `Serializable`: locks now cover a range of rows (instead of just singular rows)

> There is also a level called "Snapshot", which involves new SQL Server [settings](https://support.esri.com/en/technical-article/000013039) & a concept called "row versioning". For simplicity, Snapshot wont be covered here

## Read Uncommitted
Maximum speed! No locks means no overhead, but it also means no concurrency control. Its a free-for-all!

Since there's no locks, rows can be updated & read mid-transaction. For example:

| Transaction A      | Transaction B |
|--------------------|---------------|
| Transaction start  |               |
| Update row         |               |
|                    | Read row      |
| Commit transaction |               |

This is fine, transaction B still read the value which still ends up being correct anyways.

But what if transaction A fails?

| Transaction A                 | Transaction B |
|-------------------------------|---------------|
| Transaction start             |               |
| Update row                    |               |
|                               | Read row      |
| Tries to commit transaction   |               |
| Error, rolls back transaction |               |

Ah nuts! Now transaction B has the wrong value! If this situation would be a problem for your application, you might have to use a *stricter* isolation level...

## Read Committed (default)
As the default level, shared & exclusive locks are used. Lets continue our adventure from last time:

| Transaction A                 | Transaction B |
|-------------------------------|---------------|
| Transaction start             |               |
| Update row                    |               |
|                               | Read row      |
| Tries to commit transaction   |               |
| Error, rolls back transaction |               |

We  guchi, guchi wearing prada.

Transaction A places an exclusive lock on the row, then Transaction B tries to read the row (by placing a shared lock). Since exclusive & shared locks are incompatible, transaction B sits there, patiently, in silence, until transaction A finishes its' dance.

And hence, transaction B would finally resume, & read the correct value.

### Whew! Crisis averted...?

Crap. The above example is just 1 of the 2 possible scenarios. The other scenario is that Transaction B starts first:

| Transaction A | Transaction B     |
|---------------|-------------------|
|               | Transaction start |
|               | Read row          |
| Update row    |                   |

In read committed, shared locks are actually released immediately after the read. To think of this another way:

| Transaction A               | Transaction B          |
|-----------------------------|------------------------|
|                             | Transaction start      |
|                             | Read row (shared lock) |
|                             | Shared lock released   |
| Update row (exclusive lock) |                        |
|                             | Transaction end        |

As mentioned before, exclusive & shared locks are incompatible. Whoever arrives 2nd has to wait.

But since shared locks are released immediately after the read, Transaction A still gets dibs & can update the row - even though Transaction B is still halfway through!

This becomes an issue, because if Transaction B were to do another read **after** the row gets updated, it would read different data:

| Transaction A               | Transaction B          |
|-----------------------------|------------------------|
|                             | Transaction start      |
|                             | Read row (shared lock) |
|                             | Shared lock released   |
| Transaction start           |                        |
| Update row (exclusive lock) |                        |
| Transaction end             |                        |
|                             | Read row (shared lock) |
|                             | Transaction end        |

So if Transaction B were something like:
```csharp
var val = await db.QueryAsync<bool>("SELECT col FROM table");
if (val == true)
{
  ...   
}
```

And then Transaction A barged in halfway through (before the `IF` statement):

```csharp
await db.ExecuteAsync("UPDATE table SET col = 0");
```

Then the `IF` statement (in Transaction A) would return true, even though Transaction B set the column to `0` beforehand.

*Next week, on season 2 of transactions attacking eachother...*

## Repeatable Read
Shared locks are held until end of transaction. Continuing our adventure:

| Transaction A               | Transaction B          |
|-----------------------------|------------------------|
|                             | Transaction start      |
|                             | Read row (shared lock) |
|                             | Shared lock released   |
| Transaction start           |                        |
| Update row (exclusive lock) |                        |
| Transaction end             |                        |
|                             | Read row (shared lock) |
|                             | Transaction end        |

now becomes:

| Transaction A               | Transaction B          |
|-----------------------------|------------------------|
|                             | Transaction start      |
|                             | Read row (shared lock) |
|                             | `Shared lock NOT released`   |
| Transaction start           |                        |
| Update row (exclusive lock) |                        |
| Transaction end             |                        |
|                             | Read row (shared lock) |
|                             | Transaction end        |

And we're good! When Transaction A tries to apply exclusive lock, it has to wait there patiently because the Transaction B's shared lock is still on the row.

So

```csharp
var val = await db.QueryAsync<bool>("SELECT col FROM table");
if (val == true)
{
  ...   
}
```

Cant be "interrupted" halfway through, & the `IF` statement works as expected. Only once Transaction B ends, the shared lock is released, then Transaction A's exclusive lock is applied to do the update:

```csharp
await db.ExecuteAsync("UPDATE table SET col = 0");
```

## Serializable
This isolation level takes the cake. Database developers get down on their knees & pray to the SQL gods for whoever invented this level!

The previous levels covered common cases, but there's 1 extra scenario that often happens in applications, & thats when a SQL statement `SELECT` a range of rows, & then another Transaction `INSERT` rows in that range.

Example table:
| Id | Value |
|----|-------|
| 1  | 1     |
| 2  | 100   |

Then a `SELECT`:

```csharp
var rows = await db.QueryAsync<int>("SELECT Id FROM table WHERE Value BETWEEN 1 AND 100");
if (rows.Count() == 2)
{
  ...
}
```

Which grabs rows with `Id` 1 & 2. No problems so far right?

Oh wait a second, whats that thing called when multiple transactions bump heads? Oh yeah, HELL

```csharp
await db.ExecuteAsync("INSERT INTO table (Value) VALUES (10)");
```

If this runs before the `IF` statement, the `IF` statement will return `true`. The original `SELECT` got 2 rows, but now there's 3 rows in DB, & now the `IF` statement is checked with outdated data!

Why must programming be so complicated! Nothing to worry, because the Microsoft SQL Server devs took some good legal substances & hallucinated a new isolation level to make the `INSERT` wait until the first transaction finishes.

And thus `SERIALIZABLE` was born. To recap the `SELECT` again:

```csharp
await db.QueryAsync<int>("SELECT Id FROM table WHERE Value BETWEEN 1 AND 100");
```
In Serializable, shared & exclusive locks apply to a range of rows (instead of a single row). This means that if another transaction tries to shove another row between that range, it will be blocked.

```csharp
await db.ExecuteAsync("INSERT INTO table (Value) VALUES (10)");
```

Boom! Problem solved.
