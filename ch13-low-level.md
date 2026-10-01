# 13. Low-Level Programming

The design of Swift guarantees a number of safety properties that limit the ways in which a program can "go wrong." During compilation, type checking detects most attempts to apply an operation to a value that is inappropriate for its type, such as subtracting one string from another. Strict rules for conversions between types prevent confusion between a number and its text. Definite initialization rules ensure that no variable is read before it has been given a value. At run time, bounds checks stop an out-of-range array subscript, overflow checks stop an arithmetic wraparound, and optionals make the absence of a value a visible part of the type. Automatic reference counting frees memory when it's no longer needed, so a pointer to freed memory can't be formed in safe code. And Swift 6's concurrency checking rules out data races. Together, these make it impossible for an ordinary Swift program to corrupt memory or crash with the kind of unpredictable failures that plague C programs.

Safe Swift code works at a *high level of abstraction*, and the compiler is responsible for making the abstractions efficient. But now and then we must step outside these rules: to lay out memory exactly as a file format or network protocol demands, to talk to an operating-system API or a C library that expects pointers, to squeeze performance out of an inner loop where the abstractions cost too much, or to implement the abstractions themselves. The standard library's `Array` and `String` are written in Swift, on top of the unsafe operations we'll see here.

Swift's escape hatches are deliberately conspicuous. Every operation that can break the safety guarantees has the word *unsafe* in its name: `UnsafePointer`, `UnsafeMutableRawPointer`, `unsafeBitCast`, `withUnsafeBytes`, `unsafeDowncast`. A reader, or a `grep`, can find them all. Since Swift 6.2, the compiler can even enforce this: the *strict memory safety* mode (`-strict-memory-safety`) requires every use of an unsafe construct to be acknowledged with the `unsafe` keyword, so that a module's unsafe code is an explicit, auditable list. Unlike Go, whose `unsafe` package is a single entry point, Swift's unsafe operations are scattered through the standard library, but they share a naming convention that makes them as easy to find.

In this chapter, we'll see how to measure and control the layout of values, how to use unsafe pointers, how to call C code, and when not to do any of these things. Like reflection, low-level programming is a tool to be used sparingly, and only when it's really needed.

## 13.1. `MemoryLayout`: Size, Alignment, Stride, and Offset

Computers represent values as sequences of bytes in memory, and the way a type's values are arranged in those bytes is its *layout*. For most Swift code, layout is an implementation detail that the compiler is free to change. But when we need it, `MemoryLayout` reports it:

```swift
// swiftpl/ch13/layout
print(MemoryLayout<Bool>.size)  // 1
print(MemoryLayout<Int>.size)  // 8
print(MemoryLayout<Double>.size)  // 8
print(MemoryLayout<String>.size)  // 16
print(MemoryLayout<[Int]>.size)  // 8
print(MemoryLayout<Int?>.size)  // 9
```

`MemoryLayout<T>` has three static properties, which are always measured in bytes:

- `size`, the number of bytes a value of the type occupies.
- `alignment`, the address multiple at which values of the type must be stored. Processors read memory most efficiently, and in some cases only, at addresses that are multiples of the data's size, so each type has an alignment, usually its own size or the size of its largest scalar member.
- `stride`, the distance between consecutive elements in an array of the type. It's the size rounded up to the next multiple of the alignment.

Alignment and padding can introduce gaps in a struct. Consider these three declarations of structs with the same members, in different orders:

```swift
struct A { var flag: Bool; var count: Int32; var big: Int64 }
struct B { var flag: Bool; var big: Int64; var count: Int32 }
struct C { var big: Int64; var count: Int32; var flag: Bool }

for (name, size, stride, alignment) in [
    ("A", MemoryLayout<A>.size, MemoryLayout<A>.stride, MemoryLayout<A>.alignment),
    ("B", MemoryLayout<B>.size, MemoryLayout<B>.stride, MemoryLayout<B>.alignment),
    ("C", MemoryLayout<C>.size, MemoryLayout<C>.stride, MemoryLayout<C>.alignment),
] {
    print("\(name): size=\(size) stride=\(stride) alignment=\(alignment)")
}
```

On a 64-bit machine, the output is:

```
A: size=16 stride=16 alignment=8
B: size=24 stride=24 alignment=8
C: size=13 stride=16 alignment=8
```

Each struct holds a one-byte `Bool`, a four-byte `Int32`, and an eight-byte `Int64`, a total of 13 bytes. Swift lays out the stored properties of a struct *in declaration order*, inserting padding so that each property is aligned to its own alignment. In `A`, the flag takes a byte, then three bytes of padding bring the `Int32` to offset 4, and the `Int64` lands at 8, for 16 bytes in all. In `B`, the `Int64` can't start until offset 8, leaving seven bytes of padding after the flag, and the `Int32` ends at 20, which pads out to 24. In `C`, the large member comes first and the small ones fill in after it, so nothing is wasted between them; the `size` is 13, but since the alignment is 8, an *array* of `C` places each element 16 bytes apart: the stride.

