---
title: Analysing .NET HashSet bucket distribution
layout: post
author: Michael Nguyen
---

.NET Core has the Set abstract data type, & sets operate around this idea of "buckets".

We will look at `GetHashCode` & `Equals` in detail by:

* Incorrect implementations that "break" .NET Core
* Correct implementations & why
* The bucket distribution efficiency of various implementations

# What
We will use [System.Collections.Generic.HashSet](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs) for example collections.

The idea behind hashtables is to trade some memory for a lookup performance & efficiency boost.

Objects [added](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs#L231) get placed into a bucket.

Hashsets contain buckets, and each bucket contains a list. This means that HashSet is essentially a list of lists.

This allows lookup operations to jump to the object much quicker by leveraging the index operator.

# Working with HashTables
.NET HashTables can be generic, so you can shove your own objects into them. Example:

```c#
public class MyObject1
{
	public int Id { get; }
	public int Val { get; }

	public MyObject1(int id, int val)
	{
		Id = id;
		Val = val;
	}
}
```

[HashSet<T>.Add](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs#L231) uses various tools to figure out where to place the object in the hashset:

* Object.Equals
* Object.GetHashCode
* IEquatable<T>.Equals

## Object.Equals & Object.GetHashCode
Both of these methods are implemented in [Object](https://source.dot.net/#System.Private.CoreLib/Object.cs,d9262ceecc1719ab), & are used to check equality.

When making our own classes, we should override these to be correct.

`Object.Equals` is a **non-generic** function:
```c#
public override bool Equals(object obj)
{
	...
}
```

`Object.GetHashCode` is a **non-generic** function:
```c#
public override int GetHashCode()
{
	...
}
```

`Equals` is for normal equality stuff, but `GetHashCode` is used in hashsets to "jump" to the correct nested list.

## IEquatable<T>.Equals
We have gone over 2 non-generic functions used for equality, however modern .NET now allows us to use generic functions.

We should implement a generic version of `Object.Equals` too, because it allows .NET to internally find our equal comparison logic more efficiently.

This is done by implementing `IEquatable<T>.Equals`:
```c#
public class MyObject1 : IEquatable<MyObject1>
{
	...

	public bool Equals(MyObject1 other)
	{
		...
	}
}
```

## Putting it all together
So now we have all 3 equality functions to maximize lookup efficiency. Now we can implement them.

```c#
public class MyObject1 : IEquatable<MyObject1>
{
	public int Id { get; }
	public int Val { get; }

	public MyObject1(int id, int val)
	{
		Id = id;
		Val = val;
	}

	public override bool Equals(object obj)
	{
		return Equals(obj as MyObject1);
	}

	public bool Equals(MyObject1 other)
	{
		if (other == null)
			return false;

		return Val == other.Val;
	}

	public override int GetHashCode()
	{
		return Id;
	}
}
```

As said before, modern .NET has introduced generics so we can just make our `Object.Equals` override use our generic `IEquatable<T>.Equals` implementation.

Lucky us - devs in the old days didnt have this luxury!

We will go over improvements for the above code later on. For now, i wanna gloss over the dangers of incorrect implementations.

# .NET is NOT a unicorn with a crystal ball
.NET Core is smart but if you implement 1 of the 3 functions incorrectly, its not gonna function as expected.

Specifically, 
