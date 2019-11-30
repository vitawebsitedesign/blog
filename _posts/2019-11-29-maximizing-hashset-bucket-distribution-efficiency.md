---
title: Maximizing HashSet bucket distribution efficiency
layout: post
author: Michael Nguyen
---

![bucket distribution overview](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-overview.jpg)

The .NET Core [Set](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.iset-1?view=netframework-4.8) abstract data type operates around the idea of "buckets". Bucket distribution is affected by `Object.GetHashCode`, `Object.Equals` & `IEqualityComprarer<T>.Equals`.

This post will look at:

* Incorrect implementations of these functions that "break" .NET Core's [HashSet](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs)
* Correcting these implementations
* The bucket distribution efficiency of various implementations recommended by the internet

Before diving into any of this, we need to understand the role that hashing plays in sets.

# Hashing
In the context of sets, hashing trades memory for performance & efficiency.

This allows lookup operations to search sets much quicker by leveraging the index operator.

This examples below will use [System.Collections.Generic.HashSet](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs) as an example set.

# Working with HashSets
The HashSet generic collection is often used to store uer-defined objects. E.g.:

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

[HashSet.Add](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs#L231) uses various tools to figure out where to place the object in the hashset:

* Object.Equals
* Object.GetHashCode
* IEquatable<T>.Equals

User-defined objects need to implement the above functions to maximize lookup efficiency in `HashSet<T>`.

## Object.Equals & Object.GetHashCode
Both of these methods are implemented in [Object](https://source.dot.net/#System.Private.CoreLib/Object.cs,d9262ceecc1719ab), & are used to check equality.

When making our own classes, we should override both of these **non-generic functions**:
```c#
public override bool Equals(object obj)
{
	...
}
```
```c#
public override int GetHashCode()
{
	...
}
```

`Equals` is for normal equality stuff, whilst `GetHashCode` is used in sets to "jump" to the correct nested list (i.e.: bucket).

You may now be asking what the difference between `Object.Equals` & `IEquatable<T>.Equals` is.

## IEquatable<T>.Equals
We have gone over 2 **non-generic** functions used for equality, however modern .NET now allows us to use generic functions.

With this luxury, We should then also implement a generic version of `Object.Equals` too, because:
* it allows .NET to internally find our equal comparison logic more efficiently
* objects implementing an interface can often be substitued with similar objects implementing that interface (for functions accepting that interface)

This **generic** "version" is `IEquatable<T>.Equals`:
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

Now you know the 3 equality functions required to maximize lookup efficiency.

So far i've only given definitions, so now i will identify an example implementation detail of all 3.

## Putting it all together
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

The above code *can* work, but has many problems.

# Consequences of incorrect implementations
.NET Core is smart but if you implement 1 of these 3 equality functions incorrectly, .NET Core won't function as expected. For example, lets check out the `HashSet<T>.Contains` implementation:
```c#
public bool Contains(T item)
{
  ...

  if (buckets != null)
  {
      ...
      IEqualityComparer<T>? comparer = _comparer;
      if (comparer == null)
      {
        ...
      }
      else
      {
          int hashCode = item == null ? 0 : InternalGetHashCode(comparer.GetHashCode(item));
          // see note at "HashSet" level describing why "- 1" appears in for loop
          for (int i = buckets[hashCode % buckets.Length] - 1; i >= 0; i = slots[i].next)
          {
              if (slots[i].hashCode == hashCode && comparer.Equals(slots[i].value, item))
              {
                  return true;
              }
              ...
          }
      }
  }

  // either _buckets is null or wasn't found
  return false;
}
```
Lets take our example class `MyObject`. If we put 2 objects with the same `Val` into a hashset, but with different `Id`s, and we make `GetHashSet()` return `Id`, we can have an incorrect `GetHashCode` implementation when we check if a `Val` already exists in the hashset:

```c#
public class MyObject2 : IEquatable<MyObject2>
{
    public int Id { get; }
    public int Val { get; }

    public MyObject2(int id, int val)
    {
        Id = id;
        Val = val;
    }

    public override bool Equals(object obj)
    {
        return Equals(obj as MyObject2);
    }

    public bool Equals(MyObject2 other)
    {
        if (other == null)
            return false;

        return Id == other.Id && Val == other.Val;
    }

    public override int GetHashCode()
    {
        return Id;
    }
}
```

Hence, the `slots[i].hashCode == hashCode` bit will return false and `HashSet<T>.Contains` will return an unexpected result.

This is because even though both objects have the same `Val`, `GetHashCode()` is based on `Id`. The object will be in a different bucket, and calling `Hashset<T>.Contains(some2ndObjectWithSameValue)` will return false.

What about `Equals`? If we want to check if an object with the same `ID` already exists in a hashset, & we make `Equals` based on `Val`, then we will have an incorrect `Equals` implementation. Hence, the `comparer.Equals(slots[i].value, item)` bit will return false and all is doomed:
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
        return Val;
    }
}
```

For example, if we have a hashset with an object with Id `1`, then `Hashset<T>.Contains(anotherObjectWithDifferentVal)` would return false (even if it had `Id` of 1).

Always make sure both are implemented correctly.

That said, this raises an interesting question. If both functions serve as equality, why are there 2?

## GetHashCode VS Equals
A misconception is that both functions should behave like `Equals`.

This is wrong - `GetHashCode` just acts as a bucket ID generator. So make sure GetHashCode is fast, & that your `Equals` implementation prioritizes correctness over speed.

# Bucket distribution efficiency
Now that all that is out of way, lets jump into bucket distribution efficiency! This all depends on the implementation we choose for `GetHashCode`.

We will start with the worst implementation, then keep improving it until we get a nice juicy gooey even bucket distribution.

If you want to follow along, the code to generate my sample hashsets is below:
```c#
var set = new HashSet<MyObject4>();
var limit = 5;
for (var a = 0; a < limit; a++)
{
    for (var b = 0; b < limit; b++)
    {
        for (var c = 0; c < limit; c++)
        {
            for (var d = 0; d < limit; d++)
            {
                for (var e = 0; e < limit; e++)
                {
                    set.Add(new MyObject4(a, b, c, d, e));
                }
            }
        }
    }
}
```

And we will use the below `Equals` implementation:
```c#
public override bool Equals(object obj)
{
    return Equals(obj as MyObject4);
}

