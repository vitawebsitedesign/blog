---
title: TryParse vs Parse - how .NET framework produces their behaviour internally for non-number strings
layout: post
---
## TryParse vs Parse - how .NET framework produces their behaviour internally for non-number strings
For parse failure:
1. TryParse returns false with out parameter = 0
2. Parse throws format exception

But what code inside .NET framework 4.8 produces this behaviour?

Lets look inside it using https://referencesource.microsoft.com/#mscorlib

## TryParse
Below we look at overload TryParse(String, out Int32).

Starting from Int32.cs:TryParse(...), it eventually calls Number.cs:ParseNumber(...).

### Code flow summary inside .NET
Int32.cs:TryParse(String, out Int32)

```c#
public static bool TryParse(String s, out Int32 result) {
	return Number.TryParseInt32(s, NumberStyles.Integer, NumberFormatInfo.CurrentInfo, out result);
}
```


1. Number.cs:TryParseInt32
2. sets out result to 0 before progressing (so if it fails, result equals 0)
3. passes in arg parseDecimal=false (this comes into play @ ParseNumber(...))
4. Number.cs:TryStringToNumber
5. Number.cs:ParseNumber
6. since no stringbuilder, bigNumber = false
7. eat away currency sign (so we just get the 1st digit of the actual number to be parsed)
8. while current pointer is digit character
9. add char to NumberBuffer

```c#
...
while(true) {
	if ((ch >= '0' && ch <= '9') ...) {
		...
		if (bigNumber)	// Note: bigNumber is a false param passed in by Int32.cs:TryParse()
			sb.Append(ch);
		else
			number.digits[digCount++] = ch;	// "number" refers to the NumberBuffer
		...
	}
	...
}
...
```

4. digEnd = 0
5. adds characters to NumberBuffer (i.e.: the ref NumberBuffer param)
6. add null-terminator

```c#
if (bigNumber)
	sb.Append('\0');
else
	number.digits[digEnd] = '\0';
```

7. try parse exponent
8. set string param (passed by ref) to char pointer (address)

```c#
if ((state & StateDigits) != 0) {
	...
	if ((state & StateParens) == 0) {
		...
		str = p;	// str is the char pointer, p is the NumberBuffer pointer
		return true;
	}
	...
}
```

At this point, it returns to Number.cs:TryParseInt32(), which then calls NumberToInt32(ref NumberBuffer, ref Int32).

Which then works backwards on NumberBuffer to set Int32 (which was passed by ref).

### Behavioural explanation
So if ParseNumber fails, the call to TryParseInt32 returns false.

And since TryParseInt32 only set the out var to default(int) (i.e.: 0), this is what produces a false & 0 result.

Lets compare this logic to this TryParse brother, Parse(String)

## Parse
Below we look at overload Parse(String).

Starting from Int32.cs:Parse(...), it follows a very different path but ends up calling the same function Number.cs:ParseNumber() to do the main work.

The difference is in HOW the return result from Number.cs:ParseNumber() is handled.

### Code flow summary inside .NET
1. Int32.cs:Parse(String)
2. Number.cs:ParseInt32
3. Number.cs:StringToNumber
4. Number.cs:ParseNumber
5. returns false for invalid strings (that represent numbers) = throws format exception

```c#
internal unsafe static Single ParseSingle(String value, NumberStyles options, NumberFormatInfo numfmt) {
	...
	if (!TryStringToNumber(value, options, ref number, numfmt, false)) {
		String sTrim = value.Trim();
		if (sTrim.Equals(numfmt.PositiveInfinitySymbol)) {
			return Single.PositiveInfinity;
		}
		if (sTrim.Equals(numfmt.NegativeInfinitySymbol)) {
			return Single.NegativeInfinity;
		}
		if (sTrim.Equals(numfmt.NaNSymbol)) {
			return Single.NaN;
		}

		// This is the line we care about
		throw new FormatException(Environment.GetResourceString("Format_InvalidString"));
	}
	...
}
```
	
### Behavioural explanation
Hence, ParseNumber is called again.

The difference is that:
1. When Number.cs:ParseNumber(...) returns false to TryParse, TryParse doesnt throw an exception and just returns false to its caller
2. When Number.cs:ParseNumber(...) returns false to StringToNumber, StringToNumber throws the format exception (bubbling all the way back to Int32.cs:Parse(...))