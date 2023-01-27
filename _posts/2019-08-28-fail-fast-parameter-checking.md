---
title: Achieving fail-fast cheaply & effectively
layout: post
author: Michael Nguyen
toc: true
---
The best part about fail-fast systems is the reduced debugging effort.

The stack trace shows the earliest point of failure, and you can just shove in breakpoints and see what happens.

A non-fail-fast system would failure alot later, and the stack trace entries might be so late that you'll probably have to backtrack your initial breakpoint position with guesswork.

One way to achieving fail-fast is parameter checking - a cheap and effective method.

## Disadvantages of fail-fast
Before going on to the "how to achieve fail-fast cheap & effectively", we should note that the primary disadvantage of fail-fast is that additional code is written, and this translates into increased maintenance cost.

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

Fast forwarding into the future, perhaps our `IDENTITY` column will change so that taco ID of 0 is valid:

```sql
CREATE TABLE taco (
	id INT PRIMARY KEY IDENTITY(0, 1)
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

And.... boned. An extra expense for the team.

## When parameter checking stops becoming a cheap & effective tool in achieving fail-fast
This answer mostly depends on the consequence when invalid parameters are passed in, as well as system behaviour after the function call.

Parameter checking is best done prior to data mutation operations, because data mutation operations can cause severe headaches when life goes sideways.

For data retrieval operations, it usually depends on the system behaviour severity.

For example, if invalid parameters silently invalid data to be shown in a medical UI, you should check the parameter because the bug severity is higher (potential loss of human life) due to the lower detection and incorrect signals.

But if the system were, for example, not show data in the UI for invaild parameters, then we lose the "cheap & effective" benefits from parameter checking. In fact, its actually redundant as you'd have 2 sets of code to handle the same situation:

* Returning sentinel value when invalid parameters returned
* Then UI receives this sentinel value and portrays system state accordingly

When you could just have:

* UI portrays system state accordingly depending on received value

## Unnecessary preflight checks prior to data mutation operations
With the above arguments in mind, there is 1 exception when it comes to data mutation ops.

And this is checking parameters to determine app logic, when the parameters are already checked on the database side.

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

Theres no point to checking the parameter id. The database already checks parameters in the form of data integrity.

Theres just more code on our side, which just replicates behaviour the database will do when you insert a duplicate primary key value.

So with this code just adding an extra request to the server, perhaps a lower-cost alternative equivalent would serve best?:

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

Now we dont check parameters twice, and its just done once by the database engine.

Its:

* Less code (lower maintainability cost for entire codebase)
* Its faster due to 1 less db request
* And we shift responsibility to the database engine (which presumably will do a better job anyway).

And if we tried to insert a duplicate primary key value, a db exception will throw, and we return `default`.

The exact same behaviour as we had before, but with less code.

By NOT checking parameters, we have achieved fail-fast in a cheaper & more effective way.

## But i should always check parameters, right? To be a good programmer?
Perhaps you doing an unnecessary preflight gives you a ego kick, but [its just unnecessary weight](https://softwareengineering.stackexchange.com/a/378671).

But in the end, its less about being a good programmer, and more about making good future-proof code for the team.
