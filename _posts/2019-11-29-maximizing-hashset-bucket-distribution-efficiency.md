---
title: Maximizing HashSet bucket distribution efficiency
layout: post
author: Michael Nguyen
toc: true
---

The .NET HashSet operates around the idea of "buckets". The distribution of these buckets is affected by `Object.GetHashCode`, which generates hash codes for items.

In short, the hashing mechanism you use affects how objects get distributed into a hashset's buckets, and this affects lookup efficiency. I feel that this fact isnt given enough emphasis in todays world, so this blog post runs through this.

Explaining bucket distribution efficiency requires an understanding of how HashSets work with user-defined objects. So this post will look at:

* The role that hashing plays in sets
* How to use user-defined objects with [HashSet](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs)
* Incorrect implementations of the various equality functions that "break" .NET Core's `HashSet<T>`
* Correcting these implementations
* The bucket distribution efficiency of various implementations recommended by the internet

# Set hashing
In the context of sets (i.e.: hash tables), hashing gives devs an option to trade memory for performance & efficiency. This allows lookup operations that search `Sets` much quicker by leveraging the index operator.

The code examples that follow use [System.Collections.Generic.HashSet](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs) as an example collection, which implements `ISet<T>`.

# How are HashSets used?
The HashSet generic collection is often used to store user-defined objects. E.g.:

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

