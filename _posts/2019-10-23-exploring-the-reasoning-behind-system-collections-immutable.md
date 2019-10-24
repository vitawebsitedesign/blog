---
title: Read-only and Immutable are separate C# concepts
layout: post
author: Michael Nguyen
---
The `System.Collections.Generic` .NET namespace has a `IReadOnlyCollection<T>` interface.

But when `System.Collections.Immutable` enters the game, things start to get confusing. Aren't read-only collections immutable?

Before diving into that question; what does read-only even mean in C#?

## Read-only
> In a ref readonly method return, the [readonly modifier](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/readonly) indicates that method returns a reference and writes are not allowed to that reference.

The key phrases here are "reference" - specifically, "writes are not allowed to that reference".

Calling methods that mutate state on an `IReadOnlyCollection<T>` instance (i.e.: a read-only ref type) raise exceptions:
```c#
var mutableList = new List<int> { 1 };
var readOnlyList = mutableList.AsReadOnly();
((IList<int>)readOnlyList).Add(1);		// Methods like Add are only exposed on List<T>, not IReadOnlyCollection<T>
```

...or
```c#
var mutableList = new List<int> { 1 };
var readOnlyList = mutableList.AsReadOnly();
(readOnlyList as IList<int>).Add(1);	// Methods like Add are only exposed on List<T>, not IReadOnlyCollection<T>
```
...or
```c#
var mutableList = new List<int> { 1 };
var readOnlyList = mutableList.AsReadOnly();
(readOnlyList as List<int>).Add(1);		// NullReferenceException
```

Which is intentionally caused by the [ReadOnlyCollection .NET implementation](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/icollection.cs,a9bf1395d3addc77):
```c#
internal sealed class ReadOnlyCollection<T> ...
{
	internal ReadOnlyCollection(T[] items) {
		_items = items;
		...
	}
	
	void ICollection<T>.Add(T value) {
		throw new NotSupportedException();
	}
	
	void ICollection<T>.Add(T value) {
		throw new NotSupportedException();
	}

	void ICollection<T>.Clear() {
		throw new NotSupportedException();
	}

	bool ICollection<T>.Remove(T value) {
		throw new NotSupportedException();
	}
}
```

However there is a caveat. Since C# is (in some ways) an abstraction of C++, it's possible to have 2 variables referencing the same address space.

## Mutating shared address space
* One cannot mutate state using read-only variables.
* But one CAN mutate state using non read-only variables.

Given these 2 points, it is therefore possible to mutate the address space referenced by a read-only variable:
```c#
var items = new List<int> { 1 };	// Normal variable
var wrapper = items.AsReadOnly();	// Read-only variable
items.Add(2);						// Use normal variable to modify same address space (thats also referenced by the read-only variable)

foreach (var item in wrapper)
{
	Console.WriteLine(item);
}
Console.ReadKey();

// Prints: 1, 2
```

## Why bother with read-only when i can just use List<T>?
Developers may not want consumers mutating the data they return, because it opens up additional scenarios that the original developer didn't design for, and this leads to potential future bugs.

This is because if a consumer can do something, they will probably do it.

To allow developers to restrain consumers, the .NET team decided to expose all the state modification methods on [List](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=netframework-4.8) instead of [IList](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ilist-1?view=netframework-4.8).

And thanks to the convenient [GenericList.AsReadOnly](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs,2b710ab0bc8866ad) method, one can convert an entire collection into a read-only equivalent easily:
```c#
public ReadOnlyCollection<T> AsReadOnly() {
	...
	return new ReadOnlyCollection<T>(this);
}

...

internal sealed class ReadOnlyCollection<T> ...
{
	internal ReadOnlyCollection(T[] items) {
		_items = items;
		...
	}
}
```

## Why bother with read-only collections if state can still be modified?
Read-only collections "help" prevent consumers doing something stupid, but theres no guarantee of immutability here.

This is simply because `List<T>` state modification methods aren't Pure Functions. In other words, they modify memory in-place instead of returning a new instance.

Whilst more performant, it still leaves a door open that allows consumers to exploit the read-only memory space.

And this is why `System.Collections.Immutable` was introduced.

## Pure Functions in C#
Before diving into `System.Collections.Immutable`, we can already see [Pure Functions](https://en.wikipedia.org/wiki/Pure_function) achieving true immutability in C# 6.

I'm talking about the string interpolation feature:
```c#
var var1 = "bleh";
var str = $"{var1} goes the weasle";
```

In the above example, the string interpolation calls .NET's `System.String.Concat(Object, Object)` [code](https://referencesource.microsoft.com/#mscorlib/system/string.cs,8281103e6f23cb5c), which returns a new `String` instance:
```c#
public static String Concat(String str0, String str1) {
	...
	int str0Length = str0.Length;
	String result = FastAllocateString(str0Length + str1.Length);
	FillStringChecked(result, 0,        str0);
	FillStringChecked(result, str0Length, str1);
	
	return result;
}

unsafe private static void FillStringChecked(String dest, int destPos, String src)
{
	...
	fixed(char *pDest = &dest.m_firstChar)
		fixed (char *pSrc = &src.m_firstChar) {
			wstrcpy(pDest + destPos, pSrc, src.Length);
		}
}

internal static unsafe void wstrcpy(char *dmem, char *smem, int charCount)
{
	Buffer.Memcpy((byte*)dmem, (byte*)smem, charCount * 2); // 2 used everywhere instead of sizeof(char)
}
```
`Concat` calls `FastAllocateString` (which returns a new String instance), then `wstrcpy` is called twice to copy both strings into the new String instance.

> .NET does NOT mutate the original memory address space of `var1`!

```c#
var var1 = "bleh";
var str = $"{var1} goes the weasle";
Console.WriteLine(var1);	// Prints "bleh"
Console.WriteLine(str);		// Prints "bleh goes the weasle"
```

By applying this same design to `IReadOnlyCollection<T>`, we could achieve **true** immutability.

## Full immutability!
And this is what happened. Microsoft sat around a table, stroking their beards with glee, and gave us `System.Collections.Immutable` - a namespace full of interfaces and classes that have mutable methods that return new instances:
```c#
var items = new[] { 1 }.ToImmutableList();
var wrapper = items.Add(2);		// Add() returns a new instance, rather than modifying the memory in-place
```

This is proven by cross-checking [the .NET source](https://github.com/dotnet/corefx/blob/master/src/System.Collections.Immutable/src/System/Collections/Immutable/ImmutableList.cs#L64):
```c#
[Pure]
public static ImmutableList<TSource> ToImmutableList<TSource>(this IEnumerable<TSource> source)
{
	var existingList = source as ImmutableList<TSource>;
	if (existingList != null)
	{
		return existingList;
	}

	return ImmutableList<TSource>.Empty.AddRange(source);
}
```

## Should you use it?
Just because you see the word "immutable" doesn't mean you should *always* use it.

After all, switching from mutable to immutable collections is trading performance for [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)).

So if you require performance during state modification operations, use mutable collections. Otherwise, strive for immutable collections.

## Closing notes
Whilst instantiating new instances costs more time & space than modifying values in-memory, immutable colections prevent users from modifying data.

This in turn ensures that the code we designed is only exposed to the use cases that we initially expected at design time, therefore minimising bugs.
