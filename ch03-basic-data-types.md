# 3. Basic Data Types

Underneath, a computer knows only bits: fixed-width groups of them, called words, that it can add, compare, shift, and move around. Everything else, from numbers and text to images and invoices, is an interpretation that programs place on those bits. A programming language's data types are its vocabulary of interpretations. At one end are types that map almost directly onto the hardware, such as 64-bit integers and IEEE floating-point numbers. At the other are types built for people, such as strings that understand the world's writing systems.

Swift is unusual in one respect: it has no truly built-in types. `Int`, `Double`, `Bool`, and `String` are structs defined in the standard library, in Swift, using the same features available to your own code. The compiler gives them special treatment only where it matters for speed, mapping their operations onto single machine instructions, so they're as efficient as the primitive types of C. In every other way they're ordinary types, and you can extend them with methods and protocol conformances of your own, as Chapter 6 shows.

This chapter covers the *basic* types: numbers, Booleans, and strings, along with the literals that create them. Chapter 4 covers *composite* types such as arrays, dictionaries, structs, and tuples. Classes and actors, the reference types, come up in Chapters 6 and 9, and protocol types in Chapter 7.

## 3.1. Integers

Swift has signed and unsigned integers of four fixed sizes: `Int8`, `Int16`, `Int32`, and `Int64`, and their unsigned counterparts `UInt8` through `UInt64`. Swift 6 added `Int128` and `UInt128`. Two more types, `Int` and `UInt`, have the natural word size of the platform, which is 64 bits on every current desktop, server, and phone, and 32 bits on some embedded targets.

`Int` is the type to use by default. It's the type of array indices and counts, of string lengths, and of integer literals when nothing else determines their type. Swift's guidelines recommend `Int` even for quantities that can't be negative, such as counts and sizes, because mixing signed and unsigned values requires conversions at every turn, and because unsigned subtraction is a trap waiting to happen (we'll see one shortly). The sized and unsigned types are for when the size matters: binary file formats, network protocols, hardware registers, bit manipulation, and C interoperability.

`Int` and `Int64` are distinct types even where they have the same size, so moving a value from one to the other requires an explicit conversion.

Signed integers use two's-complement representation, so an *n*-bit signed integer holds values from −2<sup>n−1</sup> to 2<sup>n−1</sup>−1, and an unsigned one from 0 to 2<sup>n</sup>−1. Each type reports its range through `min` and `max`:

```swift
print(Int16.min, Int16.max)  // "-32768 32767"
print(UInt32.max)  // "4294967295"
```

Here are Swift's binary operators, in order of decreasing precedence:

```
<< >>                          bitwise shift
* / % &* &                     multiplication
+ - &+ &- | ^                  addition
..< ...                        range formation
is as as? as!                  casting
??                             nil-coalescing
== != < <= > >= ~= === !==     comparison
&&                             logical AND
||                             logical OR
?:                             ternary conditional
= += -= *= etc.                assignment
```

The levels differ from C's in a few places worth remembering: shifts bind more tightly than multiplication, `&` groups with multiplication, and `|` and `^` group with addition. When in doubt, parenthesize.

The arithmetic operators work on integers and floating-point numbers alike, except the remainder operator `%`, which is for integers only. Integer division truncates toward zero, so `7 / 2` is `3` and `-7 / 2` is `-3`, and the remainder takes the sign of the dividend, so `-7 % 2` is `-1`. For a remainder that's always non-negative, as when wrapping an index around a circular buffer, you need a little extra arithmetic: `((i % n) + n) % n`.

### 3.1.1. Overflow

When an arithmetic result doesn't fit in its type, the operation *overflows*. Languages disagree about what should happen then. In C, signed overflow is undefined behavior; in Java and Go, the result silently wraps around. Swift does neither. Overflow is treated as a bug, and the program *traps*, stopping at once:

```swift
func brighten(_ level: UInt8, by step: UInt8) -> UInt8 {
    level + step
}

print(brighten(250, by: 10))  // traps: arithmetic overflow
```

An overflow trap prints no message of its own; a debugger, or a crash report, shows the cause as an arithmetic overflow. When the compiler can see the overflow, as in `let b: UInt8 = 250 + 10`, it reports an error at build time instead.

The reasoning is that an overflowed value is almost never what anyone intended, and silently producing one has caused real security holes: a length calculation wraps around to a small number, a buffer is allocated too small, and data is written past its end. Overflow checks are cheap, since the processor sets a flag on overflow anyway, and the optimizer removes the ones it can prove unnecessary.

Sometimes, though, there are other things you might want, and Swift makes each of them explicit. Hash functions, checksums, and pseudo-random generators want *wrapping* arithmetic, which the overflow operators `&+`, `&-`, and `&*` provide. Graphics code often wants *saturating* arithmetic, which clamps to the type's range. And some algorithms just need to *know* whether overflow happened:

```swift
let level: UInt8 = 250
print(level &+ 10)  // "4": wraps around, keeping the low 8 bits
print(UInt8(clamping: Int(level) + 10))  // "255": saturates at the maximum

let (sum, overflowed) = level.addingReportingOverflow(10)
print(sum, overflowed)  // "4 true"
```

### 3.1.2. Comparison and Bitwise Operators

Integers of the same type are compared with `==`, `!=`, `<`, `<=`, `>`, and `>=`, which produce a `Bool`. These operators come from the `Equatable` and `Comparable` protocols, which the basic types all conform to, as do many other types; we'll meet the protocols properly in Chapter 7. There's also unary minus, for negation, and unary plus, which does nothing but is occasionally useful for symmetry.

