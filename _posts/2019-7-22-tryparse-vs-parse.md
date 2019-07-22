## TryParse vs Parse - how .NET framework produces their behaviour internally for non-number strings
For parse failure:
..* TryParse returns false with out parameter = 0
..* Parse throws format exception

But what code inside .NET framework 4.8 produces this behaviour?

## TryParse
Below we look at overload TryParse(String, out Int32).

Starting from Int32.cs:TryParse(...), it eventually calls Number.cs:ParseNumber(...).

### Code flow summary inside .NET
Int32.cs:TryParse(String, out Int32)
```c#
// Parses an integer from a String. Returns false rather
// than throwing exceptin if input is invalid
// 
[Pure]
public static bool TryParse(String s, out Int32 result) {
	return Number.TryParseInt32(s, NumberStyles.Integer, NumberFormatInfo.CurrentInfo, out result);
}
```
Number.cs:TryParseInt32
	sets out result to 0 before progressing (so if it fails, result equals 0)
	passes in arg parseDecimal=false (this comes into play @ ParseNumber(...))
Number.cs:TryStringToNumber
Number.cs:ParseNumber
	since no stringbuilder, bigNumber = false
	eat away sign
	while current pointer is digit character
		add char to NumberBuffer
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

	digEnd = 0
	adds characters to NumberBuffer (i.e.: the ref NumberBuffer param)
	add null-terminator
```c#
if (bigNumber)
	sb.Append('\0');
else
	number.digits[digEnd] = '\0';
```

	try parse exponent
	set string param (passed by ref) to char pointer (address)
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
	
returns to Number.cs:TryParseInt32(), which then calls NumberToInt32(ref NumberBuffer, ref Int32)
	works backwards on NumberBuffer to set Int32 (which was passed by ref)

### Behavioural explanation
So if ParseNumber fails, the call to TryParseInt32 returns false.

And since TryParseInt32 only set the out var to default(int) (i.e.: 0), this is what produces a false & 0 result.

Lets compare this logic to this TryParse brother, Parse(String)

## Parse
Below we look at overload Parse(String).

Starting from Int32.cs:Parse(...), it follows a very different path but ends up calling the same function Number.cs:ParseNumber() to do the main work.

The difference is in HOW the return result from Number.cs:ParseNumber() is handled.

### Code flow summary inside .NET
Int32.cs:Parse(String)
Number.cs:ParseInt32
Number.cs:StringToNumber
Number.cs:ParseNumber
	(same as above)
	returned F = throws format exception
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
		throw new FormatException(Environment.GetResourceString("Format_InvalidString"));
	}
	...
}
```
	
### Behavioural explanation
Hence, ParseNumber is called again.

The difference is that:
..* When Number.cs:ParseNumber(...) returns false to TryParse, TryParse doesnt throw an exception and just returns false to its caller
..* When Number.cs:ParseNumber(...) returns false to StringToNumber, StringToNumber throws the format exception (bubbling all the way back to Int32.cs:Parse(...))