public bool Equals(MyObject4 other)
{
    if (other == null)
        return false;

    return A == other.A
        && B == other.B
        && C == other.C
        && D == other.D
        && E == other.E;
}
```

Lets get biz-a.

## Case 1: Basic XOR
```c#
public override int GetHashCode()
{
    return A ^ B ^ C ^ D ^ E;
}
```

This is the rice-and-beans staple of the HashSet world & is often marked as a correct Stackoverflow answer. Before stating how bad this is, i'll need to dive into `XOR`.

The exclusive-or operator (XOR) just smooshes 2 bit sequences together to generate another bit sequence. When only one of the bits is 1, then 1 is generated else 0. E.g.:
```
01010011
11110001
```
becomes
```
10100010
```

The reason this is a bad `GetHashCode()` implementation is that different orders still produce the same bit sequence. Flipping the previous example gives us the same result:
```
11110001
01010011
```
```
10100010
```

So when you do a XOR for all the class properties, you get a horrendous distribution for similar property value pairs. E.g.:
```c#
var a = new MyObject { PropertyA = 1, PropertyB = 2, PropertyC = 3 };
var b = new MyObject { PropertyA = 3, PropertyB = 2, PropertyC = 1 };
var c = new MyObject { PropertyA = 3, PropertyB = 1, PropertyC = 2 };
var d = new MyObject { PropertyA = 2, PropertyB = 1, PropertyC = 3 };
```

And guess what, we are benchmarking efficiency so we care about the worst case. And this is the worst case!

Stats:

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 8                | 0.1976%            |

Damn son not even 1%! Visualized as radar chart:

![case 1 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-1.png)

This is a very disturbing situation, so lets quickly move onto a slightly-improved version.

## Case 2: unchecked XOR with prime multiplication
```c#
public override int GetHashCode()
{
    unchecked
    {
        const int prime = 23;
        var hash = (A * prime)
            ^ (B * prime)
            ^ (C * prime)
            ^ (D * prime)
            ^ (E * prime);
        return 17 + hash;
    }
}
```

The 3 improvements in this variant are:

* The `unchecked` block, which ensures overflows are suppressed. This means large values "wrap around".
* Multiplying the class properties. This causes larger numbers.
* Usage of co-prime numbers. Co-prime numbers means there is a lower chance of the multiplication operations producing the same result for similar property values.

All 3 factors give us a much larger range of bucket hash IDs, & this translates into *wider* bucket distribution with the same *efficiency*:

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 8                | 0.1976%            |

![case 2 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-2.png)

As you can see, we're getting better but this variant still suffers from the `XOR` producing the same hash code for property value combinations. We need to throw `XOR` out the window immediately.

## Case 3: do SUM instead of XOR
```c#
public override int GetHashCode()
{
    unchecked
    {
        const int prime = 23;
        var hash = (A * prime)
            + (B * prime)
            + (C * prime)
            + (D * prime)
            + (E * prime);
        return 17 + hash;
    }
}
```

We just replaced `XOR` with `+`, and damn does this make a difference!

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 21               | 0.5186%            |

What a kick - woohoo! We are finally getting somewhere!!

![case 3 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-3.png)

## Case 4: multiply with larger values
```c#
public override int GetHashCode()
{
    unchecked
    {
        const int prime = 10007;

        var hash = 17;
        hash = hash * prime + A;
        hash = hash * prime + B;
        hash = hash * prime + C;
        hash = hash * prime + D;
        hash = hash * prime + E;
        return hash;
    }
}
```
By multiplying by larger values (& supressing overflows via `unchecked`), `GetHashCode()` will generate hash IDs within a larger range. Since we are basically benchmarking here, we care about the worst case, and in most realistic scenarios we could have very large HashSets.

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 1950             | 48.1600%           |

![case 4 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-4.png)
	
Look at that distribution! Hnnnggggg.

This variant is the implementation recommended by Jon SKeet on StackOverflow, and is superior to the XOR variant when using HashSets in the worst-case scenario.

Use this, honestly its gold. Theres also another great variant Jon posts which he takes from a Java book.

## Case 5: tailored GetHashCode
Case 4 is great, like, really great.

But if YOU know YOUR exact scenario, and the exact data being put into the HashSet, you have a special opportunity where its possible to write a `GetHashCode` implementation tailored to your specific data to maximize your bucket distribution efficiency.

Lets take our example from before. The data we generate is very specific:
```c#
var set = new HashSet<MyObject4>();
var limit = 5;
for (var a = 0; a < limit; a++)
{
    for (var b = 0; b < limit; b++)
    {
        for (var c = 0; c < limit; c++)
        {
            for (var d = 0; d < limit; d++)
            {
                for (var e = 0; e < limit; e++)
                {
                    set.Add(new MyObject4(a, b, c, d, e));
                }
            }
        }
    }
}
```
We KNOW that each object will have a unique property permutation. With this knowledge, we can craft a better `GetHashCode` for this specific dataset:
```c#
public override int GetHashCode()
{
    unchecked
    {
        var str = $"{A}{B}{C}{D}{E}";
        return int.Parse(str);
    }
}
```
For example, the object:
```c#
new MyObject
{
    A = 1,
    B = 2,
    C = 3,
    D = 4,
    E = 5
}
```
will have hashcode of `12345`. And with this implementation we get a nice kick:


| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 2310             | 56.8288%           |

![case 5 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-5.png)

# What if i wanna be lazy and use GetHashCode for my Equals implementations?
Some developers seem to implement `Object.Equals` & `IEquatable<T>.Equals` by using `Object.GetHashCode`.

Good implementations of `GetHashCode` leverage overflow supressing via `unchecked` and large multiplication operations to maximize the range of hash IDs the function can generate.

This means that 2 objects that may **not** be equal, may have the same hash code.

So if you do this, 2 inequal objects could be marked equal, and youre gonna be in a for a massive headache when trying to debug issues in prod.

As said at the start of this, `Object.Equals` & `IEquatable<T>.Equals` need to focus on correctness, not speed. Leave speed to `Object.GetHashCode()`.

This is because `HashSet` lookups use the hash code to quickly jump to a bucket, and then use `Equals` to compare each object in that bucket. `Equals` is `O(n)` (compared to the hash lookup of roughly `O(1)`), so by definition it is not scalable for large N. So focus on speed for hash code generation. Theres no room for error when it comes to `Equals`.

# Closing notes
Hashsets give faster lookups at the cost of memory.

Developers may sometimes forget to override `IEquatable<T>.Equals`, `Object.Equals` & `Object.GetHashCode`. Whilst all 3 equality functions appear redundant, they are all necessary to maximizing efficiency when placed inside HashSets.

Once all 3 functions are implemented CORRECTLY, then you need to focus on a good `GetHashCode` implementation for your situation. For the vast majority of cases, using `unchecked` multiplications with large prime numbers (& combining the result of these operations across all class fields/properties) will give "safe" & efficient bucket distributions.

Inefficient bucket distributions essentially lead to more `O(n)` HashSet operations, which nullifies the whole point of HashSet - super dooper fast lookups.