[HashSet.Add](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs#L231) uses various methods to figure out where to place the object in the hashset. These methods include (& are not limited to):

* `Object.Equals`
* `Object.GetHashCode`
* `IEquatable<T>.Equals`

User-defined objects need to implement the above 3 functions to maximize lookup efficiency in `HashSet<T>`. Sometimes i see StackOverflow posts missing 1 of these 3, which is a huge kick in the nuts. If you are using modern .NET & C# 2+, you should always implement all 3 to get the most bang for your buck.

Lets run through the first 2 now!

## Object.Equals & Object.GetHashCode
Both of these methods are implemented in [Object](https://source.dot.net/#System.Private.CoreLib/Object.cs,d9262ceecc1719ab), & are used to check equality (i'll come back to this fact later).

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

`Equals` is for normal equality stuff, whilst `GetHashCode` is used to determine which bucket the object should be in (i.e.: which "nested list" the object is in).

With 2/3 functions defined, lets run through the 3rd function: `IEquatable<T>.Equals`.

## IEquatable<T>.Equals
`Object.Equals` is a non-generic function. With modern .NET & C# 2+, we can now use generic functions.

With this luxury, we should also implement the generic version of `Object.Equals` too, which is `IEquatable<T>.Equals`.
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

Benefits:
* it allows .NET to [internally find our equal comparison logic](https://github.com/microsoft/referencesource/blob/master/System.Core/System/Collections/Generic/HashSet.cs#L398) more easily. In short, less [CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language)
* objects implementing an interface can often be substitued with similar objects implementing that interface (for functions accepting that interface). This tranlates into more malleable code via [covariance & contravariance](https://docs.microsoft.com/en-us/dotnet/standard/generics/covariance-and-contravariance).
* less [boxing](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing). Again, this blog post concerns efficiency, which makes boxing is a big-enough factor to consider.

So far, we've run through the definitions for:
* `Object.Equals`
* `Object.GetHashCode`
* `IEquatable<T>.Equals`

I've only given definitions so far - i will now reveal basic implementation details for these.

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

Tihs will do for now. Our code above *can* work, but has many problems.

# Consequences of incorrect overrides
.NET Core is smart but if you implement `Object.Equals` or `Object.GetHashCode` incorrectly, .NET Core won't function as expected. The .NET developers have given you the power to provide your own implementation, but with great power comes great responsibility.

For instance, lets check out the `HashSet<T>.Contains` .NET Core implementation:
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
          ...
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

  ...
  return false;
}
```

Pay attention to the `IF` condition in the for-loop. If we implemented `Equals` correctly but `GetHashCode` incorrectly, .NET Core's `HashSet.Add` & `HashSet.Contains` begin to operate in unpredictable ways.

## Incorrectly implementing Object.GetHashCode
Lets take our example class `MyObject`. Let's say we add a `Val` to a hashset:
```c#
var set = new HashSet<MyObject>;

var obj1 = new MyObject(Id: 1, Val: 1);
set.Add(obj1);
```

Sets only contain Unique values. So if we try to add another object with the same `Val`:
```c#
var obj2 = new MyObject(Id: 2, Val: 1);
set.Contains(obj2);
```

It can still be added if we make `GetHashCode` return `Id`:
```c#
public class MyObject2 : IEquatable<MyObject2>
{
    public int Id { get; }
    public int Val { get; }

    ...

    public override int GetHashCode()
    {
        return Id;
    }
}
```

Because if we look at the .NET Core source (posted earlier on), the hash code of the 1st object is 1, and the 2nd object hash is 2. So .NET will search for bucket 2 and see that nothing exists in that nested list and will think it doesnt exist in the set yet:
```c#
int hashCode = item == null ? 0 : InternalGetHashCode(comparer.GetHashCode(item));
for (int i = buckets[hashCode % buckets.Length] - 1; i >= 0; i = slots[i].next)
{
	if (slots[i].hashCode == hashCode && comparer.Equals(slots[i].value, item))
	{
		return true;
	}
	...
}
```

Specifically, the `slots[i].hashCode == hashCode` bit will return false and this makes `HashSet<T>.Contains` return `false`.

By incorrectly implementing `GetHashCode`, we've just caused a hard-to-detect bug and made `HashSet<T>.Add` a more leaky abstraction. You may think that incorrect implementations are hard to do, but a dev may not remember to update the method after altering a user-defined classes fields/properties.

## Incorrectly implementing Object.Equals/IEquatable.Equals
The same thing can be said for `Equals`. Lets say we make it return `Id`:
```c#
public class MyObject2 : IEquatable<MyObject2>
{
    public int Id { get; }
    public int Val { get; }

    ...

    public override bool Equals(object obj)
    {
        return Equals(obj as MyObject2);
    }

    public bool Equals(MyObject2 other)
    {
        if (other == null)
            return false;

        return Id == other.Id;
    }

    public override int GetHashCode()
    {
        return Val;
    }
}
```

Then we add 2 objects with different ID's (with the same `Val`) into 1 `HashSet<T>` bucket:
```c#
var set = new HashSet<MyObject>;

var obj1 = new MyObject(Id: 1, Val: 1);
set.Add(obj1);

var obj2 = new MyObject(Id: 2, Val: 1);
set.Contains(obj2);
```

The `Contains` call will return false, because even though they're now in the same bucket (`Val`), they have different `Id`. This can cause all sorts of problems, mainly by "breaking" .NET Core's implementation of `HashSet<T>.Contains`.

Specifically, the `comparer.Equals(slots[i].value, item)` segment will return false.

## What have we learned so far?
### Lesson 1: An Equals implementation with 99.9% correctness is not enough!
If we screw up `Equals`, even if its only 99.9% correct (e.g.: you check all class properties/fields except 1), you'll cause a massive problem down the line. A colleague is gonna hit you on the back of the head.

Look - `GetHashCode` just acts as a bucket ID generator. Theres nothing magical about it. So make sure GetHashCode is fast, & that your `Equals` implementation prioritizes correctness over speed.

Its usually a good idea to make `GetHashCode` generate an int based on ALL fields/properties.

### Lesson 2: never base GetHashCode on your Equals implementation
Some developers seem to implement `Object.Equals` & `IEquatable<T>.Equals` by using `Object.GetHashCode`. E.g.:
```c#
public override bool Equals(object obj)
{
    return Equals(obj as MyObject);
}

public bool Equals(MyObject other)
{
    if (other == null)
        return false;

    return GetHashCode() == other.GetHashCode();
}

public override int GetHashCode()
{
    ...
}
```

This is a huge problem. Good implementations of `GetHashCode` leverage `int` overflow supressing via `unchecked` and large multiplication operations to maximize the range of hash IDs the function can generate.

This means that 2 objects that may **not** be equal, may have the same hash code. Basically what im saying is this:
```c#
var hash1 = int.MaxValue;
var hash2 = int.MinValue;
const bool isEqual = hash1 == hash2;
```

So when you base `Equals` on `GetHashCode`, you could have 2 **INequal** objects that get put into the same bucket, & get detected as equalvalent. Youre gonna be in a for a massive headache when trying to debug these kinds of issues in prod.

If you ever do this, just book a vacation in advance because your gonna need it.

As said before, `Object.Equals` & `IEquatable<T>.Equals` need to have 100% correctness - never EVER sacrifice even 0.00001% correctness for a slightly faster math operation. Leave the speed to your `Object.GetHashCode` override.

### Lesson 3: ensure bucket distribution is efficient
What happens if you have 1000 buckets, but only 3 buckets contain 2000 items each? This leaves 997 buckets unused.

`HashSet` lookups use the hash code to quickly jump to a bucket, and then use `Equals` to compare each object in that bucket sequentually. Since `Equals` is `O(n)` (compared to the hash lookup of roughly `O(1)`),  by definition, it is not scalable for large N.

If we use an inferior `GetHashCode` implementation, we get this situation where lookups become slow in hashsets. Its a paradox, i know. Thankfully, the following sections will help you avoid this sticky situation.

# Bucket distribution efficiency
Now that all that is out of way, lets jump into bucket distribution efficiency! This all depends on the implementation we choose for `GetHashCode`.

We will start with the worst implementations, which are often [prominent](https://stackoverflow.com/a/9828186) in [StackOverflow threads](https://stackoverflow.com/a/70375). Then we will keep improving our implementation until we get a nice juicy gooey efficient bucket distribution.

The below examples use a specific hashset to test bucket distribution, which is generated using:
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

This generated data will allow us to test permutations & combinations of hashes.

Additionally, we will use a user-defined class containing 5 properties, and the same `Equals` override for our examples:
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

This is the rice-and-beans staple of the HashSet world. To identify the problems with this, we need to know what `XOR` does.

The [exclusive-or operator](https://hackernoon.com/xor-the-magical-bit-wise-operator-24d3012ed821) (XOR) just smooshes 2 bit sequences together to generate another bit sequence. When only one of the bits is 1, then 1 is generated else 0. E.g.:
```
01010011
11110001
```
becomes
```
10100010
```

The reason this is a bad `GetHashCode()` implementation is because different [combinations](https://en.wikipedia.org/wiki/Combination) still produce the same bit sequence. In other words, different [permutations](https://en.wikipedia.org/wiki/Permutation) of the same combination just give the same hash result.

For instance, reversing the bit sequence order in our previous example produces the same end-result:
```
11110001
01010011
```
```
10100010
```

So when you do a XOR for all the class properties, you get a biased distribution for similar value permutations. An example of this in C# is:
```c#
var a = new MyObject { PropertyA = 1, PropertyB = 2, PropertyC = 3 };
var b = new MyObject { PropertyA = 3, PropertyB = 2, PropertyC = 1 };
var c = new MyObject { PropertyA = 3, PropertyB = 1, PropertyC = 2 };
var d = new MyObject { PropertyA = 2, PropertyB = 1, PropertyC = 3 };
```

And guess what - since we are benchmarking efficiency we want to test the worst case. And this situation is pretty much the worst case! Running our hashset with this `GetHashCode` XOR implementation gives us some intersting statistics:

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 8                | 0.1976%            |

Damn son! Not even 1%! Visualized as radar chart:

![case 1 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-1.png)

See all that blue? Those lines represent a used bucket. See all that grey? Thats what shame looks like.

This is a very disturbing situation with many unused buckets. I don't wanna look at this circle anymore, so lets start improving our `GetHashCode` implementation.

## Case 2: XOR with co-primes, multiplication and overflow suppression
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

The improvements in this variant are:

* The `unchecked` block, which ensures integer overflows are suppressed. This means large values "wrap around".
* Multiplying the class properties. This causes larger numbers.
* Usage of co-prime numbers. Co-prime numbers means there is a lower chance of the multiplication operations producing the same result for similar combinations of field/property values.
* Setting a minimum for the hash. If a property is 0, we won't get a bias towards the 0<sup>th</sup> bucket

All 3 factors give us a much larger range of generated bucket hash IDs, & this translates into *wider* bucket distribution with the **same** efficiency:

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 8                | 0.1976%            |

![case 2 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-2.png)

Ok, so maybe that wasnt a mind-blowing move. In fact, we cant really see the wider bucket distribution on a radar chart. HOWEVER these improvements will form the foundation for building an optimum `Object.GetHashCode` implementation.

We are still sufferring from `XOR` producing the same hash code for property value combinations. We need to throw `XOR` out the window immediately.

## Case 3: using SUM in-place of XOR
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

We only replaced `XOR` with `+`, but look at the difference!

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 21               | 0.5186%            |

What a kick - woohoo! A 0.3% improvement. We are finally getting somewhere!!

![case 3 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-3.png)

Now all those property/field value combinations give different bits. And now, we will leverage the foundations from case #2 & #3 to kick it up a notch!

## Case 4: multiplying with larger values
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
By multiplying by larger values (& supressing overflows via `unchecked`), `GetHashCode()` will generate hash IDs within a larger range. Since we are basically benchmarking here, we care about the worst case, and in most realistic scenarios we could have very large amounts of data to put in our set.

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 1950             | 48.1600%           |

![case 4 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-4.png)
	
Look at that efficient distribution!

This variant is essentially the implementation [recommended by Jon Skeet](https://stackoverflow.com/a/263416), and is superior to the XOR variant when using large HashSets.

Use this, honestly its gold. GOLD I SAY!

## Case 5: tailored GetHashCode
Case 4 is great - like, *really* great.

But if YOU know YOUR exact scenario, and the exact data being put into your HashSet, you have a special opportunity where its possible to write a `GetHashCode` implementation tailored to your specific data to maximize your bucket distribution efficiency.

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
We KNOW that each object will have a [unique property value permutation](https://en.wikipedia.org/wiki/Permutation):
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
will have hashcode of `12345`.

With this knowledge, we can improve our `GetHashCode` even more to work better for our specific dataset. And with this new `GetHashCode` implementation we get a nice kick:

| Total buckets | Buckets occupied | % buckets occupied |
| ------------- | ---------------- | ------------------ |
| 4049          | 2310             | 56.8288%           |

![case 5 bucket distribution](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/bucket-distribution-case-5.png)

A better bucket distribution efficiency with an even more *consistent* distribution. Not bad, not bad at all. :-)

# Closing notes
Hashsets give faster lookups at the cost of memory.

Developers may sometimes forget to override `IEquatable<T>.Equals`, `Object.Equals` or `Object.GetHashCode`. Whilst all 3 equality functions appear redundant, they are all necessary to maximizing efficiency when working with HashSets.

You need to focus on a good (fast) `GetHashCode` implementation for your data situation.

For the vast majority of cases, using `unchecked` multiplications with large prime numbers (& summing the result of these operations across all class fields/properties) will give efficient bucket distributions.

Inefficient bucket distributions essentially lead to more `O(n)` HashSet operations, which nullifies the whole point of HashSet - super dooper fast lookups.

The beauty of `HashSet<T>` can only be utilized through efficient bucket distributions.
