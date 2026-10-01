# 3. Basic Data Types

It's all bits at the bottom, of course, but computers operate fundamentally on fixed-size numbers called *words*, which are interpreted as integers, floating-point numbers, bit sets, or memory addresses, then combined into larger aggregates that represent packets, pixels, portfolios, poetry, and everything else. Swift offers a variety of ways to organize data, with a spectrum of data types that at one end match the features of the hardware and at the other end provide what programmers need to conveniently represent complicated data structures.

Unlike C or Go, Swift has no "built-in" types in the usual sense. `Int`, `Double`, `Bool`, and `String` are ordinary structs defined in the standard library, written in Swift, with the same kinds of declarations you can write yourself. The compiler knows how to map their innards onto machine instructions, and the optimizer makes them exactly as efficient as true primitive types, but in every other respect they're just library code, and you can extend them with new methods and protocol conformances like any other type.

Swift's types fall into a few broad families. This chapter covers the *basic* types, which include numbers, booleans, and strings. Chapter 4 covers *composite* types: arrays, dictionaries, sets, structs, and tuples. Classes and actors, the *reference* types, appear in Chapters 6 and 9, and *protocol types*, also called existentials, in Chapter 7.

## 3.1. Integers

Swift's numeric types include several sizes of integers, floating-point numbers, and SIMD vectors. Each numeric type determines the size and signedness of its values. Let's begin with integers.

Swift provides both signed and unsigned integer arithmetic. There are four distinct sizes of signed integers (8, 16, 32, and 64 bits), represented by the types `Int8`, `Int16`, `Int32`, and `Int64`, and corresponding unsigned versions `UInt8`, `UInt16`, `UInt32`, and `UInt64`. Swift 6 also added `Int128` and `UInt128`.

There are also two types called just `Int` and `UInt` that are the natural or most efficient size for signed and unsigned integers on a particular platform: 64 bits on all current desktop, server, and phone platforms, and 32 bits on some embedded targets.

`Int` is by far the most widely used numeric type, and the API Design Guidelines recommend using it for all integer quantities unless you have a specific reason to choose otherwise. Array counts and indexes are `Int`, as are string lengths. Prefer `Int` even for values that can never be negative, such as counts, because mixing signed and unsigned values requires explicit conversions everywhere. The unsigned types are for bit patterns, for binary formats, and for interoperating with C.

`Int` is *not* the same type as `Int64`, even on 64-bit platforms where they have the same representation. An explicit conversion is required to use a value of one where the other is needed.

Signed numbers are represented in two's-complement form, in which the high-order bit is reserved for the sign of the number and the range of values of an *n*-bit number is from −2<sup>n−1</sup> to 2<sup>n−1</sup>−1. Unsigned integers use the full range of bits for non-negative values and thus have the range 0 to 2<sup>n</sup>−1. For instance, the range of `Int8` is −128 to 127, whereas the range of `UInt8` is 0 to 255. Every integer type has static properties `min` and `max` that give its range:

```swift
print(Int8.min, Int8.max)  // "-128 127"
print(UInt64.max)  // "18446744073709551615"
```

Swift's binary operators for arithmetic, logic, and comparison are listed here in order of decreasing precedence:

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

These are fewer levels than C has, and they're slightly different: in particular, the shifts bind *more* tightly than multiplication, and the bitwise `&` binds as multiplication while `|` and `^` bind as addition. Parentheses may always be used to make the meaning clear or to override the precedence.

The arithmetic operators `+`, `-`, `*`, and `/` may be applied to integer and floating-point numbers. The remainder operator `%` applies only to integers. The sign of the remainder is always the same as the sign of the dividend, so `-5 % 3` and `-5 % -3` are both `-2`. The behavior of `/` depends on whether its operands are integers, so `5.0 / 4.0` is `1.25`, but `5 / 4` is `1`, since integer division truncates the result toward zero.

### 3.1.1. Overflow

Here Swift differs from both C and Go. If the result of an arithmetic operation has more bits than can be represented in the result type, it is said to *overflow*. In C, signed overflow is undefined behavior; in Go, high-order bits are silently discarded. In Swift, overflow is a *run-time error*: the program *traps*, stopping immediately with a message.

```swift
var u: UInt8 = 255
u += 1  // Fatal error: Arithmetic overflow

var i: Int8 = 127
i += 1  // Fatal error: Arithmetic overflow
```

If the overflow is visible to the compiler, as in `let x: Int8 = 127 + 1`, it's a compile-time error instead.

Trapping is a deliberate safety choice. A silent overflow can turn a bounds check into a security hole, and it's almost never what the programmer intended. The checks cost very little, since processors set an overflow flag as a side effect of arithmetic, and the optimizer removes checks it can prove unnecessary.

When you *do* want wraparound arithmetic, for hash functions, checksums, random number generators, and the like, use the *overflow operators* `&+`, `&-`, and `&*`, which discard the high-order bits:

```swift
var u: UInt8 = 255
print(u &+ 1, u &* u)  // "0 1"

var i: Int8 = 127
print(i &+ 1)  // "-128"
```

The integer types also have methods that report overflow rather than trapping, for algorithms that need to detect it:

```swift
let (result, overflow) = Int.max.addingReportingOverflow(1)
print(result, overflow)  // "-9223372036854775808 true"
```

### 3.1.2. Comparison and Bitwise Operators

Two integers of the same type may be compared using the binary comparison operators below; the type of a comparison expression is `Bool`.

```
==    equal to
!=    not equal to
<     less than
<=    less than or equal to
>     greater than
>=    greater than or equal to
```

In fact, all values of basic type (booleans, numbers, and strings) are *comparable*, meaning that two values of the same type may be compared using the `==` and `!=` operators. Furthermore, integers, floating-point numbers, and strings are *ordered* by the comparison operators. These properties are captured by the protocols `Equatable` and `Comparable`, which many other types conform to as well.

There are also unary addition and subtraction operators:

```
+    unary positive (no effect)
-    unary negation
```

Swift also provides the following bitwise binary operators, the first four of which treat their operands as bit patterns with no concept of arithmetic carry or sign:

```
&     bitwise AND
|     bitwise OR
^     bitwise XOR
~     bitwise NOT (unary prefix)
<<    left shift
>>    right shift
```

Note that `~` is bitwise complement in Swift, as in C, while Go uses `^` for both XOR and complement. Swift has no AND NOT operator; write `x & ~y`.

The code below shows how bitwise operations can be used to interpret a `UInt8` value as a compact and efficient set of 8 independent bits. It uses `String(_:radix:)` to print a number's binary digits, padding with zeros to 8 places:

```swift
func binary(_ x: UInt8) -> String {
    let digits = String(x, radix: 2)
    return String(repeating: "0", count: 8 - digits.count) + digits
}

let x: UInt8 = 1 << 1 | 1 << 5
let y: UInt8 = 1 << 1 | 1 << 2

print(binary(x))  // "00100010", the set {1, 5}
print(binary(y))  // "00000110", the set {1, 2}

print(binary(x & y))  // "00000010", the intersection {1}
print(binary(x | y))  // "00100110", the union {1, 2, 5}
print(binary(x ^ y))  // "00100100", the symmetric difference {2, 5}
print(binary(x & ~y))  // "00100000", the difference {5}

for i in 0..<8 where x & (1 << i) != 0 {  // membership test
    print(i)  // "1", "5"
}

print(binary(x << 1))  // "01000100", the set {2, 6}
print(binary(x >> 1))  // "00010001", the set {0, 4}
```

(Section 6.5 shows an implementation of integer sets that can be much bigger than a byte. For small sets with named members, the standard library's `OptionSet` protocol is the idiomatic tool; we'll see it in Section 3.6.)

In the shift operations `x << n` and `x >> n`, the `n` operand determines the number of bit positions to shift. Swift's shifts are *smart shifts*: shifting by a negative amount shifts the other way, and shifting by more than the width of the type produces 0 (or −1 for a right shift of a negative signed value), rather than being undefined. The shift amount may be any integer type, regardless of the type of `x`. For speed, the *masking shifts* `&<<` and `&>>` instead use only the low-order bits of the shift amount, as the hardware does.

Left shifts fill the vacated bits with zeros, as do right shifts of unsigned numbers, but right shifts of signed numbers fill the vacated bits with copies of the sign bit. For this reason, it's important to use unsigned arithmetic when you're treating an integer as a bit pattern.

Integer types also have useful bit-level properties: `nonzeroBitCount` (population count), `leadingZeroBitCount`, `trailingZeroBitCount`, `byteSwapped`, and `bitWidth`.

### 3.1.3. Conversions

Although Swift provides unsigned numbers and arithmetic, we tend to use the signed `Int` form even for quantities that can't be negative, such as the length of an array, though `UInt` might seem a more obvious choice. Consider this loop, which in a C-like language with unsigned lengths runs forever:

```c
for (unsigned i = len - 1; i >= 0; i--) { ... }  // C: i >= 0 is always true
```

With `Int` there's no such problem, and in Swift you would write `for i in (0..<len).reversed()` anyway, avoiding index arithmetic altogether.

In general, an explicit conversion is required to convert a value from one numeric type to another, and binary operators for arithmetic and logic (except shifts) must have operands of the same type. Although this occasionally results in longer expressions, it also eliminates a whole class of problems and makes programs easier to understand.

As an example familiar from other contexts, consider this sequence:

```swift
let apples: Int32 = 1
let oranges: Int16 = 2
let compote = apples + oranges  // compile error
```

Attempting to compile these three declarations produces an error message:

```
error: binary operator '+' cannot be applied to operands of type 'Int32' and 'Int16'
```

This type mismatch can be fixed in several ways, most directly by converting everything to a common type:

```swift
let compote = Int(apples) + Int(oranges)
```

Here Swift differs again from Go and C. Converting an integer to a narrower type with an initializer like `Int8(x)` doesn't silently truncate: if the value doesn't fit, the program traps. The integer types offer several initializers that let you say exactly what you want when the value might be out of range:

```swift
let big = 1000
Int8(big)  // traps: "Not enough bits to represent the passed value"
Int8(exactly: big)  // nil: returns an Optional
Int8(clamping: big)  // 127: saturates to the nearest representable value
Int8(truncatingIfNeeded: big)  // -24: keeps the low-order 8 bits, like C
```

Converting from a floating-point number to an integer truncates toward zero, and also traps if the value is out of range, infinite, or NaN:

```swift
let f = 3.141  // a Double
let i = Int(f)
print(f, i)  // "3.141 3"
let g = 1.99
print(Int(g))  // "1"
print(Int(g.rounded()))  // "2"
print(Int(exactly: g) as Any)  // "nil"
```

Converting the other way, `Double(i)`, may lose precision for integers above 2<sup>53</sup>, which is the one numeric conversion that doesn't check.

Integer literals of any size and type can be written as ordinary decimal numbers, or as binary numbers if they begin with `0b`, octal numbers if they begin with `0o`, or hexadecimal if they begin with `0x`. Hexadecimal digits may be upper or lower case. Underscores may be used anywhere in a numeric literal to group digits, as in `1_000_000` or `0xFFFF_FFFF`. Unlike in C, a leading zero does *not* make a literal octal: `0755` is decimal 755.

When printing numbers, `String(_:radix:uppercase:)` converts an integer to a string in any base from 2 to 36:

```swift
let o = 0o666
print(o, String(o, radix: 8))  // "438 666"
let x = 0xdeadbeef
print(x, String(x, radix: 16), String(x, radix: 16, uppercase: true))
// "3735928559 deadbeef DEADBEEF"
```

For more control over formatting, Foundation's `String(format:)` accepts C's `printf` verbs, including widths, padding, and `%x` and `%o`:

```swift
print(String(format: "%d %08x %#o", 438, 438, 438))  // "438 000001b6 0666"
```

Characters are not integers in Swift, as we'll see in Section 3.5, but every Unicode scalar has a numeric code point, available from its `value` property:

```swift
let ascii: Unicode.Scalar = "a"
let unicode: Unicode.Scalar = "国"
print(ascii, ascii.value)  // "a 97"
print(unicode, unicode.value)  // "国 22269"
print(String(unicode.value, radix: 16))  // "56fd"
```

