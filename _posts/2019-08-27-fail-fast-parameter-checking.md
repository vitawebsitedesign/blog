---
title: When to check input parameters in fail-fast systems
layout: post
author: Michael Nguyen
toc: true
---
The best part about fail-fast systems is the reduced debugging effort.

The stack trace shows the earliest point of failure, and you can just shove in breakpoints and see what happens.

When using parameter checking to achieve fail-fast, is can be done through defensive programming.

Lets see how this is done.

## Disadvantages of fail-fast
Before going on to the "how", we should note that the primary disadvantage of fail-fast is that additional code is written, and this translates into increased maintenance cost.

When the system changes in the future, perhaps input that used to be invalid is now valid, and now the code that checks input must be updated (as well as any code that handled the failure).

For example, assume we have a database table:

```sql
CREATE TABLE taco (
	id INT PRIMARY KEY IDENTITY(1, 1)
);
```

## With fail-fast:

```c#
private int GetTaco(int tacoId)
{
	if (tacoId > 0) {
		return _db.tacos.Find(t => t.Id == tacoId);
	}
	throw new ArgumentOutOfRangeException(nameof(tacoId));		// Defensively programming here - we assume default will NOT be handled downstream
}
```

Fast forwarding into the future, perhaps the column will change so that tacoId of 0 is valid:
```sql
CREATE TABLE taco (
	id INT PRIMARY KEY IDENTITY(-1, 1)
);
```

And so now the parameter checking code needs to be updated to:

```c#
private int GetTaco(int tacoId)
{
	if (tacoId > -2) {
		return _db.tacos.Find(t => t.Id == tacoId);
	}
	throw new ArgumentOutOfRangeException(nameof(tacoId));		// Defensively programming here - we assume default will NOT be handled downstream
}
```

And this bears an expense.

## Should we check parameters here?
The answer mostly depends on the consequence when invalid parameters are passed in, as well as the code that runs after the function call.

Parameter checking should always occur prior to data mutation operations, because data mutation operations can cause severe headaches when data goes sideways.

For data retrieval operations, its usually up to the system behaviour. For example, if invalid parameters silently invalid data to be shown in a UI, you should check the parameter because the bug severity is higher due to the lower detection.

But if effect were, for example, no data being shown in the UI for invaild parameters, parameter checking becomes less useful. In fact, its actually redundant as you'd have 2 sets of code to handle the same situation.

## Unnecessary preflight checks prior to data mutation operations
With the above arguments in mind, there is 1 exception for data mutation ops.

And this is writing code that duplicates the "checks parameters" bit in the database engine.

This is known as a preflight request. Consider the below example (with entity framework):

```c#
private async Task<int> GetTaco(int id)
{
	var alreadyExists = _db.taco.Any(t => t.Id == id);
	if (!alreadyExists) {
		var newTaco = new Taco(id);
		_db.taco.Add(newTaco);
		await _db.SaveChangesAsync();
		return newTaco.id;
	}
	return default;
}
```

Theres no point to checking the parameter tacoId. It doesnt determine the value of the mutated data. Theres just more code in the way which just replicates behaviour the database will do when you insert a duplicate primary key value.

So with this code just adding an extra request to the server, perhaps a lower-cost alternative equivalent would serve best:

```c#
private async Task<int> GetTaco(int id)
{
	var newTaco = new Taco(id);
	_db.taco.Add(newTaco);

	try
	{
		await _db.SaveChangesAsync();
	}
	catch
	{
		return default;
	}
	return newTaco.id;
}
```

Now we dont duplicate behaviour that is already done by the database engine.

Its less code, its faster due to 1 less db request, and we shift responsibility to the database engine (which presumably will do a better job anyway).

If we tried to insert a duplicate primary key value, a db exception will throw, and we return `default`.

The exact same behaviour as we had before, but with less code & complexity.

The parameter checking (i.e.: preflight) [was just unnecessary weight](https://softwareengineering.stackexchange.com/a/378671). And you might feel like a better programmer by checking data before these kinds of data mutation operations, but in the end, its less about being a good programmer, and more about making good future-proof code for the team.
