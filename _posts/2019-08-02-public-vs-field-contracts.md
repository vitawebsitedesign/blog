---
title: Public vs Field contracts
layout: post
author: Michael Nguyen
toc: true
---
Consider below:

## Public field
```c#
public class A {
	public int var;
}
```

vs

## Default auto getter property
```c#
public class A {
	public int Var { get; }
}
```

and consider how they are both accessed:

```c#
var instance = new A();
Console.WriteLine(A.Var);
```

Even though default auto property getters are touted by some as code smell.

But they do serve a purpose - malleable code

## Malleable (i.e.: future proof)
After fulfilling functional requirements, non-functional requirements may exist, even if not explicitly stated.

For instance, a system that requires high availability means that code might need to cooperate with a fault-tolerance system module.

Malleability is a non-functional requirement that determines business cost of future changes.

And hence, properties (even auto public getters) may be preferred to fields.

## How c# auto properties achieve malleability
To change retrieval logic for a public field, its not possible to leave the contract unaffected.

This is because we need code to run during field access, which is not possible for fields. But IS for auto getter property.

Stuff accessing it would need to change. And this can lead to a massive mountain of work.

## Examples (after auto property compilation)
### Example 1: changing the value returned based on state
```c#
public class A {
	private bool _state = false;
	public int GetVar() {
		if (_state) {
			return var;
		}
		return var + 1;
	}
}

var a = new A();
Console.WriteLine(a.GetVar());	// This change must happen in all references
```

### Example 2: validating the value and throwing if invalid
```c#
public class A {
	private bool _state = false;
	public int GetVar() {
		if (_state) {
			return var;
		}
		throw new DataException(nameof(_state));
	}
}

var a = new A();
Console.WriteLine(a.GetVar());	// This change must happen in all references
```

## Above examples with properties
With the contract unchanged, less code needs to change (yay)

Again, the below examples are post compilation of auto properties.

### Example 1: changing the value returned based on state
```c#
public class A {
	private bool _state;
	private int _var;
	
	public int Var
	{
		get
		{
			if (_state) {
				return var;
			}
			return var + 1;
		}
	}
}

var a = new A();
Console.WriteLine(a.Var);	// No changes, yay
```

### Example 2: validating the value and throwing if invalid
```c#
public class A {
	private bool _state;
	private int _var;
	
	public int Var
	{
		get
		{
			if (_state) {
				return var;
			}
			throw new DataException(nameof(_state));
		}
	}
}

var a = new A();
Console.WriteLine(a.Var);	// No changes, yay
```

## So are auto property getters code smell compared to public fields?
Nope. Even though there are no immediate benefits, code always changes, driven by a business need to change requirements to beat competitors.

Since the driving force behind it is a necessity for the business (to survive), it should always be assumed that code WILL change.

And since writing the couple extra characters to implement auto properties is financially cheap for a business, i think its naive to think that public auto properties are "code smell" in all scenarios.