The conclusion is the same as in Go: it pays to order the properties of a frequently allocated struct from largest alignment to smallest, if memory use matters. But Swift is more careful about promising anything. Unlike C, Swift makes *no guarantee* that stored properties are laid out in declaration order for ordinary structs, and in the future the compiler might reorder them. A struct whose layout *must* be exactly as written is one imported from C, which uses C's rules.

To find where a property sits, use `MemoryLayout<T>.offset(of:)`, which takes a key path:

```swift
print(MemoryLayout<A>.offset(of: \.flag)!)  // 0
print(MemoryLayout<A>.offset(of: \.count)!)  // 4
print(MemoryLayout<A>.offset(of: \.big)!)  // 8
```

It returns an optional, `nil` when the property isn't stored at a fixed offset, such as a computed property.

Here are the sizes of the common types on a 64-bit platform:

```
Int, UInt, Double, pointers   8 bytes
Float, Int32                  4 bytes
Bool, Int8, UInt8             1 byte
Character                     16 bytes   (a small string)
String                        16 bytes   (short strings, up to 15 UTF-8 bytes, are stored inline)
[T], [K: V], Set<T>           8 bytes    (a single reference to heap storage)
T?                            size of T plus one tag byte, unless T has spare bits (class references and pointers: no extra)
any Protocol                  40 bytes   (a three-word inline buffer, plus type metadata, plus witness table)
(Int) -> Int                  16 bytes   (code pointer and context pointer)
```

(The numbers are for illustration; they're implementation details, and they're why `MemoryLayout` exists.)

A word on `String`: the 16-byte representation hides a clever trick. Strings of up to 15 UTF-8 bytes are stored entirely in those 16 bytes, with no heap allocation (the *small string optimization*). Longer strings keep a pointer to heap storage, the length, and flags. This is why `"hello"` costs nothing to create, while building a long string costs an allocation.

Another consequence of layout: enums can use *spare bits* and unused bit patterns to store their tags. A `Bool?` takes just one byte, since a `Bool` has only two of the 256 possible values of a byte; the third state, `nil`, is stored as one of the unused patterns. And a class reference or pointer, which can never be zero, uses zero for `nil`, so `AnyObject?` is no larger than `AnyObject`.

## 13.2. Unsafe Pointers

Swift has pointers, but they look nothing like Go's or C's. An ordinary variable has no address you can take; `inout` arguments are the safe way to share one. Pointers are a family of generic types, each of which encodes what you can do with the memory it points to:

```
UnsafePointer<T>              points to initialized memory holding T values; read-only
UnsafeMutablePointer<T>       like UnsafePointer, but may be written
UnsafeRawPointer              points to untyped bytes; read-only
UnsafeMutableRawPointer       points to untyped bytes; may be written
UnsafeBufferPointer<T>        a pointer plus a count: a view of a contiguous run of T values
UnsafeMutableBufferPointer<T> like the above, but mutable
UnsafeRawBufferPointer        a view of a contiguous run of bytes
UnsafeMutableRawBufferPointer
```

All of them are unsafe because *nothing* checks that the memory is still allocated, that the index is in range, or that the type is the one actually stored there. The compiler and runtime trust the programmer. Misuse leads to the undefined behavior of C.

### 13.2.1. Scoped Access

The best way to use pointers is in a *scoped* closure that the standard library provides, where the pointer is valid only for the duration of the closure. The compiler guarantees that the underlying storage stays alive and isn't moved during the call, and, in return, you promise not to let the pointer escape:

```swift
// swiftpl/ch13/pointers
var x = 42
withUnsafeMutablePointer(to: &x) { p in
    p.pointee += 1  // read and write through the pointer
}
print(x)  // "43"

let numbers = [1, 2, 3, 4, 5]
let total = numbers.withUnsafeBufferPointer { buf in
    var sum = 0
    for i in 0..<buf.count {
        sum += buf[i]  // no bounds check in release builds
    }
    return sum
}
print(total)  // "15"
```

`withUnsafeBufferPointer` gives temporary access to an array's contiguous storage, which can eliminate bounds checks and be passed to C functions. The same family exists for strings (`withUTF8`), `Data` (`withUnsafeBytes`), and any contiguous collection. A pointer must not be used after its closure returns:

```swift
var escaped: UnsafePointer<Int>?
numbers.withUnsafeBufferPointer { escaped = $0.baseAddress }  // WRONG: pointer escapes
print(escaped!.pointee)  // undefined behavior: the array's storage may have moved or been freed
```

