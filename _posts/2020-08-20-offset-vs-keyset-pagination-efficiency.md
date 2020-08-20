---
title: The efficiency of offset & keyset pagination on different application layers
layout: post
author: Michael Nguyen
---

The common method of pagination is to use offset (or equivalent, such as LIMIT):

```sql
DECLARE @pageIndex INT = 0;
DECLARE @pageSize = 10;

SELECT *
FROM dbo.MyTable
OFFSET @pageIndex * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY
```

So assuming appropriately indexed, page 1 (pageIndex=0) pulls 10 rows for cache/disk.

Page 2 pulls 10 rows for apge 1, then another 10 for page 2.

And so on. As we can see, it gets less efficient as the @pageIndex gets higher. WHEN WILL THE HORROR END. Let’s write it in Big O notation, because math makes us appear smarter to others:

```
O(n) = pageIndex * pageSize + pageSize
        = i * p + p 
        = i * 2p
        = 2p
        = p
```

# Other alternatives
A lot of pagination is done this way (coined “offset pagination”), but for high-activity products, most of the time it would actually be more I/O efficient to just have the db pull all rows. Then paginate on the server side or client side.

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

And since most applications are client-side routing, so we also assume the memory cost of materialising all the rows into disk & returning them over the wire to the client.

### Client side
```javascript
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

This is interesting. Usually considered bad practice, but for busy products where pagination is used frequently (& deeply), its a better option than client-side or offset pagination because:
- DB CPU & I/O cost is lower (most paginated queries involve `WHERE` & key lookups for additional columns). DB’s are common bottleneck when startup user bases grow, so this is important.
- Server involves less code, so its faster to build & simpler to read

Disadvantages:
- Client has to download more data.
- Severe scalability issue that’s tied to # of table rows. Example, a table of 3 million rows will make users throw their laptops & phones out the window.

No dice yet on a scalable solution… or is there?

# Scalable pagination
Over time, devs decided to return a pointer to the “last position” of a pagination query (in addition to the table rows). So:

```sharp
…
return await rows.Skip(pageIndex * pageSize).Take(pageSize).ToListAsync();
```

…becomes:

```csharp
…
return rows.Skip(pageIndex * pageSize).Take(pageSize);  // Enumerable instead of List
```

And:

```sql
…
FROM dbo.MyTable
OFFSET @pageIndex * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY
```

…becomes:

```sql
…
FROM dbo.MyTable
WHERE id > @maxIdOfLastPage
OFFSET @pageIndex * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY
```

### Analysis
The concept of yielded results has been around for ages, but devs often still eagerly materialise collections when they don’t need to. It’s just what they’re used to, & they pay the price in the future.

The idea behind `WHERE` is to execute an index seek-scan to the “last position” (the page the user is currently on), then do seek-scan/scan/key lookups to finish the job.

Tada, problem solved! Big O goes from `O(p)` to:

```
O(n) = rowsReadForSeekScan + pageSize
        = r + p
```

Ah, that’s better. Now our scalability is tied to our DB index (instead of table row count). Double linked-list & B-trees FTW!

And this is known as “keyset pagination”.

# Why doesn’t everyone use keyset pagination?
It sure is awesome, but not all situations enable the use of it. Example, a dev might have to write ANSI-compliant SQL to run paginated queries across multiple database providers (such as SQL Server & Sqlite).

# The cloud kicks ass!
Certain cloud providers abstract the keyset pagination away from the consumer (developer). For Azure cosmos queries, Microsoft gives consumers the table rows & a “special string”. Then on subsequent calls, you add this “special string” to your request and BOOM, like MAGIC, you get paginated results for the next page.

These “special strings”, or *continuation tokens*, make developer life easier.

# Conclusion
Pagination method | I/O efficiency | Memory efficiency | Network efficiency | When to use
Offset pagination (database layer) | Bad | Good | Good | Need to write ANSI SQL
Offset pagination (server side) | Bad | Good (if yielded), bad otherwise | Good | Don’t need ANSI SQL & need scalability without high development cost
Offset pagination (client side) | Good | Bad | Good (don’t need extra API calls for subsequent pages) | Little/no memory & network constraints, no bottleneck with front-end dev resources, and need product scalability with fast backend releases
Keyset pagination (database layer) | Good | Good | Good | Always use this over offset pagination if supported
Keyset pagination (server side) | Good | Bad | Good | DB bottlenecking, or dont want to additional index on table exposed to significantly more writes than reads
Keyset pagination (client side) | Good | Good (if client has no memory constraint), bad otherwise | Good | Users use pagination frequently & deeply on your product, and you want to offload work onto client devices
Continuation token | Good | Good | Good | Always use this over keyset pagination if supported (lower code complexity & development cost)
