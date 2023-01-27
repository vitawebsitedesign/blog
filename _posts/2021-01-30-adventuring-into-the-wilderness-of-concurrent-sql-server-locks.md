---
title: Adventuring into the wilderness of concurrent SQL Server locks
layout: post
author: Michael Nguyen
toc: true
---

SQL Server reads & writes to rows. These can happen from multiple [transactions](https://docs.microsoft.com/en-us/sql/t-sql/language-elements/transactions-transact-sql?view=sql-server-ver15), so some concurrency control is built-in.

These built-in controls are called "locks".
- Read lock
- Write lock

> There’s also an Update lock, which involves a concept called "lock conversions". For simplicity, it wont be covered here

# Read lock
This just means a read operation. So if a transaction wants to read a row, it will place a read lock onto the row.

> The technical term for a *read lock* is **shared lock**.

# Write lock
This just means a write operation. So if a transaction wants to write to a row, it will place a write lock onto the row.

> The technical term for a *write lock* is **exclusive lock**.

## Concurrency
For starters, multiple read locks can be applied to the same row (its just reads after all, no data is being changed).

We don't care about that simple scenario. We wanna jump to the juicy stuff...

I'm talking about 2+ writes (exclusive locks) occurring on the same row(s) at the same time. In this case, [only 1 of the locks will be applied](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms186396(v=sql.105)?redirectedfrom=MSDN) which will "win the battle".

> An edge case wont be discussed here for simplicity: but in short, when there's a significant amount of row locks, SQL Server upgrades them to a table-level lock to save on space & improve efficiency. For now, we assume a scenario starting with 0 row-level locks

### 2 at the same time!
2 **exclusive locks** can't exist on the same row at the same time. When this happens, one of write operations is cafncelled.

The technical term for a *cancelled operation* is [deadlock](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms177433(v=sql.105)), & the cancelled operation is a **deadlock victim**

### A hare & a tortiose?
Most of the time though, both write operations don't occur at exactly the same moment.

Perhaps one happens, then 100 milliseconds later, another write is attempted to the same row by another transaction.

In this case, a deadlock does NOT occur. Instead, the later transaction (that was late to the party) just waits patiently until the 1st transaction releases its' exclusive lock.

However if the 1st transaction simply takes too long & the 2nd transaction waits too long, a timeout occurs.

# What does this mean for applications?
This means application code must handle:
- [Deadlock victims](https://stackoverflow.com/a/2256954)
- [Query timeouts](https://stackoverflow.com/a/62688)

To support basic concurrency in a fault-tolerant manner.