(In the Swift 6 language mode, the compiler catches this particular mistake only in simple cases. In general, the responsibility is yours.)

The standard library offers *safer* abstractions on top, for many of the tasks that once required pointers. `Span<T>` and `MutableSpan<T>` (Swift 6.2) are non-owning views of contiguous memory with bounds checking and compiler-enforced lifetimes, so they can't outlive the storage they refer to. `RawSpan` does the same for bytes. Where a `Span` is available, prefer it to `UnsafeBufferPointer`:

```swift
func sum(_ values: Span<Int>) -> Int {
    var total = 0
    for i in values.indices {
        total += values[i]  // bounds-checked, and guaranteed valid
    }
    return total
}

let array = [1, 2, 3]
print(sum(array.span))  // "6"
```

### 13.2.2. Manual Allocation

You can also allocate raw memory yourself, as `malloc` does. Memory has a *lifecycle* with explicit stages: it is allocated, then *initialized* with values, then *deinitialized* (which runs any needed cleanup for the values), then *deallocated*. Unsafe pointers expose each stage:

```swift
let p = UnsafeMutablePointer<String>.allocate(capacity: 3)  // allocated, uninitialized
p.initialize(to: "a")  // initialize p[0]
(p + 1).initialize(to: "b")
(p + 2).initialize(to: "c")

print(p[0], p[1], p[2])  // "a b c"

p.deinitialize(count: 3)  // run String's cleanup (releases any heap storage)
p.deallocate()  // free the memory
```

Pointer arithmetic is in units of the pointee type: `p + 1` is the address of the next `String`, which is `MemoryLayout<String>.stride` bytes along. Forgetting to `deinitialize` leaks the strings' storage; deinitializing twice, or deallocating without deinitializing, or using `p` after `deallocate()`, is undefined behavior. Plain `Int`s, which need no cleanup, can skip `deinitialize`; any type containing a reference, like `String` or `Array`, needs it.

This is precisely the code that sits beneath `Array`. The standard library's `ManagedBuffer` class packages the pattern, a heap object with a header and a tail-allocated buffer, for building your own collections safely and efficiently.

### 13.2.3. Raw Memory and Type Punning

Raw pointers see memory as untyped bytes. To read bytes as a typed value, use `load(fromByteOffset:as:)`, and to write one, `storeBytes(of:toByteOffset:as:)`. For example, here's how to find the byte order of the machine, by reading the bytes of an integer one at a time:

```swift
let value: UInt32 = 0x0102_0304
withUnsafeBytes(of: value) { bytes in
    print(Array(bytes))  // [4, 3, 2, 1] on a little-endian machine
}
```

And here's the inverse, assembling an integer from bytes of a file or network packet, which is the most common real use of raw memory:

```swift
let packet: [UInt8] = [0x00, 0x00, 0x01, 0x00, 0xDE, 0xAD, 0xBE, 0xEF]

let length = packet.withUnsafeBytes { raw in
    raw.loadUnaligned(fromByteOffset: 0, as: UInt32.self).bigEndian
}
print(length)  // "256"
```

`loadUnaligned` is necessary because the offset in the packet may not satisfy the alignment of `UInt32`; plain `load(as:)` requires proper alignment, and misaligned loads are undefined behavior (and on some processors, a crash). `.bigEndian` converts a value read in big-endian (network) order to the host's order. Better still, for protocol parsing, is to avoid pointers altogether and use `FixedWidthInteger.init(bigEndian:)` on shifted bytes, or a library like `swift-binary-parsing`, which do the same job in safe code.

Reinterpreting the *bits* of one type as another is called *type punning*, and Swift has three tools for it, in ascending order of danger:

```swift
// 1. A safe, defined conversion: the bit pattern of a Double
let d = 1.0
print(String(d.bitPattern, radix: 16))  // "3ff0000000000000"
print(Double(bitPattern: 0x4009_21FB_5444_2D18))  // "3.141592653589793"

// 2. Reinterpreting a value of the same size
let bits = unsafeBitCast(d, to: UInt64.self)  // same as d.bitPattern
// 3. Rebinding memory to a different type
```

Wherever a type has `bitPattern`, `init(bitPattern:)`, `init(truncatingIfNeeded:)`, or `withMemoryRebound`, prefer those over `unsafeBitCast`, which does no checking beyond the sizes. Swift's type system has an *aliasing rule* that says that accessing memory through a pointer of a type unrelated to the memory's *bound* type is undefined behavior, which is why there are explicit operations, `bindMemory(to:)` and `withMemoryRebound(to:capacity:)`, for changing the type of memory, and why raw pointers use `load`/`storeBytes` rather than plain access. If you break these rules, the optimizer may reorder or delete your reads and writes in ways that make the program misbehave in release builds but not in debug ones.

