---
title: Exploring the reasoning behind System.Collections.Immutable
layout: post
author: Michael Nguyen
---
The `IReadOnlyCollection<T>` interface was introduced as a step toward immutability in .NET.

## Why IReadOnlyCollection<T>?
Developers may not want consumers mutating their data, because then it opens up a scenario that the original developer didn't design for, and this leads to potential bugs.

To give developers this option, the .NET team decided to expose all the fun mutating methods on [List<T>](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=netframework-4.8). This achieves the developer use-case, because [IReadOnlyCollection<T>](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ireadonlycollection-1?view=netframework-4.8) doesn't inherit from `List<T>`, and therefore objects implementing that interface wont expose mutate methods.

In simple terms, this means `List<T>` represents mutable collection, with `IReadOnlyCollection<T>` as its crime-fighting brother.

Example:
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

And thus, .NET has given us another defensive programming option to help us minimise bugs.

Or have they?

## Read-only collections can still be mutated
C# variables inheriting from [Object](https://docs.microsoft.com/en-us/dotnet/api/system.object?view=netframework-4.8) are just stack pointers holding a memory address.

Just like in C++, having 2 pointers share the same address range means that when you mutate using 1 pointer, the dereferencing operation for the 2nd pointer gives back the mutated value.

And we can achieve this in C#, even though it's an abstracted language. E.g.:
```c#
var items = new List<int> { 1 };
var wrapper = items.AsReadOnly();
items.Add(2);

foreach (var item in wrapper)
{
	Console.WriteLine(item);
}
Console.ReadKey();
```
![](X.png "Z")

This is because List<T> inherits from Object, which means that both `items` and `wrapper` are just stack pointers to the same address.

To refresh your mind - whilst the [IReadOnlyCollection<T>](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ireadonlycollection-1?view=netframework-4.8) API does not expose mutable methods, the [List<T>](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=netframework-4.8) API does.

So whilst you can't do:
```c#
var items = new List<int> { 1 };
var wrapper = items.AsReadOnly();
wrapper.Add(2);
```
![](X.png "Z")

You can do:
```c#
var items = new List<int> { 1 };
var wrapper = items.AsReadOnly();
items.Add(2);
```
![](X.png "Z")

Isn't this just great?

## Defensive programming to the rescue!
Prior to `System.Collections.Immutable`, the best scenario was to always set up a situation where consumers only get 1 `IReadOnlyCollection<T>` pointer:
```c#
private static IReadOnlyCollection<int> GetItems()
{
	var items = new List<int> { 1 };
	return items.AsReadOnly();
}

...

var wrapper = GetItems();
```

Here, the `items` variable goes out of scope & eventually garbage collected.

## Real immutability
An example of C# immutability is it's string interpolation feature:
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

.NET does NOT mutate the original memory address space of `var1`!

```
var var1 = "bleh";
var str = $"{var1} goes the weasle";
Console.WriteLine(var1);	// Prints "bleh"
Console.WriteLine(str);		// Prints "bleh goes the weasle"
```

By applying this same semantics to `IReadOnlyCollection<T>` methods, we could achieve a true immutable instance.

## Immutable instances!
And this is what happened. Microsoft sat around a table, stroking their beards with glee, and gave us `System.Collections.Immutable` - a namespace full of interfaces and classes that have mutable methods that return new instances.

Example:
```c#
var items = new[] { 1 }.ToImmutableList();
var wrapper = items.Add(2);
foreach (var item in wrapper)
{
	Console.WriteLine(item);
}
Console.ReadKey();
```
![](X.png "Z")

## Closing notes
By making [mutable methods pure](https://en.wikipedia.org/wiki/Pure_function), the .NET team has given developers a true immutability option.

Whilst instantiating new instances costs more time & space than modifying values in-memory, immutable colections prevent users from modifying data, and this in turn ensures that the code we designed is only exposed to the use cases that we initially expected at design time, therefore minimising bugs.
