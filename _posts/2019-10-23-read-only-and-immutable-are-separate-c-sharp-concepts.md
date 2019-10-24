---
title: Read-only and Immutable are separate C# concepts
layout: post
author: Michael Nguyen
---
The `System.Collections.Generic` .NET namespace has a `IReadOnlyCollection<T>` interface.

But when `System.Collections.Immutable` enters the game, things start to get confusing. Aren't read-only collections immutable?

Before diving into that question; what does read-only even mean in C#?

## Read-only
> "In a ref readonly method return, the [readonly modifier](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/readonly) indicates that method returns a reference and writes are not allowed to that reference"

The key phrases here are "reference" - specifically, "writes are not allowed to that reference".

Calling methods that mutate state on an `IReadOnlyCollection<T>` instance (i.e.: a read-only ref type) raises exceptions.

However there *is* a caveat. Since C# is (in some ways) an abstraction of C++, it's possible to have 2 variables referencing the same address space.

## Mutating shared address space
To rehash what read-only means:

* One cannot mutate state using read-only variables.
* But one CAN mutate state using non read-only variables.

Given these 2 points, it is therefore possible to mutate the address space referenced by a read-only variable:
```c#
var items = new List<int> { 1 };	// Normal variable
var wrapper = items.AsReadOnly();	// Read-only variable
items.Add(2);						// Use normal variable to modify same address space (thats also referenced by the read-only variable)

foreach (var item in wrapper)
{
	Console.WriteLine(item);	// Prints: 1, 2
}
Console.ReadKey();
```

And thus, we have "modified" a read-only collection.

## Why use read-only collections instead of mutable collections?
Developers may not want consumers mutating the data they return, because it opens up additional scenarios that the original developer didn't design for, and this leads to potential future bugs.

And let's face it, if a consumer can do something, they will probably end up doing it.

To help developers in this use case, the .NET team decided to expose all the state modification methods on [List](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=netframework-4.8) instead of [IList](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ilist-1?view=netframework-4.8).

```c#
var mutableList = new List<int> { 1 };
var readOnlyList = mutableList.AsReadOnly();
((IList<int>)readOnlyList).Add(1); // Static compilation error: methods like Add are only exposed on List<T>, not IReadOnlyCollection<T>
```

```c#
var mutableList = new List<int> { 1 };
var readOnlyList = mutableList.AsReadOnly();
(readOnlyList as IList<int>).Add(1); // Static compilation error: methods like Add are only exposed on List<T>, not IReadOnlyCollection<T>
```

## Why bother with read-only collections if state can still be modified?
Read-only collections "help" encourage consumers to do the right thing, but theres no guarantee that they'll play nicely.

Think of speed limits in school zones. It encourages drivers to slow down and minimise fatalities, but theres nothing there to physically enforce a car's max speed when the driver is in a school zone.

And this is why `System.Collections.Immutable` was introduced.

## Pure Functions?
[Pure Functions](https://en.wikipedia.org/wiki/Pure_function) examples already exist in C# 6.

For example, [GenericList.AsReadOnly](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs,2b710ab0bc8866ad):
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

Or the string interpolation feature from C# 6:
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

In the above case, .NET does NOT mutate the original memory address space of `var1`. Let me illustrate this in a simpler, yet equivalent example:

```c#
var var1 = "bleh";
var str = $"{var1} goes the weasle";
Console.WriteLine(var1);	// Prints "bleh"
Console.WriteLine(str);		// Prints "bleh goes the weasle"
```

It seems that when we apply the concept of Pure Functions to methods that mutate state, one achieves **true** immutability.

## Full immutability!
And this is what happened. Microsoft sat around a table, stroking their beards with glee, and gave us `System.Collections.Immutable` - a namespace full of interfaces and classes that make mutable methods return new instances (instead of modifying state):
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

And now consumers can't do something stupid. One less thing to worry about in the complex world of programming.

## Nothing is free in the programming world
This is all about tradeoffs. Just because you see the word "immutable" doesn't mean you should *always* use it.

After all, switching from mutable to immutable collections is trading performance for [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)).

So if you require performance during state modification operations, use mutable collections. Otherwise, strive for immutable collections.

## Closing notes
Whilst instantiating new instances costs more time & space than modifying values in-memory, immutable colections prevent users from modifying data.

This in turn ensures that the code we designed is only exposed to the use cases that we initially expected at design time, therefore minimising bugs.
