---
title: Read-only is just an implementation of immutablity
layout: post
author: Michael Nguyen
toc: true
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
Read-only means:

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

And thus, we have "modified" a read-only collection. If this is possible, then why does `IReadOnlyCollection<T>` exist?

## Why even bother?
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

Whilst this sets up a "do not touch" barrier for consumers, it still does not solve the shared address space modification problem.

Look i'll be honest. Read-only collections just "help" encourage consumers to do the right thing, pushing them in the right direction. But theres no guarantee that they'll play nicely.

Think of speed limit rules for school zones. Whilst it encourages drivers to slow down and minimise fatalities, theres nothing there to physically enforce a car's max speed when the driver is in a school zone.

And this is why `System.Collections.Immutable` was introduced. To enforce consumer behaviour by converting state modification methods into pure functions.

## Pure function examples
Before diving into `System.Collections.Immutable`, it's useful to look at the application of pure functions.

In fact, [pure function](https://en.wikipedia.org/wiki/Pure_function) examples already exist. E.g. [GenericList.AsReadOnly](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs,2b710ab0bc8866ad):
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

Or perhaps you fancy the string interpolation feature from C# 6:
```c#
var var1 = "bleh";
var str = $"{var1} goes the weasle";
```

In the above example, the string interpolation calls [.NET's String.Concat(Object, Object) code](https://referencesource.microsoft.com/#mscorlib/system/string.cs,8281103e6f23cb5c), which returns a **new** `String` instance:
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

Explanation: `Concat` calls `FastAllocateString` (which returns a new String instance), then `wstrcpy` is called twice to copy both strings into the new String instance.

In other words - it's a pure function, and that is what makes `System.String` immutable.

At this point, strong evidence exists that pure functions allow one to prevent state modification by making mutation methods return new instances instead.

It's almost as if pure functions is all the .NET team needs to bring devs **true** immutability.

## True immutability
And this is what happened. Microsoft sat around a table, stroking their beards with glee, and started using pure functions.

The end result? `System.Collections.Immutable`. A namespace full of interfaces and classes that make mutable methods return new instances (instead of modifying state):
```c#
var items = new[] { 1 }.ToImmutableList();
var wrapper = items.Add(2);		// Add() returns a new instance, rather than modifying the memory in-place
```

This is proven by cross-checking [the .NET source](https://github.com/dotnet/corefx/blob/a866d9643f23ddd2f47762e02ccdd2e387a8e5c8/src/System.Collections.Immutable/src/System/Collections/Immutable/ImmutableList_1.cs#L19):
```c#
public sealed partial class ImmutableList<T> : ...
{
	[Pure]
	public ImmutableList<T> Add(T value)
	{
		var result = _root.Add(value);
		return this.Wrap(result);
	}
	
	...
			
	[Pure]
	private ImmutableList<T> Wrap(Node root)
	{
		if (root != _root)
		{
			return root.IsEmpty ? this.Clear() : new ImmutableList<T>(root);
		}
		else
		{
			return this;
		}
	}
}
```

And just like that, *true immutability* has been achieved.

Now consumers can't do something stupid. Hey, who's complaining? - it gives you one less thing to worry about in the complex world of programming.

Gosh, it seems so powerful. Does this mean you should rewrite all your code to use Immutable collections instead?

## There is never a free lunch
Every software decision is about trading off something for something. Just because you see the word "immutable" doesn't mean you should start touching yourself and use it without thought.

After all, switching from mutable to immutable collections trades efficiency for [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)).

But basically, unless you require supreme performance during state modification operations, strive for the true immutable collections.

## Closing notes
Read-only refers to preventing write methods being called on a variable, & Immutability refers to pure methods returning new instances.

Whilst instantiating new instances costs more time & space than modifying values in-memory, immutable colections prevent consumers from tampering with data they're not meant to.

This ensures that the code we designed is only exposed to the use cases that we initially expected at design time, therefore minimising sneaky bugs.