### 13.2.4. Why Pointers Are Dangerous

Go's book describes a collection of pitfalls of `unsafe.Pointer`, such as holding a `uintptr` that the garbage collector doesn't know about. Swift has no moving garbage collector, but it has its own list:

*Lifetimes.* A pointer obtained from `withUnsafePointer(to: &x)` or `array.withUnsafeBufferPointer` is valid only inside the closure. Storing it or returning it makes a dangling pointer. Likewise, the implicit conversion that lets you pass `&x` or an array to a C function that expects a pointer gives the pointer only for the duration of that call.

*Initialization state.* Reading from allocated-but-uninitialized memory, initializing already-initialized memory (leaking what was there), or deinitializing something that's uninitialized, are all undefined behavior.

*Bounds.* An `UnsafeBufferPointer` is range-checked only in debug builds, and an `UnsafePointer` is never. Reading one past the end, the classic buffer overrun, is exactly as easy in Swift as in C.

*Aliasing and exclusivity.* Swift's *law of exclusivity* assumes that a variable passed `inout` isn't accessed through any other path during the call. Pointers make it easy to break the assumption, and the compiler may exploit the assumption in optimization.

*Alignment.* Casting a byte pointer to a pointer of a larger type without verifying the alignment is undefined.

*Threads.* Pointers are not `Sendable`; sharing one among tasks is a data race unless you've arranged the synchronization yourself.

The tools for finding these bugs are the *Address Sanitizer* (`swift build --sanitize=address`), which catches out-of-bounds accesses, use-after-free, and leaks; the *Thread Sanitizer* (Section 9.6); and *Valgrind* on Linux. Run your tests under them whenever the code in question contains unsafe constructs.

## 13.3. Example: Deep Equivalence

Swift's `==` operator is defined by each type's `Equatable` conformance, and most types we use are `Equatable`: strings, numbers, arrays and dictionaries of `Equatable` elements, and structs that declare it. Some types, however, aren't. Functions and closures can't be compared, nor can types whose authors didn't conform them. And sometimes the built-in notion of equality is not the one we want. For testing, for instance, we might like to compare two complex data structures *structurally*: two values are *deeply equivalent* if they have the same shape and the same leaf values, no matter what their types say about `==`.

Go's `reflect.DeepEqual` does this, and the Go book builds a version using `unsafe.Pointer` to track the pairs of addresses already seen, so that cyclic structures don't cause infinite recursion. We'll build the Swift equivalent using `Mirror` for the structure and `ObjectIdentifier` for the cycle detection. This is the one place in this chapter where we need no unsafe pointers at all, a useful reminder that low-level doesn't have to mean unsafe.

```swift
// swiftpl/ch13/equal
/// Reports whether x and y are deeply equivalent.
func deepEqual(_ x: Any, _ y: Any) -> Bool {
    var seen = Set<Pair>()
    return equal(x, y, &seen)
}

private struct Pair: Hashable {
    let x, y: ObjectIdentifier
}

private func equal(_ x: Any, _ y: Any, _ seen: inout Set<Pair>) -> Bool {
    // Values of different dynamic types are never equivalent.
    guard type(of: x) == type(of: y) else { return false }

    // Basic types compare with ==.
    switch (x, y) {
    case let (a as Bool, b as Bool): return a == b
    case let (a as Int, b as Int): return a == b
    case let (a as Double, b as Double): return a == b || (a.isNaN && b.isNaN)
    case let (a as String, b as String): return a == b
    case let (a as Character, b as Character): return a == b
    default: break
    }
    if let a = x as? any BinaryInteger, let b = y as? any BinaryInteger {
        return a == b  // opens both existentials; same dynamic type, so this compiles to a typed ==
    }

    let mx = Mirror(reflecting: x)
    let my = Mirror(reflecting: y)
    guard mx.displayStyle == my.displayStyle else { return false }

    switch mx.displayStyle {
    case .class:
        // Cycle detection: if we're already comparing this pair, assume equal.
        let ix = ObjectIdentifier(x as AnyObject)
        let iy = ObjectIdentifier(y as AnyObject)
        if ix == iy { return true }  // the same instance
        guard seen.insert(Pair(x: ix, y: iy)).inserted else { return true }
        return equalChildren(mx, my, &seen)

    case .struct, .tuple, .collection, .optional, .enum:
        return equalChildren(mx, my, &seen)

    case .set:
        // Sets are unordered: each element of x must have an equivalent in y.
        let xs = Array(mx.children), ys = Array(my.children)
        guard xs.count == ys.count else { return false }
        return xs.allSatisfy { a in ys.contains { b in equal(a.value, b.value, &seen) } }

    case .dictionary:
        let xs = Array(mx.children), ys = Array(my.children)
        guard xs.count == ys.count else { return false }
        return xs.allSatisfy { a in
            let (ak, av) = pair(a.value)
            return ys.contains { b in
                let (bk, bv) = pair(b.value)
                return equal(ak, bk, &seen) && equal(av, bv, &seen)
            }
        }

    default:
        return false  // functions, metatypes, and other things we don't know how to compare
    }
}

private func equalChildren(_ mx: Mirror, _ my: Mirror, _ seen: inout Set<Pair>) -> Bool {
    let xs = Array(mx.children), ys = Array(my.children)
    guard xs.count == ys.count else { return false }
    for (a, b) in zip(xs, ys) {
        guard a.label == b.label, equal(a.value, b.value, &seen) else { return false }
    }
    return true
}

private func pair(_ value: Any) -> (Any, Any) {
    let c = Array(Mirror(reflecting: value).children)
    return (c[0].value, c[1].value)
}
```