The bitwise operators treat an integer as a row of bits:

```
&     bitwise AND
|     bitwise OR
^     bitwise XOR
~     bitwise NOT (prefix)
<<    left shift
>>    right shift
```

A classic use of bits is the Unix file mode, which packs nine permission flags (read, write, and execute, for the file's owner, its group, and everyone else) into the low nine bits of an integer, conventionally written in octal. The function below turns a mode into the familiar `rwxr-xr--` notation by testing each bit in turn:

```swift
// swiftpl/ch3/permissions
/// Formats the low nine bits of a Unix file mode as in `ls -l`.
func permissions(_ mode: UInt16) -> String {
    let symbols: [Character] = ["r", "w", "x"]
    var s = ""
    for bit in stride(from: 8, through: 0, by: -1) {
        let isSet = mode & (1 << bit) != 0
        s.append(isSet ? symbols[(8 - bit) % 3] : "-")
    }
    return s
}

print(permissions(0o754))  // "rwxr-xr--"
```

The expression `1 << bit` produces a number with only that bit set, and `mode & (1 << bit)` is nonzero exactly when `mode` has the bit set too. The other operators correspond to common operations on sets of permissions:

```swift
// swiftpl/ch3/permissions (continued)
let mode: UInt16 = 0o754
print(permissions(mode | 0o002))  // "rwxr-xrw-": grant others write (union)
print(permissions(mode & ~0o444))  // "-wx--x---": remove all read bits (difference)
print(permissions(mode ^ 0o111))  // "rw-r--r-x": toggle every execute bit
print(permissions(0o777 & ~0o022))  // "rwxr-xr-x": apply the common umask 022
```

`|` sets bits, `&` combined with `~` clears them, and `^` flips them. For sets of a few named flags, the `OptionSet` protocol of Section 3.6 wraps exactly these operations in a friendlier interface.

In `x << n` and `x >> n`, `n` gives the number of positions to shift. Swift's shifts are *smart shifts*: a shift by more than the type's width yields 0 (or −1, for a right shift of a negative number), and a negative shift amount shifts the other way, instead of being undefined as in C. The shift amount may be any integer type. When the last bit of speed matters, the *masking shifts* `&<<` and `&>>` behave like the hardware, using only the low bits of the shift amount.

Left shifts fill the vacated low bits with zeros. Right shifts of unsigned values fill the high bits with zeros, but right shifts of signed values copy the sign bit into them. That's the right behavior for arithmetic, but it's a surprise when you meant to treat the value as plain bits, which is one reason to use unsigned types for bit patterns.

Every integer type also offers bit-level properties: `nonzeroBitCount` (the number of 1 bits), `leadingZeroBitCount`, `trailingZeroBitCount`, `byteSwapped`, and `bitWidth`. Each compiles to a single instruction on most processors.

### 3.1.3. Conversions

Swift never converts between numeric types implicitly. Even a lossless conversion from `Int32` to `Int64` must be written out, and the operands of an arithmetic operator must have the same type:

```swift
let width: Int32 = 640
let scale = 1.5
let scaled = width * scale  // compile error
```

```
error: binary operator '*' cannot be applied to operands of type 'Int32' and 'Double'
```

You fix it by deciding which type the computation should use, and saying so:

```swift
let scaled = Double(width) * scale  // 960.0
```

The rule is strict on purpose. Implicit conversions are where silent truncations, sign mix-ups, and precision losses come from, and requiring them to be written makes each one visible.

Converting to a narrower type deserves special care, because the value might not fit. A plain conversion like `Int16(x)` checks and traps if it doesn't. When out-of-range values are expected, the integer types offer initializers that say what to do instead:

```swift
let reading = 70_000
Int16(reading)  // traps: "Not enough bits to represent the passed value"
Int16(exactly: reading)  // nil: returns an optional
Int16(clamping: reading)  // 32767: saturates to the nearest representable value
Int16(truncatingIfNeeded: reading)  // 4464: keeps the low 16 bits, as C would
```

Converting a floating-point value to an integer discards the fractional part, rounding toward zero, and traps if the value is out of range, infinite, or not a number. To round first, say so:

```swift
let temperature = 98.6
print(Int(temperature))  // "98"
print(Int(temperature.rounded()))  // "99"
print(Int(exactly: temperature) as Any)  // "nil": not a whole number
```

The conversion in the other direction, `Double(someInt)`, is the one numeric conversion that can lose information silently: integers above 2<sup>53</sup> can't all be represented exactly as a `Double`.

Here's why `Int` is preferred even for values that can't be negative. Suppose a program tracks the sizes of two buffers as unsigned values and wants to know how much larger one is:

```swift
let used: UInt = 3
let capacity: UInt = 5
let spare = used - capacity  // traps: the true answer, -2, isn't a UInt
```

With `Int`, the subtraction produces −2, which is a perfectly good answer to check for. With `UInt`, the only possible outcomes are a trap (in Swift) or a huge wrapped-around number (in C), and both are bugs. Reserve unsigned types for bit patterns and external formats.

Integer literals can be written in decimal, in binary with a `0b` prefix, in octal with `0o`, or in hexadecimal with `0x`; underscores may separate digits anywhere for readability, as in `1_000_000` or `0xFF_FF`. A leading zero alone means nothing special, so `0755` is the decimal number 755, not an octal one, which avoids a classic C mistake.

To format an integer in another base, use `String(_:radix:uppercase:)`:

```swift
print(String(255, radix: 2))  // "11111111"
print(String(0o755, radix: 8))  // "755"
print(String(48879, radix: 16), String(48879, radix: 16, uppercase: true))  // "beef BEEF"
```

For column alignment, zero padding, and other control, Foundation's `String(format:)` accepts the formatting directives of C's `printf`:

```swift
print(String(format: "[%5d] [%-5d] [%05d] [%x]", 42, 42, 42, 255))
// "[   42] [42   ] [00042] [ff]"
```

Characters aren't integers in Swift, but every Unicode scalar, the unit Unicode assigns numbers to (Section 3.5), has a numeric code point, available through its `value` property:

```swift
let e: Unicode.Scalar = "é"
let euro: Unicode.Scalar = "€"
print(e.value, String(e.value, radix: 16))  // "233 e9"
print(euro.value, String(euro.value, radix: 16))  // "8364 20ac"
```

## 3.2. Floating-Point Numbers

Swift's main floating-point types are `Float` (32 bits), `Double` (64 bits), and `Float16` (16 bits, mostly for graphics and machine learning); x86 platforms also have `Float80`. All follow the IEEE 754 standard that modern processors implement.

`Double` is the one to use unless you have a reason not to, and it's the type a floating-point literal gets when nothing says otherwise. It carries about 15 to 17 significant decimal digits and ranges up to about 1.8 × 10<sup>308</sup>; `Float` carries only about 7 digits and ranges up to about 3.4 × 10<sup>38</sup>. Seven digits run out quickly. `Float` can't even represent every integer above 2<sup>24</sup>:

```swift
let big: Float = 16_777_217  // 2^24 + 1
print(big)  // "16777216.0": the nearest Float is 2^24
```

Use `Float` where memory or bandwidth dominates, such as large arrays of samples or GPU data, and `Double` everywhere else. The static properties `greatestFiniteMagnitude`, `leastNonzeroMagnitude`, `ulpOfOne`, and others describe each type's limits.

Floating-point arithmetic is approximate, because most decimal fractions have no exact binary representation. A famous consequence:

```swift
var total = 0.0
for _ in 0..<10 {
    total += 0.1
}
print(total, total == 1.0)  // "0.9999999999999999 false"
```

Each `0.1` is actually the nearest binary fraction to 0.1, slightly off, and the errors accumulate. Never test computed floating-point values for exact equality; check whether they're within some tolerance, or, for money and other quantities that must be exact, use integers (a count of cents) or Foundation's `Decimal`.

Floating-point literals need digits before the decimal point (`.5` isn't a valid literal, since a leading dot means an implicit member like `.pi`), and may have an exponent, written with `e`:

```swift
let speedOfLight = 2.998e8  // meters per second
let electronMass = 9.109e-31  // kilograms
```

`print` shows a `Double` with the fewest digits that convert back to exactly the same value, which is why `print(0.1)` shows `0.1` even though the stored value isn't exactly a tenth. For reports, format explicitly. `String(format:)` supports `%f` (fixed notation), `%e` (exponent notation), and `%g` (whichever is more compact), each with optional width and precision. Here's a table of the growth of $1,000 at 5% annual interest:

```swift
import Foundation

print("year     balance")
for year in stride(from: 0, through: 30, by: 5) {
    let balance = 1000 * pow(1.05, Double(year))
    print(String(format: "%4d  %10.2f", year, balance))
}
```

```
year     balance
   0     1000.00
   5     1276.28
  10     1628.89
  15     2078.93
  20     2653.30
  25     3386.35
  30     4321.94
```

The `%10.2f` directive right-aligns each number in a 10-character field with two digits after the decimal point. Foundation also has a richer, locale-aware formatting API, as in `balance.formatted(.currency(code: "USD"))`, which is what user-facing applications should use.

Mathematical functions such as `pow`, `sin`, `exp`, and `log` come from the platform's C library, which `Foundation` makes available (as do `Darwin`, `Glibc`, and `Musl` directly). The standard library itself provides `squareRoot()`, `rounded(_:)`, `magnitude`, `isFinite`, and the basic arithmetic, and the `swift-numerics` package's `RealModule` offers generic, portable versions of all the elementary functions.

IEEE floating point has special values for results that ordinary numbers can't express. Dividing a nonzero number by zero gives positive or negative *infinity*, and operations with no meaningful answer, such as zero divided by zero or the square root of a negative number, give *NaN*, "not a number." Unlike integer division by zero, none of this traps:

```swift
let zero = 0.0
print(1 / zero, -1 / zero)  // "inf -inf"
print((zero / zero).isNaN, (-1.0).squareRoot().isNaN)  // "true true"
```

NaN has a property that catches people out: it's not equal to anything, including itself. Every comparison involving NaN is `false`, except `!=`, which is `true`:

```swift
let nan = Double.nan
print(nan == nan, nan < 1, nan > 1)  // "false false false"
```

So never use NaN as a marker for "no value" and then test for it with `==`. Test with `isNaN`, or better, use an optional, which says "maybe no value" in the type:

```swift
/// Returns the average of the values, or nil if there are none.
func average(_ values: [Double]) -> Double? {
    guard !values.isEmpty else { return nil }
    return values.reduce(0, +) / Double(values.count)
}
```

### 3.2.1. Example: A Heat Map

Our next program puts floating-point arithmetic to work drawing a picture. It renders a function of two variables, *z* = *f*(*x*, *y*), as a *heat map*: a grid of squares, each colored according to the function's value at its center, from blue for the lowest values through white to red for the highest. The output is SVG, which any web browser can display.

```swift
// swiftpl/ch3/heatmap
// Heatmap renders a function of two variables as an SVG grid of colored cells.
import Foundation

let cells = 60  // number of cells along each axis
let cellSize = 8.0  // size of each cell in pixels
let range = 6.0  // x and y run from -range to +range

let side = Int(Double(cells) * cellSize)
print("<svg xmlns='http://www.w3.org/2000/svg' width='\(side)' height='\(side)'>")
for row in 0..<cells {
    for col in 0..<cells {
        // Map the center of the cell to a point (x, y), with y increasing upward.
        let x = range * (2 * (Double(col) + 0.5) / Double(cells) - 1)
        let y = range * (1 - 2 * (Double(row) + 0.5) / Double(cells))
        let z = f(x, y)
        print("<rect x='\(Double(col) * cellSize)' y='\(Double(row) * cellSize)' "
            + "width='\(cellSize)' height='\(cellSize)' fill='\(color(z))'/>")
    }
}
print("</svg>")

/// The function to plot. Its values should lie roughly in -1...1.
func f(_ x: Double, _ y: Double) -> Double {
    sin(x) * cos(y)
}

/// Maps v in -1...1 to a color: blue for -1, white for 0, red for +1.
func color(_ v: Double) -> String {
    let t = max(-1, min(1, v))  // clamp to the expected range
    let r, g, b: Int
    if t < 0 {
        r = Int(255 * (1 + t))
        g = r
        b = 255
    } else {
        r = 255
        g = Int(255 * (1 - t))
        b = g
    }
    return String(format: "#%02x%02x%02x", r, g, b)
}
```

```
$ swift run heatmap > waves.svg
```

The program juggles two coordinate systems. The grid has integer *rows* and *columns*, counted from the top left as SVG expects. The function, though, is defined on a continuous plane of `Double` coordinates centered on the origin, with *y* increasing upward. The two lines that compute `x` and `y` convert from one to the other: `(Double(col) + 0.5) / Double(cells)` is the cell's center as a fraction of the width, from 0 to 1; multiplying by 2 and subtracting 1 shifts it to the range −1 to 1; multiplying by `range` scales it to the plane. The *y* formula subtracts from 1 instead, flipping the vertical axis.

`color` turns a value into a color in three steps. It clamps the value into −1...1 with `min` and `max`, so that a function that strays outside the expected range produces saturated colors rather than nonsense. It then fades from blue to white over the negative half and from white to red over the positive half. Finally it formats the three components as a CSS hex color like `#ff8080`. Note the declaration `let r, g, b: Int` without initial values: Swift's definite-initialization analysis (Section 2.3) confirms that both branches of the `if` assign all three before they're used.

With `sin(x) * cos(y)`, the picture is a checkerboard of soft red and blue blobs, a pattern often called an *egg crate*. Try other functions: `sin(hypot(x, y))` gives concentric ripples, and `(x * y) / (range * range)` a saddle.

**Exercise 3.1:** Some functions produce infinities or NaNs at some points (try `1 / (x * y)`). `color` clamps infinities, but `max` and `min` give unreliable results for NaN. Make the program paint any non-finite value gray, using `isFinite`.

**Exercise 3.2:** Instead of assuming the function's values lie in −1...1, compute all the values first, find the actual minimum and maximum, and scale the colors to fit.

**Exercise 3.3:** Replace the blue-white-red scale with a perceptually uniform one, such as *viridis*, by interpolating between a small table of reference colors.

**Exercise 3.4:** Serve the heat map from a web server (Section 1.7), with query parameters for the grid size and the plotted range. Remember to set the `Content-Type` header to `image/svg+xml`.

## 3.3. Complex Numbers

The standard library has no complex number type, but the Swift project publishes one in the `swift-numerics` package: `Complex<RealType>`, from the `ComplexModule` product, almost always used as `Complex<Double>`. Add `https://github.com/apple/swift-numerics` from version `1.0.0` to your package's dependencies, and `ComplexModule` (or `Numerics`, which includes everything) to your target's.

A complex number has a real part and an imaginary part:

```swift
import ComplexModule

let a = Complex(3.0, 4.0)  // 3 + 4i
let b = Complex(1.0, -2.0)  // 1 - 2i
print(a + b, a * b)  // "(4.0, 2.0) (11.0, -2.0)"
print(a.real, a.imaginary)  // "3.0 4.0"
print(a.length)  // "5.0", the magnitude |a|
```

The usual arithmetic operators work, as do `==` and `!=`, though complex numbers have no ordering. Besides `length`, the magnitude, a complex number has `lengthSquared`, which avoids a square root, and `phase`, its angle from the positive real axis. A complex number can also be built from a magnitude and phase with `Complex(length:phase:)`, and elementary functions are available as static methods such as `Complex.exp(_:)` and `Complex.sqrt(_:)`:

```swift
print(Complex.sqrt(Complex(-4.0, 0)))  // "(0.0, 2.0)": the square root of -4 is 2i
```

### 3.3.1. Example: Finding Frequencies

Complex numbers are at their most useful in signal processing, where they describe oscillations: a complex number of length 1 and phase θ is a point on the unit circle, and stepping θ forward traces a rotation. The *discrete Fourier transform* (DFT) uses such rotations to answer a question that's hard to answer by looking at a signal: what frequencies does it contain?

The program below builds one second of a signal sampled 64 times, made of two sine waves, a strong one at 3 cycles per second and a weaker one, half as loud, at 10. It then computes the DFT and reports which frequencies are present:

```swift
// swiftpl/ch3/frequencies
// Frequencies finds the component frequencies of a signal with a DFT.
import ComplexModule
import Foundation

let n = 64  // samples per second
let samples = (0..<n).map { i in
    let t = Double(i) / Double(n)  // time in seconds
    return sin(2 * .pi * 3 * t) + 0.5 * sin(2 * .pi * 10 * t)
}

let spectrum = dft(samples)
for (k, x) in spectrum.prefix(n / 2).enumerated() where x.length > 1 {
    print(String(format: "%2d Hz: magnitude %.1f", k, x.length))
}

/// Returns the discrete Fourier transform of the samples.
func dft(_ samples: [Double]) -> [Complex<Double>] {
    let n = samples.count
    return (0..<n).map { k in
        var sum = Complex<Double>.zero
        for (t, x) in samples.enumerated() {
            let angle = -2 * Double.pi * Double(k * t) / Double(n)
            sum += Complex(x) * Complex(length: 1, phase: angle)
        }
        return sum
    }
}
```

```
$ swift run frequencies
 3 Hz: magnitude 32.0
10 Hz: magnitude 16.0
```

For each candidate frequency *k*, the transform multiplies every sample by a point rotating backward around the unit circle *k* times over the second, and adds up the results. If the signal contains a component at that frequency, the component and the rotation stay in step and their products accumulate; if not, they drift in and out of phase and largely cancel. The length of each sum measures how much of that frequency is present: here, 32 for the 3 Hz wave and 16 for the 10 Hz wave of half the amplitude, with every other bin essentially zero. (For a real-valued signal, the second half of the spectrum mirrors the first, so we look only at the first `n / 2` bins.) The phase of each sum, which we ignore here, would tell us where in its cycle each component starts.

`Complex(x)` turns a real sample into a complex number with zero imaginary part, and `Complex(length: 1, phase: angle)` is the point on the unit circle at that angle, which is *e*<sup>iθ</sup>, or cos θ + *i* sin θ. Without a complex type, we'd have to carry out the multiplication by hand, as separate real and imaginary sums of sines and cosines. The complex arithmetic expresses the idea directly.

This direct transform takes time proportional to *n*², which is fine for 64 samples and hopeless for a million. The *fast Fourier transform* computes the same result in time proportional to *n* log *n* and is one of the most important algorithms in computing.

**Exercise 3.5:** Implement the recursive radix-2 fast Fourier transform for inputs whose length is a power of two, and check that it agrees with `dft` to within a small tolerance.

**Exercise 3.6:** Add random noise to the signal with `Double.random(in:)` and observe how the spectrum changes. How loud can the noise get before the 10 Hz component is hard to pick out?

**Exercise 3.7:** Write a function that returns both roots of a quadratic equation *ax*<sup>2</sup> + *bx* + *c* = 0 as `Complex<Double>` values, so that it works even when the discriminant is negative.

**Exercise 3.8:** Run `dft` with `Complex<Float>` instead of `Complex<Double>`, by making it generic over the real type. How large do the errors in the "empty" bins become as `n` grows?

## 3.4. Booleans

A `Bool` is either `true` or `false`. Conditions in `if`, `while`, and `guard` must be `Bool`s, and comparisons produce them. The prefix operator `!` negates a Boolean, and a `var` Boolean can be flipped in place with `toggle()`. Style tip: compare Booleans to nothing. Write `if isEnabled`, not `if isEnabled == true`.

The logical operators `&&` and `||` *short-circuit*: if the left operand decides the answer, the right operand isn't evaluated at all. That makes it safe to guard an operation with a test that must pass first:

```swift
if index < items.count && items[index].isValid {
    // items[index] is accessed only when index is in range
}
```

`&&` has higher precedence than `||`, just as multiplication has higher precedence than addition, so `a || b && c` means `a || (b && c)`. When the two are mixed, parentheses make the intent clearer.

In an `if`, `while`, or `guard` condition, a comma also means "and," and lets Boolean tests be mixed with optional bindings, each clause able to use names bound before it:

```swift
if let user = currentUser, user.isAdmin, !user.isSuspended {
    showAdminPanel(for: user)
}
```

There's no implicit conversion between `Bool` and numbers. If you need a 0 or 1, ask for it with `b ? 1 : 0`. In practice the need rarely arises, because what you usually want is a count of how many things are true, and collections can answer that directly:

```swift
let checks = [true, false, true, true]
print(checks.filter { $0 }.count)  // "3"
print(checks.allSatisfy { $0 }, checks.contains(false))  // "false true"
```

## 3.5. Strings

Swift's `String` type represents text as a sequence of *characters*, where a character means what a reader would call a character. That sounds like a definition no one could disagree with, but most languages don't actually follow it. In C, a string is an array of bytes. In Java and JavaScript, it's an array of 16-bit UTF-16 code units. In Go, it's a sequence of bytes that's usually UTF-8. In all of them, a single visible character may occupy several elements, and naïve indexing can cut one in half. Swift's `Character` type represents a whole *extended grapheme cluster*, the Unicode standard's definition of a user-perceived character, however many code points that takes.

Consider a string with a flag emoji, which Unicode encodes as two code points, and an "é" written as a plain "e" followed by a combining accent:

```swift
let s = "🇨🇦 cafe\u{301}"
print(s)  // "🇨🇦 café"
print(s.count)  // "6": 🇨🇦, space, c, a, f, é
print(s.unicodeScalars.count)  // "8"
print(s.utf16.count)  // "10"
print(s.utf8.count)  // "15"
```

`count` counts characters. The string's *views*, `unicodeScalars`, `utf16`, and `utf8`, present the same text as code points, as UTF-16 code units, and as UTF-8 bytes, and their counts differ accordingly. Each view is a collection that can be iterated and searched, and so is the string itself, element by element:

```swift
for c in "naïve" {  // c is a Character
    print(c, terminator: " ")
}
// "n a ï v e "
```

### 3.5.1. Indices and Substrings

Since characters vary in size, finding the *n*th one means scanning from the start. Swift won't disguise that cost as a cheap-looking `s[n]`, so strings can't be subscripted with integers. Positions are instead values of type `String.Index`, obtained from the string:

```swift
let phrase = "Swift on Linux"
print(phrase[phrase.startIndex])  // "S"
let i = phrase.index(phrase.startIndex, offsetBy: 6)
print(phrase[i])  // "o"
print(phrase[i...])  // "on Linux"
```

This is clumsy at first, but most string code doesn't need positions at all, because higher-level operations cover the common needs: `prefix(_:)`, `suffix(_:)`, `dropFirst(_:)`, `hasPrefix(_:)`, `contains(_:)`, `firstIndex(of:)`, `split(separator:)`, `replacing(_:with:)`, and so on:

```swift
print(phrase.prefix(5))  // "Swift"
print(phrase.hasSuffix("Linux"))  // "true"
print(phrase.split(separator: " "))  // "["Swift", "on", "Linux"]"
```

Slicing a string, whether with a range of indices or with methods like `prefix`, gives a `Substring`. A substring shares the original string's storage, so creating one is cheap, but it also keeps the whole original alive. Substrings are for short-lived use while processing text; convert to `String` before storing one: `let name = String(phrase.prefix(5))`.

String comparison follows Unicode *canonical equivalence*: two strings are equal if they contain the same characters, however those characters are encoded, so the precomposed "é" (U+00E9) and "e" followed by U+0301 compare equal. The ordering used by `<` compares Unicode scalar values, which is consistent and fast, good for sorted keys and binary search, but it isn't the alphabetical order a person expects. For text shown to users, sort with Foundation's `localizedStandardCompare(_:)`.

Strings are values. A `var` string can be changed in place with `append`, `+=`, `insert`, and `remove`, and changing one string never changes another, though, thanks to copy-on-write (Section 4.2), copies share storage until one of them is modified.

### 3.5.2. String Literals

A string literal is text in double quotes, with backslash escapes for special characters:

```
\0      null character
\\      backslash
\t      horizontal tab
\n      newline
\r      carriage return
\"      double quote
\'      single quote
\u{n}   Unicode scalar with hexadecimal code point n (1 to 8 digits)
\(x)    interpolation of the expression x
```

So `"\u{1F600}"` is a grinning face, and `"caf\u{E9}"` is "café."

A *raw string*, delimited by `#"` and `"#`, treats backslashes literally; escapes and interpolations in it must include the same number of `#` signs (`\#n`, `\#(x)`). Raw strings are handy for regular expressions and Windows paths:

```swift
let version = #"\d+\.\d+\.\d+"#  // the regex \d+\.\d+\.\d+
let path = #"C:\Program Files\Swift"#
```

A *multiline* literal is delimited by `"""` on lines of its own. Its line breaks are part of the string, and the closing delimiter's indentation is stripped from each line, so the literal can be indented along with the surrounding code (Section 1.4).

### 3.5.3. Unicode

Text in computers began with ASCII, which encodes 128 characters (English letters, digits, punctuation, and control codes) in seven bits. It was enough for the American engineers who designed it and inadequate for everyone else. For decades, each language community devised its own incompatible encodings, and text frequently arrived as gibberish when it crossed a border.

*Unicode* (`unicode.org`) replaced that chaos with a single numbering of every character in every writing system: letters and ideographs, accents, symbols, emoji, and much else, well over a hundred thousand in all. Each is assigned a number called a *code point*, or in Swift, a *Unicode scalar*, represented by the type `Unicode.Scalar`.

Code points aren't the end of the story, because a single visible character can be built from several of them: a letter followed by combining accents, a pair of "regional indicator" scalars forming a national flag, a family emoji assembled from several person emoji joined by invisible joiners, or a Korean syllable composed from its letters. Unicode defines how code points group into user-perceived characters, called *extended grapheme clusters*, and a Swift `Character` is one such cluster. That's why `"🇨🇦".count` is 1 even though it contains two scalars.

`Character` offers properties drawn from the Unicode database, such as `isLetter`, `isNumber`, `isWhitespace`, `isUppercase`, `isASCII`, and `asciiValue`, so that code can classify text correctly in any script without consulting tables of code points.

### 3.5.4. UTF-8

A string has to be stored as bytes somehow, and Swift uses UTF-8. UTF-8 encodes each code point in one to four bytes; the high bits of the first byte say how many bytes the sequence uses, and each continuation byte begins with the bits `10`:

```
0xxxxxxx                              code points 0–127          (ASCII)
110xxxxx 10xxxxxx                     128–2047
1110xxxx 10xxxxxx 10xxxxxx            2048–65535
11110xxx 10xxxxxx 10xxxxxx 10xxxxxx   65536–0x10ffff
```

The design has many virtues. ASCII text is unchanged, byte for byte, so UTF-8 is compatible with decades of existing software. Common text is compact. A decoder can find where a character starts from any position by skipping continuation bytes, and no character's encoding appears inside another's, so byte-level searching works. Sorting by bytes sorts by code point. These properties have made UTF-8 the dominant encoding of the web and of files everywhere.

A native Swift string stores its UTF-8 bytes directly, so the `utf8` view involves no conversion and iterating over it is as fast as iterating over an array of bytes. For data that's mostly ASCII, such as CSV files, log lines, protocol headers, and JSON, processing the `utf8` view instead of characters can be several times faster, since it skips grapheme-cluster analysis.

A short program shows where the multi-byte characters in a string begin:

```swift
let s = "Hello, 世界"
print(s.count, s.utf8.count)  // "9 13"
for (offset, byte) in s.utf8.enumerated() where byte >= 0xC0 {
    print("a multi-byte character starts at byte \(offset)")
}
// a multi-byte character starts at byte 7
// a multi-byte character starts at byte 10
```

Bytes at or above `0xC0` begin a multi-byte sequence; continuation bytes lie between `0x80` and `0xBF`.

To make a string from bytes, `String(decoding:as:)` decodes UTF-8 and replaces any malformed sequence with the replacement character U+FFFD (usually drawn as a question mark in a diamond), while `String(validating:as:)` returns `nil` on malformed input:

```swift
let bytes: [UInt8] = [0x4F, 0x4B, 0xFF]
print(String(decoding: bytes, as: UTF8.self))  // "OK\u{FFFD}"
print(String(validating: bytes, as: UTF8.self) as Any)  // "nil"
```

Working at the wrong level of the string corrupts text. Truncating a string to fit a display, for instance, must count characters, not bytes; cutting the UTF-8 bytes of "été" after four bytes splits the second "é" in half:

```swift
let word = "été"
print(String(decoding: word.utf8.prefix(4), as: UTF8.self))  // "ét\u{FFFD}": broken
print(word.prefix(2))  // "ét": correct
```

### 3.5.5. Processing Text

Most string processing uses methods that `String` gets from the collection protocols it conforms to, which give strings many of the same algorithms as arrays (`map`, `filter`, `reversed`, `contains`, `first(where:)`, `split`, `firstRange(of:)`, `replacing(_:with:)`), plus Foundation's case-insensitive and locale-aware operations and the standard library's `Regex` type, written `/.../` (Swift 5.7).

Here's a small example of character-by-character processing: *run-length encoding*, which compresses runs of a repeated character into a count and the character, so that `"aaabccdddd"` becomes `"3a1b2c4d"`:

```swift
// swiftpl/ch3/rle
/// Returns the run-length encoding of s: each run of a repeated
/// character becomes its length followed by the character.
func runLengthEncoded(_ s: String) -> String {
    var out = ""
    var rest = Substring(s)
    while let c = rest.first {
        let run = rest.prefix(while: { $0 == c })
        out += "\(run.count)\(c)"
        rest = rest.dropFirst(run.count)
    }
    return out
}

print(runLengthEncoded("aaabccdddd"))  // "3a1b2c4d"
print(runLengthEncoded("🇨🇦🇨🇦🇨🇦"))  // "3🇨🇦"
```

The loop consumes the input from the front. `rest.first` is the character beginning the current run; `prefix(while:)` measures how far the run extends; and `dropFirst` advances past it. Because `rest` is a `Substring`, each step creates only a view, not a copy. Because all the operations work in `Character`s, a run of flags is counted as a run of flags, three of them, rather than as six unrelated code points.

When building a string piece by piece, append to a `var` string. Appending is amortized constant time, because strings grow their storage geometrically, as arrays do. As an example, here's a function that writes bytes in hexadecimal, eight to a line, in the style of a hex dump:

```swift
/// Formats bytes as two-digit hexadecimal numbers, eight per line.
func hexDump(_ bytes: some Sequence<UInt8>) -> String {
    var out = ""
    for (i, b) in bytes.enumerated() {
        if i > 0 {
            out += i % 8 == 0 ? "\n" : " "
        }
        if b < 0x10 {
            out += "0"
        }
        out += String(b, radix: 16)
    }
    return out
}

print(hexDump("Hello, 世界".utf8))
// 48 65 6c 6c 6f 2c 20 e4
// b8 96 e7 95 8c
```

The output makes the encoding visible: seven ASCII bytes, then three bytes each for 世 (`e4 b8 96`) and 界 (`e7 95 8c`).

**Exercise 3.9:** Write `runLengthDecoded(_:)`, the inverse of `runLengthEncoded`, and check that decoding an encoding gives back the original. What should the decoder do with malformed input such as `"3"` or `"a"`? What about input whose characters are themselves digits?

**Exercise 3.10:** Write a function `truncated(_ s: String, to n: Int) -> String` that shortens a string to at most `n` characters, ending with "…" when anything was cut, and preferring to cut at a space if there's one in the last few characters.

**Exercise 3.11:** Extend `hexDump` to print an offset at the start of each line and, at the end, the printable ASCII characters of the line's bytes, in the style of the `xxd` tool.

### 3.5.6. Conversions between Strings and Numbers

To convert a number to its decimal text, use string interpolation or `String(_:)`; to choose a base, use `String(_:radix:)`:

```swift
let n = 2025
print("\(n)", String(n), String(n, radix: 16))  // "2025 2025 7e9"
```

Parsing goes the other way. Every numeric type has an initializer that takes a string and returns an optional, `nil` if the text isn't a valid number or is out of range for the type. Integer types also accept a `radix:`:

```swift
Int("42")  // Optional(42)
Int(" 42")  // nil: no surrounding whitespace allowed
Int("ff", radix: 16)  // Optional(255)
UInt8("300")  // nil: too big for a UInt8
Double("2.5e3")  // Optional(2500.0)
```

The parsers are strict: a leading sign is allowed, but nothing else besides digits, so text from users or files usually needs trimming first. Because the results are optional, parsing goes naturally with `guard let`, `if let`, and `??`:

```swift
guard let port = Int(portText), (1...65535).contains(port) else {
    fatalError("invalid port: \(portText)")
}
let retries = Int(retriesText) ?? 3
```

## 3.6. Literals and Constants

A *literal* is a value written directly in source code: `42`, `3.14`, `"hello"`, `true`, `[1, 2, 3]`, `nil`. Swift's literals are more flexible than they first appear, because a literal has no fixed type of its own.

### 3.6.1. Literal Types

The type of a literal comes from its context:

```swift
let a = 42  // Int, the default for integer literals
let b: Double = 42  // the literal becomes a Double
let c: UInt8 = 42  // the literal becomes a UInt8
let d = 42 + 0.5  // Double: the context requires it
let e: Float = 1 / 3  // Float division, 0.33333334, not integer division
```

Each kind of literal corresponds to a protocol. A type conforming to `ExpressibleByIntegerLiteral` can be created from an integer literal, and similarly for floating-point, string, Boolean, array, dictionary, and `nil` literals. When the compiler meets a literal, it picks a type from the surrounding expression, falling back on a default (`Int`, `Double`, `String`, `Bool`, `Array`, or `Dictionary`) only when nothing else decides it. So a literal fits wherever any suitable type is expected, without conversions, and a literal that can't fit is caught at compile time:

```swift
let small: Int8 = 200  // compile error: integer literal '200' overflows when stored into 'Int8'
```

Once a literal has become a value, it's an ordinary value of its type, and ordinary rules apply. In particular, arithmetic on literals happens in the type of the result, not with some unlimited precision:

```swift
let x = 1 << 70  // 0: an Int smart-shifted past its width
let y = 9_223_372_036_854_775_807 + 1  // compile error: arithmetic operation '9223372036854775807 + 1' (on type 'Int') results in an overflow
```

User-defined types can opt in to literal syntax. We gave `Kilometers` integer and floating-point literals in Section 2.5, and many standard types use the protocols too: an array literal can create a `Set`, and a string literal can create a `Character` or a `Unicode.Scalar`, as we did in Section 3.1.

Named constants are simply `let` declarations. The optimizer folds simple constant expressions into the code that uses them, so naming a value costs nothing at run time:

```swift
let maxConnections = 512
let requestTimeout = 30.0  // seconds

enum Physics {
    static let speedOfLight = 299_792_458.0  // meters per second
    static let standardGravity = 9.806_65  // meters per second squared
}
```

An `enum` with no cases, like `Physics`, can't be instantiated, which makes it a convenient *namespace* for grouping related constants.

### 3.6.2. Enumerations

When a value can be one of a fixed set of alternatives, model it with an enumeration:

```swift
enum Suit {
    case clubs, diamonds, hearts, spades
}

var trump = Suit.hearts
trump = .spades  // the type is known, so the leading dot is enough
```

An enum is a real type. A `Suit` can't be confused with an integer or a string, and a `switch` over one must handle every case (or provide a `default`), so adding a case later makes the compiler point out every switch that needs updating.

An enum can have *raw values* of an integer, floating-point, or string type, so that each case corresponds to a fixed value, which is useful when cases must be stored in a file or sent over a network. Integer raw values count up automatically from the last one given, starting at 0; string raw values default to the case names:

```swift
enum Priority: Int, CaseIterable {
    case low = 1, normal, high, urgent
}

print(Priority.urgent.rawValue)  // "4"
print(Priority(rawValue: 2)!)  // "normal"
print(Priority(rawValue: 9) as Any)  // "nil"
print(Priority.allCases.map(\.rawValue))  // "[1, 2, 3, 4]"
```

Creating a case from a raw value can fail, since not every number is a priority, so `init(rawValue:)` returns an optional. Conforming to `CaseIterable` makes the compiler provide `allCases`, every case in declaration order.

Enumerations can do much more: cases can carry associated values of their own, and enums can have methods and conform to protocols. Section 4.7 covers these *algebraic data types* in depth, and Section 4.9 the pattern matching that takes them apart.

### 3.6.3. Option Sets

Sometimes the alternatives aren't exclusive: text can be bold *and* italic. Rather than a single case, such a value is a set of independent flags, traditionally packed into the bits of an integer, as with the file permissions of Section 3.1. Swift's `OptionSet` protocol packages that technique:

```swift
struct TextStyle: OptionSet {
    let rawValue: UInt8

    static let bold = TextStyle(rawValue: 1 << 0)
    static let italic = TextStyle(rawValue: 1 << 1)
    static let underline = TextStyle(rawValue: 1 << 2)
    static let strikethrough = TextStyle(rawValue: 1 << 3)

    static let emphasis: TextStyle = [.bold, .italic]  // a combination
}
```

The struct stores the bits as its `rawValue` and names each flag as a static constant with one bit set. From that, `OptionSet` provides the operations of a set (`contains`, `insert`, `remove`, `union`, `intersection`, `symmetricDifference`, and the rest) and lets values be written as array literals:

```swift
var style: TextStyle = [.bold, .underline]
print(style.rawValue)  // "5"
print(style.contains(.bold))  // "true"
style.remove(.bold)
style.formUnion(.emphasis)
print(style.contains([.italic, .underline]))  // "true"
print(style.isSuperset(of: .emphasis))  // "true"
```

There's no need to write `isBold` or `setItalic` helpers; the set operations already say everything, and they compile to the same bit manipulation you would write by hand.

**Exercise 3.12:** Add raw values of type `String` to `Suit`, and use `CaseIterable` to print a deck of 52 cards, such as `"A♠"`.

**Exercise 3.13:** Give `TextStyle` a `description` that lists the names of the styles it contains, like `[bold, italic]`. Can you avoid writing each name twice? (Hint: consider a static array of `(TextStyle, String)` pairs.)
