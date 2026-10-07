# Numbers, BigDecimal/BigInteger, and Money
## Why double/float are wrong for money
- Floating-point types (float, double) are not suitable for representing monetary values due to precision issues. They can introduce rounding errors in calculations, which is unacceptable in financial applications.
- `double`/`float` use binary floating-point (IEEE 754) — a format that can only exactly represent sums of powers of two. Most decimal fractions (0.1, 0.3, ...) have no exact binary representation, so they're stored as the closest representable approximation.

```java
double a = 0.1;
double b = 0.2;
System.out.println(a + b); // 0.30000000000000004
```
- This isn't a rounding "bug" — it's the format working exactly as designed. For scientific/graphics work the tiny error is irrelevant. 
- For a trading platform's account balance, it's unacceptable: errors compound across thousands of transactions, and two clients with "the same" balance can fail an == check.
- Hence, we can use `BigDecimal` for monetary values. `BigDecimal` provides arbitrary-precision decimal numbers and allows for precise control over rounding behavior.

## `BigDecimal` — exact decimal arithmetic
- `BigDecimal` represents a number as two parts: an arbitrary-precision integer (unscaledValue) and a scale (number of digits after the decimal point). 
- 123.45 is stored as "unscaled value" → 12345, "scale" → 2. Because the underlying integer is exact (backed by `BigInteger`, see below), there's no binary-approximation error at all.
- `BigDecimal` is immutable — every arithmetic method (add, subtract, multiply, divide) returns a new instance, same discipline as String.

### The constructor trap
```java
BigDecimal bad  = new BigDecimal(0.1);      // 0.1000000000000000055511151231257827021181583404541015625
BigDecimal good = BigDecimal.valueOf(0.1);  // 0.1
BigDecimal best = new BigDecimal("0.1");    // 0.1
```

- `new BigDecimal(double)` converts the already-imprecise `double` bit pattern exactly — faithfully preserving the binary rounding error rather than fixing it. 
- `BigDecimal.valueOf(double)` goes through `Double.toString()` first, giving the "expected" decimal. 
- The String constructor is the safest and most common in practice, since it never touches a double at all: for an account balance read from a database or user input, you want the string/decimal path, not a double path.

**Domain example — `Account` balance:**
```java
BigDecimal balance = new BigDecimal("1000.00");
BigDecimal deposit = new BigDecimal("250.50");
BigDecimal updated = balance.add(deposit); // 1250.50, exact
```

### Division — the one operation that can throw
- Addition, subtraction, and multiplication on `BigDecimal` always terminate exactly. 
- Division doesn't: `1 / 3` has no finite decimal representation. 
- Calling `divide` without specifying rounding, when the result is non-terminating, throws `ArithmeticException`:
```java
new BigDecimal("1").divide(new BigDecimal("3")); // ArithmeticException
new BigDecimal("1").divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP); // 0.33
```
- You must supply either a scale + `RoundingMode`, or a `MathContext` (precision + rounding mode, used when you want to cap total significant digits rather than decimal places). 
- `RoundingMode` is an enum — `HALF_UP` (standard "round half away from zero," what most people mean by "normal rounding"), `HALF_EVEN` (banker's rounding — minimizes cumulative bias over many roundings, used in some financial standards), `DOWN` (truncate toward zero), `UP`, `FLOOR`, `CEILING`, etc.

### equals vs compareTo — the classic gotcha
- `BigDecimal`'s `equals()` method checks both value and scale. Two `BigDecimal`s with the same numeric value but different scales are not equal:
```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("1.00");
System.out.println(a.equals(b)); // false
System.out.println(a.compareTo(b)); // 0 — they are numerically equal
```
- Use `compareTo()` for numeric equality checks, and `equals()` only when you care about both value and scale (e.g., when using `BigDecimal` as a key in a `HashMap` or `HashSet`).

## BigInteger — arbitrary-precision whole numbers
- `BigInteger` is `BigDecimal`'s integer-only sibling: an immutable, arbitrary-precision integer, not bounded by long's 64 bits. 
- It's what `BigDecimal` uses internally for its unscaled value. 
- Use it when a quantity could genuinely exceed `Long.MAX_VALUE` (~9.2 × 10¹⁸) — large factorial/combinatorial computations, cryptographic key material, or hash-like accumulations — which is rare in ordinary application code but common in exactly those two domains.

```java
BigInteger a = new BigInteger("123456789012345678901234567890");
BigInteger b = BigInteger.valueOf(2).pow(100); // 2^100, exact
```

- Same immutability discipline, same add/subtract/multiply/divide/mod method family, divide by zero throws `ArithmeticException` (no non-terminating-decimal concern here since there's no decimal point — just a plain divide-by-zero check)

## Choosing the Right Numeric Type

| Need                                                                        | Type           |
|-----------------------------------------------------------------------------|----------------|
| Money, prices, anything needing exact decimal arithmetic                    | `BigDecimal`   |
| Whole numbers that might exceed `long` range                                | `BigInteger`   |
| Counters, loop indices, ordinary bounded quantities                         | `int` / `long` |
| Non-monetary measurements where tiny error is tolerable (physics, graphics) | `double`       |