The function begins by comparing the dynamic types of the two values, since values of different types are never equivalent. It then handles the basic types directly, including a special rule for floating-point NaN, which the built-in `==` says is unequal to itself but which a deep comparison sensibly treats as equal to another NaN. Everything else is compared by its children, using `Mirror`. Arrays and structs are compared element by element or field by field, so that `Optional` and `enum` values, whose children are their payload, are compared by the payload, and the labels must match as well, so that two different enum cases with equal payloads are not confused. Sets and dictionaries are unordered, so they're compared by looking for each element's partner, a quadratic algorithm that's fine for tests, and for a cleaner one, see Exercise 13.2.

Cyclic structures are the reason for `seen`. When two class instances are compared, the pair of their identities is recorded; if the same pair comes up again while we're still working on the first comparison, we're looking at a cycle, and we assume equality, which is correct if the rest of the comparison succeeds. This is the same technique as the Go original, which keys on pairs of pointers.

Let's test it:

```swift
print(deepEqual([1, 2, 3], [1, 2, 3]))  // true
print(deepEqual([1, 2, 3], [1, 2, 4]))  // false
print(deepEqual(["a": [1]], ["a": [1]]))  // true
print(deepEqual(1.0, Double.nan))  // false
print(deepEqual(Double.nan, Double.nan))  // true
print(deepEqual(Int8(1), Int16(1)))  // false: different types
print(deepEqual(nil as Int?, nil as Int?))  // true

// Circular linked lists.
final class Link {
    var value: Int
    var tail: Link?
    init(_ value: Int) { self.value = value }
}

func circle(_ values: Int...) -> Link {
    let nodes = values.map(Link.init)
    for i in nodes.indices { nodes[i].tail = nodes[(i + 1) % nodes.count] }
    return nodes[0]
}

print(deepEqual(circle(1, 2, 3), circle(1, 2, 3)))  // true
print(deepEqual(circle(1, 2, 3), circle(1, 2, 4)))  // false
```

Without the `seen` set, the comparison of the circular lists would recur until the stack overflowed.