## 3.2. Floating-Point Numbers

Swift provides three main sizes of floating-point numbers, `Float16`, `Float`, and `Double`, which are 16, 32, and 64 bits wide. (On x86 platforms there's also `Float80`, the extended-precision type supported by that hardware.) Their arithmetic properties are governed by the IEEE 754 standard implemented by all modern CPUs.

Values of these numeric types range from tiny to huge. The static property `greatestFiniteMagnitude` gives the largest value: about 3.4e38 for `Float` and about 1.8e308 for `Double`. The smallest positive values are near 1.4e-45 and 4.9e-324, respectively, and available as `leastNonzeroMagnitude`.

A `Float` provides approximately six decimal digits of precision, whereas a `Double` provides about 15 digits. `Double` should be preferred for most purposes, and it is the type that a floating-point literal gets when nothing else determines its type. `Float` computations accumulate error rapidly unless one is quite careful, and the smallest positive integer that cannot be exactly represented as a `Float` is not very large:

```swift
var f: Float = 16_777_216  // 1 << 24
print(f == f + 1)  // "true"!
```

Floating-point numbers can be written literally using decimals, like this:

```swift
let e = 2.71828  // (approximately)
```

Digits may be omitted after the decimal point, but not before it: `.707` is not a valid literal, because a leading dot in Swift means an implicit member expression like `.pi`. Very small or very large numbers are better written in scientific notation, with the letter `e` or `E` preceding the decimal exponent:

```swift
let avogadro = 6.02214129e23
let planck = 6.62606957e-34
```

Floating-point values print using the shortest decimal representation that round-trips to the same value, so `print(0.1)` prints `0.1`, while `print(0.1 + 0.2)` prints `0.30000000000000004`. For tables and other output where precision should be controlled, use `String(format:)` with `%g` (the most compact representation that has adequate precision), `%e` (exponent), or `%f` (no exponent), all of which accept a field width and numeric precision. Foundation also offers a more flexible formatting API for locale-aware output, `x.formatted(.number.precision(.fractionLength(3)))`.

```swift
import Foundation

for x in 0..<8 {
    print(String(format: "x = %d e^x = %8.3f", x, exp(Double(x))))
}
```

The code above prints the powers of *e* with three decimal digits of precision, aligned in an eight-character field:

```
x = 0 e^x =    1.000
x = 1 e^x =    2.718
x = 2 e^x =    7.389
x = 3 e^x =   20.086
x = 4 e^x =   54.598
x = 5 e^x =  148.413
x = 6 e^x =  403.429
x = 7 e^x = 1096.633
```

Elementary functions like `sin`, `exp`, and `sqrt` come from the platform's C math library, available through `Foundation` (or directly through `Darwin` on Apple platforms and `Glibc` or `Musl` on Linux). The standard library itself provides `squareRoot()`, `rounded(_:)`, `magnitude`, and the usual arithmetic, and the `swift-numerics` package's `RealModule` offers generic, cross-platform versions of all the elementary functions.

The floating-point types have static properties for the special values defined by IEEE 754: the positive and negative infinities, which represent numbers of excessive magnitude and the result of division by zero; and NaN ("not a number"), the result of such mathematically dubious operations as `0.0 / 0.0` or `(-1.0).squareRoot()`.

```swift
let z = 0.0
print(z, -z, 1 / z, -1 / z, z / z)  // "0.0 -0.0 inf -inf nan"
```

Floating-point division by zero does *not* trap, unlike integer division by zero. The property `isNaN` tests whether its argument is a not-a-number value, and `Double.nan` returns such a value. It's tempting to use NaN as a sentinel value in a numeric computation, but testing whether a specific computational result is equal to NaN is fraught with peril because any comparison with NaN *always* yields `false` (except `!=`, which is always the negation of `==`):

```swift
let nan = Double.nan
print(nan == nan, nan < nan, nan > nan)  // "false false false"
```

If a function that returns a floating-point result might fail, it's better to report the failure with an optional, like this:

```swift
func compute() -> Double? {
    // ...
    if failed {
        return nil
    }
    return result
}
```

The next program illustrates floating-point graphics computation. It plots a function of two variables `z = f(x, y)` as a wire mesh 3-D surface, using SVG, as we did in Section 1.4. It draws a three-dimensional surface by computing, for each cell of a 100 × 100 grid, the *z* value of the function at its four corners, projecting them isometrically onto the two-dimensional SVG canvas, and drawing the cell as a polygon.

```swift
// swiftpl/ch3/surface
// Surface computes an SVG rendering of a 3-D surface function.
import Foundation

let width = 600.0, height = 320.0  // canvas size in pixels
let cells = 100  // number of grid cells
let xyrange = 30.0  // axis ranges (-xyrange..+xyrange)
let xyscale = width / 2 / xyrange  // pixels per x or y unit
let zscale = height * 0.4  // pixels per z unit
let angle = Double.pi / 6  // angle of x, y axes (=30°)

let sin30 = sin(angle), cos30 = cos(angle)

print(
    """
    <svg xmlns='http://www.w3.org/2000/svg' style='stroke: grey; fill: white; \
    stroke-width: 0.7' width='\(Int(width))' height='\(Int(height))'>
    """)
for i in 0..<cells {
    for j in 0..<cells {
        let (ax, ay) = corner(i + 1, j)
        let (bx, by) = corner(i, j)
        let (cx, cy) = corner(i, j + 1)
        let (dx, dy) = corner(i + 1, j + 1)
        print(
            String(
                format: "<polygon points='%g,%g %g,%g %g,%g %g,%g'/>",
                ax, ay, bx, by, cx, cy, dx, dy))
    }
}
print("</svg>")

func corner(_ i: Int, _ j: Int) -> (Double, Double) {
    // Find point (x,y) at corner of cell (i,j).
    let x = xyrange * (Double(i) / Double(cells) - 0.5)
    let y = xyrange * (Double(j) / Double(cells) - 0.5)

    // Compute surface height z.
    let z = f(x, y)

    // Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
    let sx = width / 2 + (x - y) * cos30 * xyscale
    let sy = height / 2 + (x + y) * sin30 * xyscale - z * zscale
    return (sx, sy)
}

func f(_ x: Double, _ y: Double) -> Double {
    let r = hypot(x, y)  // distance from (0,0)
    return sin(r) / r
}
```

Notice that the function `corner` returns two values, the coordinates of the corner of the cell, as a tuple.

Explaining how the program works requires only basic geometry, but it's fine to skip over it, since the point is to illustrate floating-point computation. The essence of the program is mapping between three different coordinate systems. The first is a 2-D grid of 100 × 100 cells identified by integer coordinates (*i*, *j*), starting at (0, 0) in the far back corner. We plot from the back to the front so that background polygons may be obscured by foreground ones.

The second coordinate system is a mesh of 3-D floating-point coordinates (*x*, *y*, *z*), where *x* and *y* are linear functions of *i* and *j*, translated so that the origin is in the center, and scaled by the constant `xyrange`. The height *z* is the value of the surface function *f*(*x*, *y*).

The third coordinate system is the 2-D image canvas, with (0, 0) in the top left corner. Points in this plane are denoted (*sx*, *sy*). We use an isometric projection to map each 3-D point (*x*, *y*, *z*) onto the 2-D canvas. A point appears farther to the right on the canvas the greater its *x* value or the *smaller* its *y* value. And a point appears farther down the canvas the greater its *x* value or *y* value, and the smaller its *z* value. The vertical and horizontal scale factors for *x* and *y* are derived from the sine and cosine of a 30° angle. The scale factor for *z*, 0.4, is an arbitrary parameter.

For each cell in the 2-D grid, the main program computes the coordinates on the image canvas of the four corners of the polygon ABCD, where B corresponds to (*i*, *j*) and A, C, and D are its neighbors, then prints an SVG instruction to draw it.

**Exercise 3.1:** If the function `f` returns a non-finite `Double` value, the SVG file will contain invalid `<polygon>` elements (although many SVG renderers handle this gracefully). Modify the program to skip invalid polygons. (Hint: `isFinite`.)

**Exercise 3.2:** Experiment with visualizations of other functions. Can you produce an egg box, moguls, or a saddle?

**Exercise 3.3:** Color each polygon based on its height, so that the peaks are colored red (`#ff0000`) and the valleys blue (`#0000ff`).

**Exercise 3.4:** Following the approach of the Lissajous server in Section 1.7, construct a web server that computes surfaces and writes SVG data to the client. The server must set the `Content-Type` header to `image/svg+xml`.

## 3.3. Complex Numbers

Swift's standard library doesn't include complex numbers, but the Swift project maintains a package that does: `swift-numerics`, whose `ComplexModule` provides a generic `Complex<RealType>` struct, usually used as `Complex<Double>`. Add the package to your manifest as a dependency on `https://github.com/apple/swift-numerics` from version `1.0.0`, and the product `ComplexModule` (or `Numerics`, which includes everything) to your target.

A complex number is created from its real and imaginary components, which are accessible as properties:

```swift
import ComplexModule

let x = Complex(1.0, 2.0)  // 1+2i
let y = Complex(3.0, 4.0)  // 3+4i
print(x * y)  // "(-5.0, 10.0)"
print((x * y).real)  // "-5.0"
print((x * y).imaginary)  // "10.0"
```

All the usual arithmetic operators work, and complex numbers may be compared for equality with `==` and `!=`. They are not ordered. `ComplexModule` provides the magnitude of a complex number as `length` and its square, which is cheaper to compute, as `lengthSquared`; the phase angle is `phase`. Elementary functions like `exp` and `sqrt` are available as static members, `Complex.exp(z)` and `Complex.sqrt(z)`:

```swift
print(Complex.sqrt(Complex(-1.0, 0)))  // "(0.0, 1.0)", that is, i
```

The following program uses `Complex<Double>` arithmetic to generate a Mandelbrot set.

```swift
// swiftpl/ch3/mandelbrot
// Mandelbrot emits a PGM image of the Mandelbrot fractal.
import ComplexModule
import Foundation

let xmin = -2.0, ymin = -2.0, xmax = 2.0, ymax = 2.0
let width = 1024, height = 1024

var pixels = [UInt8](repeating: 0, count: width * height)
for py in 0..<height {
    let y = Double(py) / Double(height) * (ymax - ymin) + ymin
    for px in 0..<width {
        let x = Double(px) / Double(width) * (xmax - xmin) + xmin
        // Image point (px, py) represents complex value z.
        pixels[py * width + px] = mandelbrot(Complex(x, y))
    }
}
var image = Data("P5\n\(width) \(height)\n255\n".utf8)
image.append(contentsOf: pixels)
FileHandle.standardOutput.write(image)  // NOTE: ignoring errors

func mandelbrot(_ z: Complex<Double>) -> UInt8 {
    let iterations = 200
    let contrast = 15

    var v = Complex<Double>.zero
    for n in 0..<iterations {
        v = v * v + z
        if v.lengthSquared > 4 {
            return UInt8(truncatingIfNeeded: 255 - contrast * n)
        }
    }
    return 0  // black
}
```

The two nested loops iterate over each point in a 1024 × 1024 grayscale raster image representing the −2 to +2 portion of the complex plane. The program tests whether repeatedly squaring and adding the number that point represents eventually "escapes" the circle of radius 2. If so, the point is shaded by the number of iterations it took to escape. If not, the value belongs to the Mandelbrot set, and the point remains black. Finally, the program writes to its standard output the image in the simple PGM format: a short text header followed by one byte per pixel. Most image viewers can display PGM files, and tools like ImageMagick can convert them to PNG.

```
$ swift run -c release mandelbrot > mandelbrot.pgm
```

(The `-c release` flag builds with optimization. For numeric programs like this, an optimized build can be 10 or 50 times faster than a debug build, which keeps all the run-time checks and doesn't inline anything.)

Notice the conversion `UInt8(truncatingIfNeeded:)`. The expression `255 - contrast * n` ranges from 255 down to −2730, and we want its low-order 8 bits, so that the shading cycles through the gray levels as the iteration count increases. A plain `UInt8(...)` would trap for any value outside 0...255. This is a typical example of how Swift makes you say what you mean when an integer conversion can lose information.

We compare `lengthSquared` against 4 rather than `length` against 2 to avoid computing a square root on every iteration.

**Exercise 3.5:** Implement a full-color Mandelbrot set using the PPM format (`P6`, three bytes per pixel) instead of PGM.

**Exercise 3.6:** Supersampling is a technique to reduce the effect of pixelation by computing the color value at several points within each pixel and taking the average. The simplest method is to divide each pixel into four "subpixels." Implement it.

**Exercise 3.7:** Another simple fractal uses Newton's method to find complex solutions to a function such as *z*<sup>4</sup> − 1 = 0. Shade each starting point by the number of iterations required to get close to one of the four roots. Color each point by the root it approaches.

**Exercise 3.8:** Rendering fractals at high zoom levels demands great arithmetic precision. Implement the same fractal using `Complex<Float>`, `Complex<Double>`, and (on x86) `Complex<Float80>`. How do they compare in performance and memory usage? At what zoom levels do rendering artifacts become visible?

**Exercise 3.9:** Write a web server that renders fractals and writes the image data to the client. Allow the client to specify the *x*, *y*, and zoom values as parameters to the HTTP request.

## 3.4. Booleans

A value of type `Bool`, or *boolean*, has only two possible values, `true` and `false`. The conditions in `if`, `while`, and `guard` statements are booleans, and comparison operators like `==` and `<` produce a boolean result. The unary operator `!` is logical negation, so `!true` is `false`, or, one might say, `(!true == false) == true`, although as a matter of style, we always simplify redundant boolean expressions like `x == true` to `x`. A `var` boolean can be negated in place with `x.toggle()`.

Boolean values can be combined with the `&&` (AND) and `||` (OR) operators, which have *short-circuit* behavior: if the answer is already determined by the value of the left operand, the right operand is not evaluated, making it safe to write expressions like this:

```swift
s != "" && s.first == "x"  // fine even if s is empty
```

Since `&&` has higher precedence than `||` (mnemonic: `&&` is boolean multiplication, `||` is boolean addition), no parentheses are required for conditions of this form:

```swift
if "a" <= c && c <= "z" || "A" <= c && c <= "Z" || "0" <= c && c <= "9" {
    // ...ASCII letter or digit...
}
```

In a condition, a comma also means AND, and allows optional bindings to be mixed with boolean tests. Each clause can use the names bound by those before it:

```swift
if let first = s.first, first.isLetter, s.count > 3 {
    // ...
}
```

There is no implicit conversion from a boolean value to a numeric value like 0 or 1, or vice versa. It's necessary to use an explicit conditional:

```swift
let i = b ? 1 : 0
```

It might be worth writing a conversion function if this operation were needed often:

```swift
extension Int {
    /// Creates 1 from true and 0 from false.
    init(_ b: Bool) {
        self = b ? 1 : 0
    }
}

let n = Int(true)  // 1
```

The inverse operation is so simple that it doesn't warrant a function, but for symmetry here it is:

```swift
func itob(_ i: Int) -> Bool { i != 0 }
```

Notice that our `Int(_:)` initializer is added to the standard library's `Int` type with an *extension*. Extensions are covered in Chapter 6.

## 3.5. Strings

A Swift string is an immutable-by-default, mutable-if-`var` sequence of *characters*, where a character is what a human reader would consider a single symbol of text. This sounds obvious, but it's unusual. In C a string is a sequence of bytes, in Java and JavaScript it's a sequence of UTF-16 code units, and in Go it's a sequence of bytes that conventionally holds UTF-8. Swift is one of very few languages whose basic string abstraction is the *extended grapheme cluster* defined by the Unicode standard.

To see the difference, consider a string containing a flag emoji, which Unicode represents as two "regional indicator" code points, and an accented letter written as a base letter followed by a combining accent:

```swift
let s = "🇨🇦 cafe\u{301}"
print(s)  // "🇨🇦 café"
print(s.count)  // "6": 🇨🇦, space, c, a, f, é
print(s.unicodeScalars.count)  // "8"
print(s.utf16.count)  // "10"
print(s.utf8.count)  // "15"
```

The `count` property gives the number of characters; the string's *views*, `unicodeScalars`, `utf16`, and `utf8`, give access to its contents at the level of Unicode code points and of the two common encodings. All four are collections, so you can iterate over any of them:

```swift
for c in "héllo" {  // c is a Character
    print(c, terminator: " ")
}
// "h é l l o "
```

The price of character-level semantics is that strings can't be indexed by integer. Finding the *n*th character requires walking from the start of the string, because characters have variable width, and Swift refuses to make an O(*n*) operation look like an O(1) one. Instead, positions within a string are values of type `String.Index`, obtained from the string itself:

```swift
let hello = "hello, world"
print(hello[hello.startIndex])  // "h"
let i = hello.index(hello.startIndex, offsetBy: 7)
print(hello[i])  // "w"
print(hello[..<i])  // "hello, "
```

This takes some getting used to. In practice, you rarely need integer offsets, because the standard library provides higher-level operations like `prefix(_:)`, `dropFirst(_:)`, `firstIndex(of:)`, `split(separator:)`, `hasPrefix(_:)`, and `contains(_:)`:

```swift
print(hello.prefix(5))  // "hello"
print(hello.dropFirst(7))  // "world"
print(hello.hasPrefix("hell"))  // "true"
print(hello.contains("lo, w"))  // "true"
```

Subscripting a string with a range of indexes, or calling methods like `prefix` and `dropFirst`, produces a `Substring`. A substring shares storage with its base string, so making one is O(1), but it keeps the whole base string alive. Substrings are meant for temporary use, during parsing, for example. Convert one to a `String` before storing it for long:

```swift
let first = String(hello.prefix(5))
```

Strings may be compared with `==` and `<`. Comparison is by *canonical equivalence*: two strings are equal if they're made of the same characters, regardless of how those characters are encoded. So `"café"` written with a precomposed `é` (U+00E9) equals `"cafe\u{301}"`. Strings are ordered by comparing their Unicode scalar values, which is a stable order suitable for sorting keys but not the "dictionary" order that a human expects; for user-facing sorting, use Foundation's `localizedStandardCompare(_:)`.

Strings in Swift are values, like everything else. A `var` string can be modified in place with `append`, `+=`, `insert`, `remove`, and `replaceSubrange`, and modifying it never affects any other string, because copies share storage only until one of them is modified.

```swift
var s = "left foot"
let t = s
s += ", right foot"
print(s)  // "left foot, right foot"
print(t)  // "left foot"
```

### 3.5.1. String Literals

A string value can be written as a *string literal*, a sequence of characters enclosed in double quotes:

```swift
"Hello, 世界"
```

Within a double-quoted string literal, *escape sequences* that begin with a backslash `\` can be used to insert special characters. The full list is short:

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

So `"\u{4E16}\u{754C}"` is the same as `"世界"`.

A *raw string literal* is surrounded by one or more `#` characters: `#"..."#`. Within it, backslashes are ordinary characters, and escapes and interpolations must be written with the same number of `#` signs, as `\#n` or `\#(x)`. Raw strings are convenient for regular expressions and Windows paths:

```swift
let pattern = #"\d+\.\d+"#  // the regex \d+\.\d+
let path = #"C:\Users\ken"#
```

A *multiline string literal* begins and ends with `"""` on lines of their own, as we saw in Section 1.4. Line breaks within it are part of the string, and the indentation of the closing delimiter is stripped from every line, so it can be indented to fit the code around it. Multiline literals are convenient for HTML templates, JSON, help messages, and the like:

```swift
let usage = """
    swift run fetch url...
      -s    print status code
      -h    print headers
    """
```

### 3.5.2. Unicode

Long ago, life was simple and there was, at least in a parochial view, only one character set to deal with: ASCII, the American Standard Code for Information Interchange. ASCII, or more precisely US-ASCII, uses 7 bits to represent 128 "characters": the upper- and lower-case letters of English, digits, and a variety of punctuation and device-control characters. For much of the early days of computing, this was adequate, but it left a very large fraction of the world's population unable to use their own writing systems in computers. With the growth of the Internet, data in myriad languages has become much more common. How can this rich variety be dealt with at all and, if possible, efficiently?

The answer is *Unicode* (`unicode.org`), which collects all of the characters in all of the world's writing systems, plus accents and other diacritical marks, control codes like tab and carriage return, and plenty of esoterica, and assigns each one a standard number called a *Unicode code point*, or, in Swift terminology, a *Unicode scalar*. Swift represents one as a `Unicode.Scalar`, a 21-bit value.

That isn't the whole story, though. What a reader sees as one character may be made up of several code points: a letter followed by one or more combining accents, a flag made of two regional indicators, a family emoji made of several person emoji joined by zero-width joiners, a Hangul syllable built from its jamo, or a Devanagari letter with a vowel sign. Unicode defines rules for grouping code points into *extended grapheme clusters*, and Swift's `Character` type represents one such cluster. That's why `"🇨🇦".count` is 1 even though it is two scalars, and why `"e\u{301}" == "é"`.

`Character` has convenient properties based on the Unicode database, like `isLetter`, `isNumber`, `isWhitespace`, `isUppercase`, `isASCII`, and `asciiValue`, so you rarely need to look at scalars directly.

### 3.5.3. UTF-8

Swift strings are stored natively in UTF-8, a variable-length encoding of Unicode code points as bytes. UTF-8 was invented by Ken Thompson and Rob Pike, two of the creators of Go, and is now a Unicode standard. It uses between 1 and 4 bytes to represent each code point, but only 1 byte for ASCII characters, and only 2 or 3 bytes for most code points in common use. The high-order bits of the first byte of the encoding for a code point indicate how many bytes follow. A high-order 0 indicates 7-bit ASCII, where each code point takes only 1 byte, so it is identical to conventional ASCII. A high-order 110 indicates that the code point takes 2 bytes; the second byte begins with 10. Larger code points have analogous encodings.

```
0xxxxxxx                              code points 0–127          (ASCII)
110xxxxx 10xxxxxx                     128–2047                   (values <128 unused)
1110xxxx 10xxxxxx 10xxxxxx            2048–65535                 (values <2048 unused)
11110xxx 10xxxxxx 10xxxxxx 10xxxxxx   65536–0x10ffff             (other values unused)
```

A variable-length encoding precludes direct indexing to access the *n*th character of a string, but UTF-8 has many desirable properties to compensate. The encoding is compact, compatible with ASCII, and self-synchronizing: it's possible to find the beginning of a character by backing up no more than three bytes. It's also a prefix code, so it can be decoded from left to right without any ambiguity or lookahead. No code point's encoding is a substring of any other, or even of a sequence of others, so you can search for a code point by just searching for its bytes. The lexicographic byte order equals the Unicode code point order, so sorting UTF-8 works naturally. There are no embedded NUL (zero) bytes, which is convenient for programming languages that use NUL to terminate strings.

Since Swift 5, the `utf8` view of a native string is a direct view of its storage, so iterating over it is as fast as iterating over a byte array. When performance matters and you're processing ASCII-oriented data like CSV, JSON, or HTTP headers, working with the `utf8` view (a collection of `UInt8`) can be several times faster than working with characters:

```swift
let s = "Hello, 世界"
print(s.utf8.count)  // "13"
print(s.count)  // "9"

for (i, b) in s.utf8.enumerated() where b >= 0x80 {
    print(i, String(b, radix: 16))
}
// 7 e4
// 8 b8
// ...
```

To decode bytes into a string, use `String(decoding:as:)`, which replaces any ill-formed sequence with the Unicode replacement character, `\u{FFFD}`, printed as a white question mark inside a black hexagonal or diamond-like shape. Or use `String(validating:as:)` (Swift 6), which returns `nil` if the bytes are not valid UTF-8:

```swift
let bytes: [UInt8] = [0x48, 0x69, 0xFF]
print(String(decoding: bytes, as: UTF8.self))  // "Hi\u{FFFD}"
print(String(validating: bytes, as: UTF8.self) as Any)  // "nil"
```

### 3.5.4. Strings, Bytes, and the Standard Library

Most string processing in Swift uses the methods of `String` itself and of the protocols it conforms to, `Collection`, `BidirectionalCollection`, and `RangeReplaceableCollection`. These give strings the same algorithms as arrays: `map`, `filter`, `reversed`, `contains`, `first(where:)`, `split`, `joined`, `firstRange(of:)`, `replacing(_:with:)`, and many more. Foundation adds Unicode-aware case-insensitive comparison, trimming of whitespace, and locale-aware operations, and the standard library's `Regex` type (Swift 5.7) and its literal syntax `/.../` provide regular expressions.

The `basename` function below was inspired by the Unix shell utility of the same name. In our version, `basename(s)` removes any prefix of `s` that looks like a file system path with components separated by slashes, and it removes any suffix that looks like a file type:

```swift
print(basename("a/b/c.swift"))  // "c"
print(basename("c.d.swift"))  // "c.d"
print(basename("abc"))  // "abc"
```

The first version of `basename` does all the work without the help of libraries:

```swift
// swiftpl/ch3/basename1
// basename removes directory components and a .suffix.
// e.g., a => a, a.swift => a, a/b/c.swift => c, a/b.c.swift => b.c
func basename(_ path: String) -> String {
    var s = Substring(path)
    // Discard last "/" and everything before.
    var i = s.endIndex
    while i > s.startIndex {
        let prev = s.index(before: i)
        if s[prev] == "/" {
            s = s[i...]
            break
        }
        i = prev
    }
    // Preserve everything before last ".".
    i = s.endIndex
    while i > s.startIndex {
        let prev = s.index(before: i)
        if s[prev] == "." {
            s = s[..<prev]
            break
        }
        i = prev
    }
    return String(s)
}
```

A simpler version uses the collection method `lastIndex(of:)`:

```swift
// swiftpl/ch3/basename2
func basename(_ path: String) -> String {
    var s = Substring(path)
    if let slash = s.lastIndex(of: "/") {
        s = s[s.index(after: slash)...]
    }
    if let dot = s.lastIndex(of: ".") {
        s = s[..<dot]
    }
    return String(s)
}
```

Both versions slice a `Substring` rather than building new strings at each step, so they allocate memory only once, at the end. An index from a substring is valid in its base string and vice versa, which makes this kind of incremental slicing easy.

Foundation's `URL` and the `swift-system` package's `FilePath` have functions for manipulating hierarchical names like file paths and URLs, and should be used in real programs, where they correctly handle the many edge cases.

Let's look at another substring example. The task is to take a string representation of an integer, such as `"12345"`, and insert commas every three places, as in `"12,345"`. This version only works for integers; handling floating-point numbers is left as an exercise.

```swift
// swiftpl/ch3/comma
// comma inserts commas in a non-negative decimal integer string.
func comma(_ s: String) -> String {
    if s.count <= 3 {
        return s
    }
    let i = s.index(s.endIndex, offsetBy: -3)
    return comma(String(s[..<i])) + "," + s[i...]
}
```

The argument to `comma` is a string. If its length is less than or equal to 3, no comma is necessary. Otherwise, `comma` calls itself recursively with a substring consisting of all but the last three characters, and appends a comma and the last three characters to the result of the recursive call. The `+` operator can concatenate a `String` with a `Substring`, producing a `String`.

(Of course, for real output you'd use Foundation's number formatting, `12345.formatted()`, which inserts the separators appropriate for the user's locale.)

A `String` can be built up efficiently by appending to a `var`. If you're constructing it from bytes, accumulate them in a `[UInt8]` and decode once at the end. The `intsToString` function below converts an array of integers into a string representation that looks like an array literal, with the elements separated by commas and enclosed in square brackets:

```swift
// swiftpl/ch3/printints
// intsToString is like String(describing:) but adds spaces after the commas.
func intsToString(_ values: [Int]) -> String {
    var buf = "["
    for (i, v) in values.enumerated() {
        if i > 0 {
            buf += ", "
        }
        buf += String(v)
    }
    buf += "]"
    return buf
}

print(intsToString([1, 2, 3]))  // "[1, 2, 3]"
```

That's one of those cases where the library already has a better answer: `"[" + values.map(String.init).joined(separator: ", ") + "]"`. We'll explain `map` in Section 5.5.

**Exercise 3.10:** Write a non-recursive version of `comma`, building the result in a `var` string.

**Exercise 3.11:** Enhance `comma` so that it deals correctly with floating-point numbers and an optional sign.

**Exercise 3.12:** Write a function that reports whether two strings are anagrams of each other, that is, they contain the same letters in a different order. Should "listen" and "Silent" be anagrams? Should strings that differ only in Unicode normalization be considered anagrams?

### 3.5.5. Conversions between Strings and Numbers

In addition to conversions between strings, characters, and bytes, it's often necessary to convert between numeric values and their string representations. This is done with initializers on the destination type.

To convert an integer to a string, one option is string interpolation; another is `String(_:)`:

```swift
let x = 123
let y = "\(x)"
print(y, String(x))  // "123 123"
```

`String(_:radix:)` formats numbers in a different base, as we've seen.

To parse a string representing an integer, use the integer type's failable initializer, `Int(_:)`, `UInt8(_:)`, and so on, which returns `nil` if the string isn't a valid number or is out of range for the type. An optional `radix:` argument specifies the base:

```swift
let a = Int("123")  // Optional(123)
let b = Int("12a")  // nil
let c = Int("ff", radix: 16)  // Optional(255)
let d = Int8("300")  // nil: out of range
```

Parsing is strict: leading and trailing whitespace are not allowed, though a leading `+` or `-` is. Floating-point types have the same kind of initializer, `Double("3.14")`, which accepts decimal and hexadecimal notation, `"inf"`, and `"nan"`.

Because these initializers return optionals, the usual way to use them is with `if let`, `guard let`, or `??`:

```swift
guard let port = Int(portArgument), (1...65535).contains(port) else {
    fatalError("invalid port: \(portArgument)")
}
let retries = Int(retriesArgument) ?? 3
```

## 3.6. Literals and Constants

In Go, constants are expressions whose value is known to the compiler and whose evaluation is guaranteed to occur at compile time. Swift takes a different approach. It has no separate category of constant expressions; a `let` holds any value, computed whenever it's initialized. Instead, Swift makes *literals* flexible.

### 3.6.1. Literal Types

A numeric literal like `42` or `3.5` has no type of its own. Its type is inferred from context:

```swift
let a = 42  // Int, the default for integer literals
let b: Double = 42  // the literal 42 becomes a Double
let c: UInt8 = 42  // the literal 42 becomes a UInt8
let d = 42 + 3.5  // Double: 42 can be a Double, so it is
let e: Float = 1 / 3  // Float 0.33333334, not integer division
```

Each literal kind corresponds to a protocol. Any type that conforms to `ExpressibleByIntegerLiteral` can be initialized from an integer literal, and likewise for floating-point, string, boolean, array, dictionary, and `nil` literals. When a literal appears in an expression, the compiler picks the type from context, falling back on a default (`Int`, `Double`, `String`, `Bool`, `Array`, `Dictionary`) only when the context doesn't determine one. This gives many of the benefits of Go's untyped constants: a literal can be used wherever any numeric type is expected, without conversion, and a literal that doesn't fit is a compile-time error:

```swift
let big: UInt8 = 256  // compile error: integer literal '256' overflows when stored into 'UInt8'
```

But literals are not constants: once a literal becomes a value, it has an ordinary type and all the usual rules apply. Arithmetic on literals happens at the precision of the type the expression ends up with, not at some arbitrary precision as in Go. Thus:

```swift
let x = 1 << 70  // 0: an Int smart-shifted past its width
let y = 9_223_372_036_854_775_807 + 1  // compile error: arithmetic operation overflows
```

Because user-defined types can conform to the literal protocols, we saw in Section 2.5 how `Celsius` could be written as `let c: Celsius = 100`. Many standard library types do the same: a `Set<String>` can be written as an array literal, and an `Optional` can be written as `nil`.

Literals and simple constant expressions are folded at compile time by the optimizer. If you want to give a constant a name, a `let` at global or static scope is the idiom, and the compiler will usually turn it into an immediate value in the generated code:

```swift
let maxConnections = 1024
let timeout = 2.5  // seconds

enum Physics {
    static let speedOfLight = 299_792_458.0  // meters per second
}
```

The `enum Physics`, with no cases, is a common Swift idiom for a *namespace*: a type that can't be instantiated, used only to group related static members.

### 3.6.2. Enumerations

Go uses its constant generator `iota` to create sequences of related constants. Swift uses enumerations, which are true types, not just integers with names:

```swift
enum Weekday {
    case sunday, monday, tuesday, wednesday, thursday, friday, saturday
}

var today = Weekday.wednesday
today = .thursday  // type inferred from context
```

An enum may have a *raw value* type, in which case each case has a value of that type. For integer raw values, the compiler assigns 0, 1, 2, and so on, just as `iota` does, unless you specify otherwise. For string raw values, each case's raw value defaults to its name:

```swift
enum Weekday: Int, CaseIterable {
    case sunday, monday, tuesday, wednesday, thursday, friday, saturday
}

print(Weekday.tuesday.rawValue)  // "2"
print(Weekday(rawValue: 5)!)  // "friday"
print(Weekday.allCases.count)  // "7"
```

The initializer `Weekday(rawValue:)` is failable, since not every integer corresponds to a weekday. Conformance to `CaseIterable` makes the compiler generate an `allCases` collection containing every case in declaration order.

Unlike Go constants, enum values are type-safe: you can't pass an arbitrary integer where a `Weekday` is expected, and a `switch` over a `Weekday` must handle all seven cases.

### 3.6.3. Option Sets

As a more complex example of `iota`, Go's `net` package declares names for the bits of an unsigned integer that indicate the properties of a network interface. Swift's equivalent is a struct conforming to the `OptionSet` protocol:

```swift
struct NetFlags: OptionSet {
    let rawValue: UInt

    static let up = NetFlags(rawValue: 1 << 0)  // interface is up
    static let broadcast = NetFlags(rawValue: 1 << 1)  // interface supports broadcast access capability
    static let loopback = NetFlags(rawValue: 1 << 2)  // interface is a loopback interface
    static let pointToPoint = NetFlags(rawValue: 1 << 3)  // interface belongs to a point-to-point link
    static let multicast = NetFlags(rawValue: 1 << 4)  // interface supports multicast access capability
}
```

The struct wraps an integer (the `rawValue`) and declares each named flag as a static constant with one bit set. In exchange for those few lines, the `OptionSet` protocol supplies a full set of operations through its default implementations: `contains`, `insert`, `remove`, `union`, `intersection`, and more, plus array-literal syntax for creating a set:

```swift
var v: NetFlags = [.multicast, .up]
print(v.rawValue)  // "17"
print(v.contains(.up))  // "true"
v.remove(.up)
print(v.contains(.up))  // "false"
v.insert(.broadcast)
v.formUnion(.up)
print(v.contains([.broadcast, .up]))  // "true"
```

There's no need to write `isUp`, `turnDown`, and `setBroadcast` functions, as one would in Go, because the generic set operations already express them clearly.

As a final example, here's a set of named byte sizes. Since there's no exponentiation operator, we use shifts:

```swift
enum ByteSize {
    static let kiB = 1 << 10  // 1024
    static let miB = 1 << 20  // 1048576
    static let giB = 1 << 30  // 1073741824
    static let tiB = 1 << 40  // 1099511627776
    static let piB = 1 << 50  // 1125899906842624
    static let eiB = 1 << 60  // 1152921504606846976
}
```

Unlike in Go, the next step, `1 << 70`, can't be represented as an `Int`; as we saw above, the smart shift just produces 0. You'd need a larger type such as `UInt128`, because literal arithmetic is done in the type of the result.

**Exercise 3.13:** Write `static let` declarations for KB, MB, up through YB (powers of 1000) as compactly as you can. Which of them overflow `Int`? Which type could represent them all?

**Exercise 3.14:** Add a `description` to `NetFlags` that lists the names of the flags that are set, like `[up, multicast]`. Is there a way to avoid writing each name twice?
