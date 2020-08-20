---
title: Dear Santa, no more offset pagination - it sucks!
layout: post
author: Michael Nguyen
---

A common pagination method is to use an offset-like operator (or equivalent, such as Sqlite LIMIT):

```sql
DECLARE @pageIndex INT = 0;
DECLARE @pageSize = 10;

SELECT *
FROM dbo.MyTable
OFFSET @pageIndex * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY
```

So assuming an appropriately indexed table, page 1 (pageIndex=0) would pull 10 rows for cache/disk.

Page 2 pulls 10 rows for page 1, then another 10 for page 2.

And so on. As we can see, it gets less efficient as the `@pageIndex` gets higher. WHEN WILL THE HORROR END.

```
O(n) = pageIndex * pageSize + pageSize
     = i * p + p 
     = i * 2p
     = i * p
     = ip
     = O(ip)
```

# Other alternatives
A lot of pagination is done this way (coined “offset pagination”), but for products under high workload, most of the time it would actually be more I/O efficient to just have the db pull all rows. Then paginate on the server side or client side.

### Server side
```csharp
var rows = await dbContext.MyTable.ToListAsync(); 
return await rows.Skip(pageIndex * pageSize).Take(pageSize).ToListAsync();
```

Here we:
1. Pull all database table rows from disk/cache
1. Transfer that data over the wire onto the API (e.g.: micro service call)
1. Instantiate new array onto heap, containing all table rows
1. Seek an Enumerable to the offset (start) index
1. Pull data into response stream

And since most applications are client-side routing, we can assume the memory cost of materialising all the rows into disk & returning them over the wire to the client.

### Client side
```
const url = `{root}`/api/v1/my-table`;
$.get(url, allTableRows => {
  const pageIndex = 0;
  const pageSize = 2;
  const offset = pageIndex * pageSize;
  const limit = offset + pageSize;

  const rowsForPage = allTableRows.slice(offset, limit);
  …
});
```

This is interesting - it's usually considered bad practice, but for "busy" products where pagination is used frequently (& deeply), its a better option than client-side or offset pagination because:
- DB CPU & I/O cost is lower (most paginated queries involve `WHERE` & key lookups). DB’s are a common & expensive-to-fix bottleneck when startup user bases grow, so this is an important factor.
- Server requires less code footprint, so its faster to build, release & read.

But what about the disadvantages?:
- Client has to download more data (ouch, bad news for mobile users)
- Severe scalability issue that’s tied to # of table rows. Example, a table with 3 million rows will cause users to throw their laptops & phones out the window.

No dice yet on a scalable solution… or is there?

# Scalable pagination
Over time, devs decided to return a pointer to the “last position” of an iterative operation. So:

```csharp
return await rows.Skip(pageIndex * pageSize).Take(pageSize).ToListAsync();
```

…becomes:

```csharp
return rows.Skip(pageIndex * pageSize).Take(pageSize);  // Enumerable instead of List
```

And:

```sql
FROM dbo.MyTable
OFFSET @pageIndex * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY
```

…becomes:

```sql
FROM dbo.MyTable
WHERE id > @maxIdOfLastPage
OFFSET @pageIndex * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY
```

### What are we even lookin' at?
The concept of yielded results has been around for ages, but devs often still eagerly materialise collections when they don’t need to. It’s just what they’re used to, they like the feeling of comfort, but they pay the price in the future (both mentally & financially).

The idea behind `WHERE` is to execute an index seek-scan to the “last position” (i.e.: the last row of the page the user is currently on), then do seek-scan/scan/key lookups to finish the job.

Tada, problem solved! Big O goes from `O(ip)` to:

```
O(n) = rowsReadForSeekScan + pageSize
     = r + p
     = O(r + p)
```

Ah, that’s better. Now our scalability is tied to our DB index (instead of table row count). Double linked-list & B-trees FTW!

This method is “keyset pagination”.

# Why doesn’t everyone use keyset pagination?
Whoa hold on a sec buddy! - lets not start refactoring large codebases that dont provide real business value!

Keyset sure is awesome, but not all situations suit it. Example, a dev might have to write ANSI-compliant SQL to run paginated queries across multiple database providers (such as on both SQL Server & Sqlite).

# The cloud kicks ass!
As the cherry on the top, certain cloud providers have gone even further & abstracted the Keyset logic away from the consumer (developer)!

For Azure cosmos queries, Microsoft gives consumers the table rows & a “special string”. Then on subsequent calls, you add this “special string” to your request and BOOM, like *MAGIC*, you get paginated results for the next page (really, really fast).

These “special strings”, or **continuation tokens**, make developer life easier. Hey, thats us!

# Conclusion
Pagination method | I/O efficiency | Memory efficiency | Network efficiency | When to use
|--- |--- |--- |--- |---
Offset pagination (database layer) | Bad | Good | Good | Need to write ANSI SQL
Offset pagination (server side) | Bad | Good (if yielded), bad otherwise | Good | Don’t need ANSI SQL & need scalability without high development cost
Offset pagination (client side) | Good | Bad | Good (don’t need extra API calls for subsequent pages) | Little/no memory & network constraints, no bottleneck with front-end dev resources, and need product scalability with fast backend releases
Keyset pagination (database layer) | Good | Good | Good | Always use this over offset pagination if supported
Keyset pagination (server side) | Good | Bad | Good | DB bottlenecking, or dont want to additional index on table exposed to significantly more writes than reads
Keyset pagination (client side) | Good | Good (if client has no memory constraint), bad otherwise | Good | Users use pagination frequently & deeply on your product, and you want to offload work onto client devices
:star: Continuation token | Good | Good | Good | Always use this over keyset pagination if supported (lower code complexity & development cost)