Deep equivalence is mostly useful in tests, where you want to compare complex values for which defining `Equatable` would be overkill. (Our Swift Testing section's `#expect(a == b)` requires `Equatable`; a helper that wraps `deepEqual` fills the gap.) In production code, prefer to conform your types to `Equatable` and let the compiler synthesize `==`. It's faster, and the rules are explicit.

**Exercise 13.1:** Make `deepEqual` produce a *description of the first difference* it finds, such as `x.items[2].name: "a" != "b"`, to make test failures easier to diagnose.

**Exercise 13.2:** Improve the set and dictionary comparison so that it's not quadratic, in the cases where the elements are `Hashable`. (Hint: a cast to `any Hashable` can be opened, and `AnyHashable` can store the result.)

**Exercise 13.3:** Write a function `isCyclic(_ x: Any) -> Bool` that reports whether a value contains a reference cycle.

## 13.4. Calling C Code with Direct Interoperation

A great deal of useful code is written in C: operating system interfaces, database engines, compression and cryptography libraries, graphics drivers, and decades of other code. Swift was designed from its first day to use C libraries directly, with no wrapper code or *foreign function interface* layer, as Go needs `cgo` for. The Swift compiler includes a C compiler front end (Clang), and it can read C header files itself.

### 13.4.1. Importing C Headers

When you import a module that wraps a C library, the compiler reads the C declarations and presents them as Swift ones. The system libraries are already imported this way: `Foundation` brings in much of the C standard library, and on Linux the module `Glibc` (or `Musl`, or on Windows `WinSDK`) and on Apple platforms `Darwin` expose the rest. So we can call C library functions directly:

```swift
// swiftpl/ch13/ccalls
#if canImport(Darwin)
import Darwin
#elseif canImport(Glibc)
import Glibc
#elseif canImport(Musl)
import Musl
#endif

print(getpid())  // process ID, from <unistd.h>
print(strlen("hello"))  // "5", from <string.h>; the Swift String converts to a C string
print(abs(-3), sqrt(2.0))  // from <stdlib.h> and <math.h>

var now = time(nil)
var parts = tm()
localtime_r(&now, &parts)
print(1900 + Int(parts.tm_year))  // the current year
```

The compiler translates C types to Swift ones by simple rules:

```
C                          Swift
char, signed char          CChar (Int8)
unsigned char              CUnsignedChar (UInt8)
short, int, long           CShort, CInt, CLong (Int16, Int32, Int)
long long                  CLongLong (Int64)
size_t                     Int
float, double              Float, Double (CFloat, CDouble)
bool                       Bool
const T *                  UnsafePointer<T>?
T *                        UnsafeMutablePointer<T>?
void *                     UnsafeMutableRawPointer?
const char *               UnsafePointer<CChar>?   (accepts a Swift String as an argument)
struct S                   struct S, with the same members and an initializer
enum E                     struct E, with static constants (or a Swift enum for NS_ENUM)
function pointer           @convention(c) (...) -> ...
```

A C function that may return a null pointer returns a Swift *optional* pointer, so null is handled like any other `nil`. Calls to C functions are *always* unsafe in the sense that the compiler can't check what they do, but the translation of the types makes it easy to write correct code.

Most of the pointer conversions are done for you. A Swift `String` is automatically converted to a temporary C string when passed as a `const char *` argument; `&x` is converted to a pointer to a variable for the duration of the call; and an array passed as `UnsafePointer<T>` provides a pointer to its contiguous elements. That's what made `localtime_r(&now, &tm)` work above.

### 13.4.2. Wrapping a C Library

For libraries other than the system ones, you create a module that tells the compiler where to find the headers. SwiftPM provides two ways. The easy one is a C target, in the package itself, containing the C source files and a header in an `include` directory:

```
mypackage/
    Package.swift
    Sources/
        CBzip/
            bzip.c
            include/
                bzip.h
        bzip/
            main.swift
```

The C target is declared in the manifest as an ordinary `.target`, and Swift targets can depend on it and `import CBzip`.

The second is a *system library target*, which wraps a C library that's already installed on the machine, such as `libbz2`. It consists of a *module map* that names the header, and the library to link:

```
// Sources/CBz2/module.modulemap
module CBz2 [system] {
    header "shim.h"
    link "bz2"
    export *
}
```

```c
// Sources/CBz2/shim.h
#include <bzlib.h>
```

```swift
// Package.swift
.systemLibrary(name: "CBz2", pkgConfig: "bzip2", providers: [.apt(["libbz2-dev"]), .brew(["bzip2"])]),
```

Now `import CBz2` makes everything in `bzlib.h` available. The program below is a bzip2 compressor, built to the same design as the one in the Go book: it reads from the standard input and writes the compressed data to the standard output. This time, though, the Swift side is a *thin, safe wrapper* around the C API, so that the unsafe parts are confined to one small type:

```swift
// swiftpl/ch13/bzipper/Sources/bzipper/Writer.swift
import CBz2
import Foundation

/// A Writer compresses the bytes written to it and passes the result to a sink.
final class BZ2Writer {
    private var stream = bz_stream()
    private let sink: (Data) throws -> Void
    private var finished = false

    init(blockSize: Int32 = 9, sink: @escaping (Data) throws -> Void) throws {
        self.sink = sink
        let status = BZ2_bzCompressInit(&stream, blockSize, 0, 0)
        guard status == BZ_OK else { throw BZ2Error.code(status) }
    }

    /// Compresses the bytes in data.
    func write(_ data: Data) throws {
        var input = data
        try input.withUnsafeMutableBytes { (raw: UnsafeMutableRawBufferPointer) in
            stream.next_in = raw.baseAddress?.assumingMemoryBound(to: CChar.self)
            stream.avail_in = UInt32(raw.count)
            while stream.avail_in > 0 {
                try pump(action: BZ_RUN)
            }
        }
    }

    /// Flushes any buffered data and finishes the compressed stream.
    func close() throws {
        guard !finished else { return }
        finished = true
        stream.avail_in = 0
        defer { BZ2_bzCompressEnd(&stream) }
        var status = BZ_FINISH_OK
        while status != BZ_STREAM_END {
            status = try pump(action: BZ_FINISH)
        }
    }

    /// Runs one step of the compressor, delivering its output to the sink.
    @discardableResult
    private func pump(action: Int32) throws -> Int32 {
        var out = [CChar](repeating: 0, count: 64 * 1024)
        let status = out.withUnsafeMutableBufferPointer { buf -> Int32 in
            stream.next_out = buf.baseAddress
            stream.avail_out = UInt32(buf.count)
            return BZ2_bzCompress(&stream, action)
        }
        guard status >= 0 else { throw BZ2Error.code(status) }
        let produced = out.count - Int(stream.avail_out)
        if produced > 0 {
            try sink(Data(bytes: out, count: produced))
        }
        return status
    }

    deinit {
        if !finished { BZ2_bzCompressEnd(&stream) }
    }
}

enum BZ2Error: Error {
    case code(Int32)
}
```

and the main program, which is as simple as the safe API makes it:

```swift
// swiftpl/ch13/bzipper/Sources/bzipper/main.swift
// Bzipper reads input, bzip2-compresses it, and writes it out.
import Foundation

let out = FileHandle.standardOutput
let w = try BZ2Writer { data in try out.write(contentsOf: data) }
while case let chunk = FileHandle.standardInput.availableData, !chunk.isEmpty {
    try w.write(chunk)
}
try w.close()
```

Let's try it out, using a compressed copy of the text of this chapter's section list. Our compressor does what the standard `bzip2` utility does, as we can verify by decompressing its output with that:

```
$ swift build -c release
$ wc -c < ch13.md
38871
$ .build/release/bzipper < ch13.md | wc -c
10432
$ .build/release/bzipper < ch13.md | bunzip2 | cmp - ch13.md && echo ok
ok
```

(The sizes are illustrative.) Let's review the code, since it's a model of how to use C from Swift. The `bz_stream` struct, imported from the C header, is created with `bz_stream()`; Swift gives every imported C struct a zero-argument initializer that fills it with zeroes, which is exactly what the C library wants. Its fields appear as Swift properties, with the C names. `BZ2_bzCompressInit(&stream, ...)` passes the *address* of our stream, which is valid because `stream` is a stored property of a *class* (so it has a stable address) for the duration of the call; and since the C library keeps a pointer to the structure internally, the stream must not move. That's the reason `BZ2Writer` is a class and not a struct: a struct could be copied, or moved by the compiler.

The two `withUnsafe...` calls are the heart of the safety story. `Data.withUnsafeMutableBytes` and `Array.withUnsafeMutableBufferPointer` give the C code scoped, valid pointers to Swift-owned memory. We take care to finish all the C calls that use `next_in` and `next_out` *inside* those closures, so the pointers are never used after they become invalid. Failure to do so is the commonest bug in Swift code that wraps C.

All unsafe operations are confined to `BZ2Writer`. Its clients see a Swift API: `write`, `close`, and a thrown error, with no pointers, no manual `free`, and a `deinit` that releases the C library's resources if the client forgets, so the guarantees of ARC extend to the C library's lifetime rules. That's the way to wrap C code: write a thin layer that makes the C API safe, and keep it small enough to review carefully.

### 13.4.3. Callbacks

Many C libraries take *callbacks*, function pointers that they call back. Swift can pass a Swift function or non-capturing closure as a C function pointer, as long as it doesn't capture any context, since a C function pointer has nowhere to store captured values:

```swift
// qsort sorts an array using a comparison callback.
var values: [Int32] = [5, 2, 9, 1]
qsort(&values, values.count, MemoryLayout<Int32>.stride) { a, b in
    let x = a!.load(as: Int32.self), y = b!.load(as: Int32.self)
    return x < y ? -1 : (x > y ? 1 : 0)
}
print(values)  // "[1, 2, 5, 9]"
```

The closure is converted to a `@convention(c)` function pointer, which is allowed because it captures nothing. C libraries usually pass a `void *` *context* argument through to the callback, for exactly the purpose of letting it get at its state. In Swift, we use this to pass a reference to a Swift object, converting it to an opaque pointer with `Unmanaged`:

```swift
final class Handler {
    func receive(_ value: Int32) { print("got", value) }
}

let handler = Handler()
let context = Unmanaged.passUnretained(handler).toOpaque()  // a plain pointer
// ...pass `context` to a C function along with a @convention(c) callback...

// Inside the callback:
// let handler = Unmanaged<Handler>.fromOpaque(context).takeUnretainedValue()
```

`passUnretained` creates the pointer without changing the object's reference count, so the object must stay alive for as long as C may call back, which is the programmer's job to ensure; `passRetained` adds a reference count that the callback's cleanup must release. Reference counting and C's manual memory management have to be reconciled by hand here, because the compiler can't help.

### 13.4.4. Exposing Swift to C

The reverse direction is also possible. The attribute `@_cdecl("name")` exports a Swift function with a C calling convention and name, so that C code can call it, and it's how Swift libraries are used as plugins by C programs. The Swift compiler can also produce a C header for a module's public interface, with `-emit-clang-header-path`. For anything more than a few functions, a better route is C++ interoperability, which Swift supports directly: with `.interoperabilityMode(.Cxx)` in a target's settings, Swift code can import C++ headers and use C++ types and functions, including classes, templates (when instantiated), and the C++ standard library's containers, and C++ code can call into Swift.

```swift
// Package.swift
.target(name: "Wrapper", swiftSettings: [.interoperabilityMode(.Cxx)])
```

### 13.4.5. Memory Safety Annotations in C Headers

Pointers in C headers carry no information about ownership, length, or lifetime, which is why they import as unsafe pointers. Since Swift 6.2, headers can be annotated with Clang's *bounds-safety* and *lifetime* attributes, such as `__counted_by(n)` for a pointer with a known length, and Swift will import such a function with a safe `Span` or buffer parameter instead of a bare pointer. Where a library supports these annotations, calling it becomes safer without any change to the library's own code.

## 13.5. Another Word of Caution

We ended Chapter 12 with a warning about reflection, and this chapter's closing warning is stronger.

Unsafe code is code where you, not the compiler, are responsible for the properties of the program: that every pointer points to valid memory of the type you believe it holds, that memory is initialized exactly once and deinitialized exactly once, that nothing is accessed concurrently, and that none of Swift's assumptions are violated. If you're wrong, the result isn't an error message, or even a crash at the point of the mistake. It's *undefined behavior*: the program may crash later, in some unrelated place; produce wrong answers; behave one way in debug builds and another in release builds, one way on your machine and another on your customer's; or open a security hole through which an attacker can run code. Undefined behavior is not a bug that testing reliably finds. Some of the worst vulnerabilities in computing history are of this kind.

That calls for conservative use. A few rules of thumb will serve you well:

*Don't, unless you must.* The safe constructs, `Array`, `String`, `Data`, `Span`, and generics, are fast; most programs that seem to need pointers need only a better algorithm. Measure first (Chapter 11), and keep a safe version around to compare against.

*Prefer the narrowest tool.* A `Span` is safer than an `UnsafeBufferPointer`, which is safer than an `UnsafePointer`, which is safer than a raw pointer. `withUnsafeBytes(of:)` is safer than `unsafeBitCast`. A `bitPattern` initializer is safer than either. An `inout` is safer than a pointer to a variable.

*Use scoped access, not stored pointers.* The `withUnsafe...` family keeps the pointer valid for exactly the time you need it. Pointers that live in properties or global variables are the source of most use-after-free bugs.

*Confine and encapsulate.* Put unsafe code in the smallest possible type or function, with a safe public interface, and document the invariants it relies on in comments. Then you need to review only that code, and its clients can't violate its assumptions. The `BZ2Writer` of the last section is the model.

*Test hard.* Run the tests under the Address Sanitizer and the Thread Sanitizer, and use `swift build -c release` as well as debug, since undefined behavior often shows itself only with optimization. Use `-strict-memory-safety` to get a list of every unsafe construct in the module.

*Leave the language its freedom.* Don't depend on the layout of types that haven't promised one, on the internal representation of the standard library's types, or on the behavior of the compiler in the version you happen to have today.

With this chapter, we've reached the edge of what Swift's type system and run time guarantee. Beyond it lies hardware, and beyond that, everything else. Most of the time, you'll never need to cross the line. But when you do, you'll know that the language has a clear way of marking the crossing, and that it's your job to pay attention to what happens on the other side.

**Exercise 13.4:** Write a function that returns the number of bytes of padding in a struct, given its `MemoryLayout`. Apply it to several structs, and then reorder their properties to minimize it.

**Exercise 13.5:** Implement a simple growable array of `Int`s with `UnsafeMutablePointer`, with `append`, subscripting, `count`, and a `deinit` that frees the memory. Then make it a struct with copy-on-write semantics, using `ManagedBuffer` and `isKnownUniquelyReferenced`.

**Exercise 13.6:** Write a program that reads the first 8 bytes of a file and interprets them as a big-endian `UInt64`, in three ways: with `loadUnaligned`, with shifts, and with `UInt64(bigEndian:)`. Compare their speed.

**Exercise 13.7:** Using a C target in your package, write a small C function that sums an array of `int`s, and call it from Swift with an `Array<Int32>`. What does the imported Swift signature look like? Then add the `__counted_by` annotation and see how it changes.

**Exercise 13.8:** Write a wrapper for the bzip2 decompressor, `BZ2Reader`, modeled on `BZ2Writer`, and a program `bunzipper` that decompresses its standard input.

**Exercise 13.9:** Write a Swift wrapper for the C `getenv` and `setenv` functions, returning `String?`, and compare it with `ProcessInfo.processInfo.environment`. What does `getenv` return, and who owns the memory?
