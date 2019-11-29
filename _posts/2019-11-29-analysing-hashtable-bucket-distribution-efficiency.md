---
title: Analysing .NET HashSet bucket distribution efficiency
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
.NET Core is smart but if you implement 1 of the 3 functions incorrectly, its not gonna function as expected:
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
For example, lets take our example class `MyObject`. If we put 2 objects with the same `Val` into a hashset, but different `Id`s, and we make `GetHashSet()` return `Id`, we can have an incorrect `GetHashCode` implementation when we check if a `Val` already exists in the hashset. Hence, the `slots[i].hashCode == hashCode` bit will return false and all is doomed:
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

This is because even though both objects have the same `Val`, `GetHashCode()` is based on `Id`. The object will be in a different bucket, and calling `Hashset<T>.Contains(some2ndObjectWithSameValue)` will return false.

What about `Equals`? If we want to check if an object with the same `ID` already exists in a hashset, & we make `Equals` based on `Val`, then we will have an incorrect `Equals` implementation. Hence, the `comparer.Equals(slots[i].value, item)` bit will return false and all is doomed (again):
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

<TODO IMG>

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

All 3 factors give us a much larger range of bucket hash IDs, & this translates into better bucket distribution:

<TODO IMG>

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

<TODO IMG>

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

<TODO IMG>
	
Look at that distribution! Hnnnggggg.

This variant is the implementation recommended by Jon SKeet on StackOverflow, and is superior to the XOR variant when using HashSets in the worst-case scenario.

# You have been warned
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

Inefficient bucket distributions essentially lead to more O(n) HashSet operations, which nullifies the whole point of HashSet - super dooper fast lookups.
