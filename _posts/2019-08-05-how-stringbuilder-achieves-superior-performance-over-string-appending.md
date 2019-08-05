---
title: How StringBuilder achieves superior performance over string appending
layout: post
author: Michael Nguyen
---
StringBuilder is faster than String for append operations.

## Benchmark

```c#
private static void DoString(int loop)
{
	string str = "a";
	for (var i = 0; i < loop; i++)
	{
		str += "a";
	}
}
```

```c#
private static void DoStringBuilder(int loop)
{
	var sb = new StringBuilder("a");
	for (var i = 0; i < loop; i++)
	{
		sb.Append("a");
	}
}
```

[Repo](https://github.com/vitawebsitedesign/stringbuilder-string-benchmark)

![Benchmark (textual)](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/string-vs-stringbuilder-benchmark-summary.jpg "Benchmark (textual)")

![Benchmark (visual)](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/string-vs-stringbuilder-benchmark-chart.jpg "Benchmark (visual)")

Lets check it out.

## String
Below is .NET 4.8 source for String.Concat(String, String) (i.e.: += overload):

```c#
public static String Concat(String str0, String str1) {
	...
	String result = FastAllocateString(str0Length + str1.Length);
	...
	return result;
}
```

So when you read the msdn String documentation:

> A String object is called immutable ...

[https://docs.microsoft.com/en-us/dotnet/api/system.string?view=netframework-4.8#immutability-and-the-stringbuilder-class](https://docs.microsoft.com/en-us/dotnet/api/system.string?view=netframework-4.8#immutability-and-the-stringbuilder-class)

The call to FastAllocateString (unmanaged assembly code) is how string achieves immutability when using the + overload.

## StringBuilder
.NET 4.8 source for StringBuilder.Append(String):

```c#
public StringBuilder Append(String value) {
	...
	unsafe {
		fixed (char* valuePtr = value)
		fixed (char* destPtr = &chunkChars[chunkLength])
			string.wstrcpy(destPtr, valuePtr, valueLen);
	}
	m_ChunkLength = newCurrentIndex;
	...
}
```

and...

```c#
internal static unsafe void wstrcpy(char *dmem, char *smem, int charCount)
{
	Buffer.Memcpy((byte*)dmem, (byte*)smem, charCount * 2); // 2 used everywhere instead of sizeof(char)
}
```

## Analysis

We can see that for `StringBuilder.Append`, instead of re-allocating memory for a new string instance & returning a pointer to the new address, it just expands the capacity at the existing memory address and appends content (i.e.: the `value` param) at the end via pointer.

We can also see the ["square own capacity by 2 when i get too big"](https://docs.microsoft.com/en-us/dotnet/api/system.text.stringbuilder?view=netframework-4.8#how-stringbuilder-works) code.

Hence, both of these points prove that how StringBuilder operates compared to `String.Concat(String, String)`.

## Should i always use StringBuilder instead of String += ?
By performing memory capacity expansion & string addition in-place (instead of allocating memory at new addresses), the .NET team has given developers an option to sacrifice immutability for speed.

You'll need to decide on your own, but generally `var str = "a" + "b";` is more readable than `var sb = new StringBuilder("a"); var str = sb.Append("b");`

At lower quantities, the performance difference is negligible compared to the readability gain.

At higher quantities, the cost/benefit flips around, so you should use StringBuilder then.

Basically once the performance difference becomes visible to the naked eye, its probably a good idea to use StringBuilder instead.
