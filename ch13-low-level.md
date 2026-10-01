# 13. Low-Level Programming

Everything in this book so far has been *safe* Swift. Safety here has a specific meaning: whatever mistakes a safe program makes, it can't corrupt memory. The guarantees come from many directions. The type checker rejects operations on the wrong kind of value. Definite-initialization analysis ensures that no variable is read before it's written. Array subscripts are bounds-checked, integer arithmetic is overflow-checked, and `nil` can appear only where an optional type allows it. Automatic reference counting frees objects only when nothing refers to them, so no reference can outlive its object. And since Swift 6, the compiler rules out data races. A safe program can still be wrong, and it can still crash, but its failures are deterministic and diagnosable: a trap with a message, at the point of the mistake.

Most programs never need to leave that world. Some do. A program might have to read a binary file format byte by byte, call a C library whose functions take raw pointers, manage memory by hand in a hot loop, or build a new data structure out of uninitialized storage, as the standard library's own collections are built. For such jobs Swift provides *unsafe* operations, which do exactly what they're told and check nothing.

Swift marks this territory clearly. Every type and function that can break memory safety carries *unsafe* in its name: `UnsafePointer`, `UnsafeMutableRawBufferPointer`, `withUnsafeBytes`, `unsafeBitCast`, `Unmanaged.takeUnretainedValue`. Such code stands out in review and is easy to find with a search. Swift 6.2 goes further with an opt-in *strict memory safety* mode (`-strict-memory-safety`), in which every use of an unsafe construct must be acknowledged with the `unsafe` keyword, so the compiler can produce a complete inventory of a module's unsafe code.

This chapter covers the layout of values in memory, the pointer types, an example of parsing binary data, and calling C. As with reflection in Chapter 12, the important skill isn't only how to do these things but when not to.

## 13.1. `MemoryLayout`: Size, Alignment, Stride, and Offset

Every value occupies some bytes of memory, arranged in some order: its *layout*. In safe Swift, layout is the compiler's business, and the compiler is free to choose. When you need to know it, perhaps to size a buffer, to match a binary format, or just to understand memory use, the generic `MemoryLayout` type reports it:

```swift
// swiftpl/ch13/layout
print(MemoryLayout<Bool>.size)  // 1
print(MemoryLayout<Int>.size)  // 8
print(MemoryLayout<Double>.size)  // 8
print(MemoryLayout<String>.size)  // 16
print(MemoryLayout<[Int]>.size)  // 8
print(MemoryLayout<Int?>.size)  // 9
```

`MemoryLayout<T>` reports three measurements, all in bytes:

- `size`: how many bytes a value of type `T` occupies.
- `alignment`: values of `T` must be placed at addresses that are a multiple of this number. Hardware accesses memory most efficiently, and on some processors only, when data is aligned to its size, so a type's alignment is usually that of its most demanding member.
- `stride`: the distance from one element to the next in an array of `T`, which is the size rounded up to a multiple of the alignment.

Alignment means that the order of a struct's properties can affect its size. Consider three versions of an event record containing the same three fields, a flag, a 32-bit code, and a 64-bit timestamp, in different orders:

```swift
// swiftpl/ch13/layout (continued)
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

On a 64-bit machine:

```
A: size=16 stride=16 alignment=8
B: size=20 stride=24 alignment=8
C: size=13 stride=16 alignment=8
```

The fields need 13 bytes in total. Today's compiler places a struct's stored properties in declaration order, adding padding wherever a property would otherwise be misaligned. In `A`, three bytes of padding after the `Bool` put the `Int32` at offset 4, and the `Int64` follows at offset 8: 16 bytes, no waste at the end. In `B`, the `Int64` must start at offset 8, so seven bytes after the `Bool` are wasted; the `Int32` then ends at offset 20, and arrays of `B` need four more bytes of padding per element to keep the next `Int64` aligned, for a stride of 24. In `C`, the largest field goes first and the others pack in behind it, for a size of 13, though the stride is still 16 because of the 8-byte alignment.

So when a struct is stored by the million, ordering its properties from largest alignment to smallest can save real memory. But note the word "today." Swift *doesn't promise* to lay out a native struct's properties in declaration order, and a future compiler could reorder them itself. The only struct layouts that are guaranteed are those of types imported from C, which follow C's rules. Never write code whose correctness depends on a Swift struct's layout.

`MemoryLayout<T>.offset(of:)` reports where a stored property begins, given a key path to it:

```swift
// swiftpl/ch13/layout (continued)
print(MemoryLayout<A>.offset(of: \.flag)!)  // 0
print(MemoryLayout<A>.offset(of: \.count)!)  // 4
print(MemoryLayout<A>.offset(of: \.big)!)  // 8
```

The result is optional because not every key path leads to stored data at a fixed offset; a computed property, for instance, gives `nil`.

For orientation, here are typical sizes on 64-bit platforms. They are implementation details, which is why `MemoryLayout` exists:

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

Two entries deserve comment. A `String` is 16 bytes no matter how long it is, but those bytes are used cleverly: a string of up to 15 UTF-8 bytes is stored entirely inside them, with no heap allocation at all, while a longer string stores a pointer to heap storage along with its length and some flags. That's why short strings are so cheap.

And optionals often cost nothing. An enum can store its case in bit patterns that its payload never uses. A `Bool` uses only two of a byte's 256 values, so `Bool?` keeps `nil` in one of the others and still fits in one byte. A class reference or a non-optional pointer is never zero, so its optional uses zero for `nil`, and `AnyObject?` is exactly as large as `AnyObject`.

## 13.2. Unsafe Pointers

Safe Swift has no address-of operator. You can't take a pointer to a variable and keep it; `inout` parameters (Section 2.3) cover the safe uses of passing a variable by reference. For everything else there's a family of pointer types, each describing what may be done with the memory it addresses:

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

The word *unsafe* means that nothing in the language or runtime verifies what you do with these types. Nothing checks that the memory is still allocated, that an index is in range, or that the memory actually holds a `T`. If any of those is wrong, the program's behavior is undefined, exactly as in C.

### 13.2.1. Scoped Access

The safest way to use a pointer is to borrow one for a limited time. A family of standard library functions does exactly that: each takes a closure, calls it with a pointer to some existing storage, and guarantees that the storage stays valid and doesn't move until the closure returns. Your side of the bargain is not to use the pointer afterwards:

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

`withUnsafeBufferPointer` exposes an array's contiguous element storage, which can be handed to C or iterated without bounds checks. Similar methods exist on strings (`withUTF8`), `Data` (`withUnsafeBytes`), and other contiguous collections. Smuggling the pointer out of the closure is a serious bug that the compiler can catch only in the simplest cases:

```swift
var escaped: UnsafePointer<Int>?
numbers.withUnsafeBufferPointer { escaped = $0.baseAddress }  // WRONG: pointer escapes
print(escaped!.pointee)  // undefined behavior: the array's storage may have moved or been freed
```

For many uses that once needed buffer pointers, Swift 6.2 offers a safe alternative. `Span<T>` is a read-only view of contiguous memory; `MutableSpan<T>` is a mutable one; and `RawSpan` views raw bytes. Spans check their bounds, and the compiler enforces that a span can't outlive the storage it views, so the escaping mistake above can't be written with one. Prefer spans wherever an API offers them:

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

Pointers can also manage memory you allocate yourself. Typed memory passes through four states, and the pointer API makes each transition explicit: memory is *allocated* (it exists but holds nothing), *initialized* (it holds valid values), *deinitialized* (the values are destroyed, releasing anything they refer to), and *deallocated* (it's returned to the system):

```swift
let p = UnsafeMutablePointer<String>.allocate(capacity: 3)  // allocated, uninitialized
p.initialize(to: "a")  // initialize p[0]
(p + 1).initialize(to: "b")
(p + 2).initialize(to: "c")

print(p[0], p[1], p[2])  // "a b c"

p.deinitialize(count: 3)  // run String's cleanup (releases any heap storage)
p.deallocate()  // free the memory
```

Arithmetic on a typed pointer counts in elements: `p + 1` is one `String` further on, `MemoryLayout<String>.stride` bytes away. Every step must be done exactly once and in order. Skipping deinitialization leaks whatever the values referred to. Deinitializing twice, reading uninitialized memory, or touching memory after deallocation is undefined behavior. (Types with no references inside them, such as `Int` and `Double`, need no cleanup, so for them deinitialization does nothing and may be skipped.)

This is the machinery underneath `Array` and the other collections. To build a collection of your own on it, start with the standard library's `ManagedBuffer`, which combines a reference-counted header with a tail-allocated block of elements and takes care of the allocation itself.

### 13.2.3. Raw Memory and Type Punning

A raw pointer sees memory as bytes, with no type attached. You read a typed value from raw memory with `load(fromByteOffset:as:)` (or `loadUnaligned`, discussed below) and write one with `storeBytes(of:toByteOffset:as:)`. Viewing an integer's bytes, for instance, reveals the machine's byte order:

```swift
let value: UInt32 = 0x0102_0304
withUnsafeBytes(of: value) { bytes in
    print(Array(bytes))  // [4, 3, 2, 1] on a little-endian machine
}
```

Going the other way, from bytes to a value, is the bread and butter of binary formats:

```swift
let packet: [UInt8] = [0x00, 0x00, 0x01, 0x00, 0xDE, 0xAD, 0xBE, 0xEF]

let length = packet.withUnsafeBytes { raw in
    raw.loadUnaligned(fromByteOffset: 0, as: UInt32.self).bigEndian
}
print(length)  // "256"
```

`load` requires the address to be properly aligned for the type being loaded; `loadUnaligned` doesn't, which is what you want for fields at arbitrary offsets in a byte stream. `.bigEndian` reinterprets the loaded value as big-endian (network byte order) on a little-endian host. Section 13.3 develops this into a complete example.

Treating the bits of one type as another type is called *type punning*. Swift offers several ways to do it, and they differ greatly in safety:

```swift
// 1. A safe, defined conversion: the bit pattern of a Double
let d = 1.0
print(String(d.bitPattern, radix: 16))  // "3ff0000000000000"
print(Double(bitPattern: 0x4009_21FB_5444_2D18))  // "3.141592653589793"

// 2. Reinterpreting a value of the same size
let bits = unsafeBitCast(d, to: UInt64.self)  // same as d.bitPattern

// 3. Viewing memory bound to one type as another
var pair: (UInt32, UInt32) = (1, 2)
withUnsafeMutablePointer(to: &pair) { p in
    p.withMemoryRebound(to: UInt32.self, capacity: 2) { words in
        print(words[0] + words[1])  // "3": the tuple's elements, read as UInt32s
    }
}
```

Use the purpose-built conversions (`bitPattern`, `init(bitPattern:)`, `init(truncatingIfNeeded:)`) wherever they exist. `unsafeBitCast` checks only that the two types have the same size. Rebinding memory is subtler still. Swift assumes that memory *bound* to one type isn't accessed as an unrelated type, and the optimizer relies on that assumption. Changing what a region of memory holds must go through `bindMemory(to:capacity:)` or `withMemoryRebound(to:capacity:)`, and byte-level access to typed memory should go through raw pointers. Code that breaks these rules may work in a debug build and fail mysteriously in an optimized one.

### 13.2.4. Why Pointers Are Dangerous

It's worth listing the ways pointer code goes wrong, because each is a mistake that safe Swift makes impossible:

*Lifetime.* A pointer is valid only while the memory it addresses is. Pointers from `withUnsafe...` functions expire when the closure returns, and the pointers produced by passing `&x` or an array to a C function expire when that call returns. Keeping one any longer creates a dangling pointer.

*Initialization.* Reading memory that was never initialized, initializing memory that already holds a value (which leaks the old value), or deinitializing memory that holds nothing are all undefined behavior.

*Bounds.* `UnsafeBufferPointer` checks bounds only in debug builds, and `UnsafePointer` never does. A one-past-the-end read or write is as easy in Swift as in C.

*Exclusivity.* Swift assumes that while a variable is being modified through `inout`, nothing else is accessing it (Section 2.3.2). A pointer can provide that "something else," and the optimizer won't expect it.

*Alignment.* Treating a byte address as a pointer to a larger type, without checking that it's suitably aligned, is undefined behavior.

*Concurrency.* Pointers aren't `Sendable`. Two tasks sharing one are in data-race territory unless they're synchronized by hand.

Tools help catch these errors at run time. The Address Sanitizer (`swift build --sanitize=address`) detects out-of-bounds accesses, use-after-free, and leaks; the Thread Sanitizer (Section 9.6) detects races; and Valgrind is available on Linux. Run the tests of any code containing unsafe constructs under them regularly.

## 13.3. Example: Reading a Binary File Header

Binary file formats are a common reason to work with raw bytes. Let's write a program, `pnginfo`, that reports the dimensions of PNG images by reading just the beginning of each file.

A PNG file starts with a fixed eight-byte *signature*. Then come a series of *chunks*, each with a 4-byte length, a 4-byte type code, the chunk's data, and a checksum. The first chunk must be the image header, of type `IHDR`, whose data begins with the image width and height as 4-byte big-endian integers, followed by a byte for the bit depth and one for the color type. Laid out by offset:

```
offset  size  contents
0       8     signature: 89 50 4E 47 0D 0A 1A 0A
8       4     length of the first chunk (13)
12      4     chunk type: "IHDR"
16      4     width, big-endian
20      4     height, big-endian
24      1     bit depth
25      1     color type
```

So the first 26 bytes tell us everything we want. Here's a function that decodes them:

```swift
// swiftpl/ch13/pnginfo
struct PNGInfo {
    var width: UInt32
    var height: UInt32
    var bitDepth: UInt8
    var colorType: UInt8
}

enum PNGError: Error {
    case tooShort
    case notPNG
    case missingHeader
}

private let signature: [UInt8] = [0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]

/// Decodes the image header from the first bytes of a PNG file.
func pngInfo(_ bytes: [UInt8]) throws -> PNGInfo {
    guard bytes.count >= 26 else { throw PNGError.tooShort }
    guard bytes.starts(with: signature) else { throw PNGError.notPNG }
    guard bytes[12..<16].elementsEqual("IHDR".utf8) else { throw PNGError.missingHeader }
    return bytes.withUnsafeBytes { raw in
        PNGInfo(
            width: UInt32(bigEndian: raw.loadUnaligned(fromByteOffset: 16, as: UInt32.self)),
            height: UInt32(bigEndian: raw.loadUnaligned(fromByteOffset: 20, as: UInt32.self)),
            bitDepth: raw[24],
            colorType: raw[25])
    }
}
```

and a main program that applies it to each file named on the command line, reading only the bytes it needs:

```swift
// swiftpl/ch13/pnginfo (continued)
import Foundation

for path in CommandLine.arguments.dropFirst() {
    do {
        let file = try FileHandle(forReadingFrom: URL(filePath: path))
        defer { try? file.close() }
        let head = try file.read(upToCount: 26) ?? Data()
        let info = try pngInfo(Array(head))
        print("\(path): \(info.width)×\(info.height), \(info.bitDepth)-bit, color type \(info.colorType)")
    } catch {
        print("\(path): \(error)")
    }
}
```

```
$ swift run pnginfo logo.png photo.jpg
logo.png: 512×512, 8-bit, color type 6
photo.jpg: notPNG
```

All the checks happen in safe code, *before* any raw memory is touched. Only after `bytes.count >= 26` has been established do we enter `withUnsafeBytes`, where the offsets 16 through 25 are therefore known to be in bounds. The integers are read with `loadUnaligned`, because offsets in a file have no reason to respect the alignment of `UInt32`, and converted with `UInt32(bigEndian:)`, since PNG stores integers most significant byte first. Everything unsafe is contained in one expression whose preconditions are checked right above it.

Two tempting shortcuts are worth rejecting explicitly. The first is to declare a Swift struct with fields in the order of the header and load the whole thing in one step, with `raw.load(as: Header.self)`. That would depend on the struct's layout matching the file's, which Swift doesn't guarantee (Section 13.1). It would also introduce padding between fields, and it would read multi-byte integers in the host's byte order rather than the file's. C programmers sometimes get away with this trick; in Swift it's simply wrong. The second shortcut is skipping the length check, on the grounds that real PNG files are always longer than 26 bytes. Files come from outside the program, and a truncated or hostile file would turn that assumption into an out-of-bounds read.

Is the unsafe code even necessary? Here's the same width computed with no pointers at all:

```swift
func bigEndianUInt32(_ bytes: ArraySlice<UInt8>) -> UInt32 {
    bytes.reduce(0) { $0 << 8 | UInt32($1) }
}

let width = bigEndianUInt32(bytes[16..<20])
```

With optimization, the compiler often turns this loop into the same single load-and-byte-swap instruction sequence as the pointer version. For a header read once per file, the difference is unmeasurable either way. When parsing large volumes of binary data, measure before reaching for pointers (Chapter 11). Spans (`bytes.span`, and `RawSpan` for bytes) offer bounds-checked access with performance close to raw pointers, and the `swift-binary-parsing` package provides a safe, declarative parser for exactly this kind of format.

**Exercise 13.1:** Extend `pnginfo` to walk all the chunks in a file, printing each chunk's type and length. Stop safely at the `IEND` chunk or at the end of the data, whichever comes first, and reject lengths that would run past the end.

**Exercise 13.2:** Rewrite `pngInfo` to use no unsafe code, with `bigEndianUInt32` or a `RawSpan`, and benchmark it against the version above on a million in-memory headers.

**Exercise 13.3:** Add support for another format, such as GIF (whose width and height are little-endian 16-bit integers at offsets 6 and 8), choosing the decoder by the file's signature.

## 13.4. Calling C Code with Direct Interoperation

An enormous amount of valuable code is written in C: operating-system interfaces, databases, compression, cryptography, image codecs, scientific libraries. Swift can call C functions directly, without writing glue code or using a separate binding generator. The Swift compiler contains Clang's C front end and reads C headers itself, presenting their declarations to Swift code as if they had been written in Swift.

### 13.4.1. Importing C Headers

The C standard library and operating-system APIs are already available as Swift modules: `Darwin` on Apple platforms, `Glibc` or `Musl` on Linux, and `WinSDK` on Windows (and `Foundation` re-exports much of them). After importing the right one, C functions are called like Swift functions:

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

C types map onto Swift types according to fixed rules:

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
pointer to opaque struct   OpaquePointer?
struct S                   struct S, with the same members and an initializer
enum E                     struct E, with static constants (or a Swift enum for NS_ENUM)
function pointer           @convention(c) (...) -> ...
#define N 42               let N: Int32 = 42   (simple constant macros only)
```

Pointers that C allows to be null become Swift optionals, so null handling uses the familiar tools. And the compiler converts arguments automatically in the common cases: a Swift `String` passed where C expects `const char *` becomes a temporary NUL-terminated copy, `&x` becomes a pointer to the variable `x`, and an array becomes a pointer to its first element. That's how `localtime_r(&now, &parts)` works above. Each of these temporary pointers is valid only for the duration of the call.

Not everything in a header can be imported. Function-like macros, and macros that do anything more than define a simple constant, have no Swift equivalent and are silently skipped. We'll meet one shortly.

### 13.4.2. Wrapping a C Library

A library that isn't part of the system needs a Swift module to present its headers. There are two ways to provide one. If you have the C sources, put them in a C target in your package: `.c` files in `Sources/CName/`, and public headers in `Sources/CName/include/`. SwiftPM compiles the C code and makes the headers importable as `CName`.

If the library is installed on the system, declare a *system library target*. It consists of a *module map*, which tells the compiler which header to read and which library to link, and usually a small "shim" header. Here's one for SQLite, the embedded SQL database engine:

```
// Sources/CSQLite/module.modulemap
module CSQLite [system] {
    header "shim.h"
    link "sqlite3"
    export *
}
```

```c
// Sources/CSQLite/shim.h
#include <sqlite3.h>
```

```swift
// Package.swift
.systemLibrary(name: "CSQLite", pkgConfig: "sqlite3",
               providers: [.apt(["libsqlite3-dev"]), .brew(["sqlite"])]),
```

The `pkgConfig` name lets SwiftPM find the library's flags with `pkg-config`, and the `providers` list tells users which package to install if it's missing. (On Apple platforms, the SDK already provides SQLite as the module `SQLite3`, so there you can skip the system library target and `import SQLite3`.)

With `import CSQLite`, all of SQLite's C API is available to Swift. Using it directly from application code, though, would scatter pointers, manual cleanup, and integer error codes everywhere. The right approach is to write a small wrapper that converts the C API into a safe Swift one, and to confine all the unsafe code to it:

```swift
// swiftpl/ch13/sqlite/Sources/SQLite/Database.swift
import CSQLite

public struct SQLiteError: Error, CustomStringConvertible {
    public let code: Int32
    public let message: String
    public var description: String { "SQLite error \(code): \(message)" }
}

/// SQLITE_TRANSIENT is a C macro that casts -1 to a function pointer,
/// which Swift can't import; this reproduces it. It tells SQLite to copy
/// bound strings, since Swift's temporary C strings don't outlive the call.
private var SQLITE_TRANSIENT: sqlite3_destructor_type {
    unsafeBitCast(-1, to: sqlite3_destructor_type.self)
}

/// A Database is a connection to an SQLite database.
public final class Database {
    private var handle: OpaquePointer?

    public init(path: String) throws {
        let rc = sqlite3_open(path, &handle)
        guard rc == SQLITE_OK else {
            let error = SQLiteError(code: rc, message: String(cString: sqlite3_errmsg(handle)))
            sqlite3_close(handle)  // a handle is allocated even on failure
            handle = nil
            throw error
        }
    }

    deinit {
        sqlite3_close(handle)
    }

    /// Executes one or more SQL statements that return no rows.
    public func execute(_ sql: String) throws {
        var message: UnsafeMutablePointer<CChar>?
        let rc = sqlite3_exec(handle, sql, nil, nil, &message)
        guard rc == SQLITE_OK else {
            defer { sqlite3_free(message) }
            throw SQLiteError(code: rc, message: message.map { String(cString: $0) } ?? "unknown")
        }
    }

    /// Runs a query with text parameters bound to its ? placeholders,
    /// and returns every row as an array of optional strings.
    public func query(_ sql: String, _ args: String...) throws -> [[String?]] {
        var stmt: OpaquePointer?
        guard sqlite3_prepare_v2(handle, sql, -1, &stmt, nil) == SQLITE_OK else {
            throw lastError()
        }
        defer { sqlite3_finalize(stmt) }

        for (i, arg) in args.enumerated() {
            sqlite3_bind_text(stmt, Int32(i + 1), arg, -1, SQLITE_TRANSIENT)
        }

        var rows: [[String?]] = []
        while true {
            let rc = sqlite3_step(stmt)
            if rc == SQLITE_DONE {
                break
            }
            guard rc == SQLITE_ROW else {
                throw lastError()
            }
            var row: [String?] = []
            for column in 0..<sqlite3_column_count(stmt) {
                if let text = sqlite3_column_text(stmt, column) {
                    row.append(String(cString: text))
                } else {
                    row.append(nil)  // SQL NULL
                }
            }
            rows.append(row)
        }
        return rows
    }

    private func lastError() -> SQLiteError {
        SQLiteError(code: sqlite3_errcode(handle), message: String(cString: sqlite3_errmsg(handle)))
    }
}
```

Clients of `Database` never see a pointer:

```swift
// swiftpl/ch13/sqlite/Sources/langs/main.swift
import SQLite

let db = try Database(path: ":memory:")
try db.execute("""
    CREATE TABLE langs (name TEXT, year INTEGER);
    INSERT INTO langs VALUES ('C', 1972), ('Go', 2009), ('Swift', 2014);
    """)
for row in try db.query("SELECT name, year FROM langs WHERE year > ? ORDER BY year", "2000") {
    print(row.map { $0 ?? "NULL" }.joined(separator: "\t"))
}
```

```
$ swift run langs
Go	2009
Swift	2014
```

Several patterns in the wrapper are typical of C interoperation.

*Opaque handles.* SQLite's `sqlite3` and `sqlite3_stmt` types are declared but never defined in its header, so Swift can't know their layout and imports pointers to them as `OpaquePointer`. Functions that return a handle through a `sqlite3 **` parameter are called with `&handle`, a pointer to our optional `OpaquePointer` variable.

*Ownership by a class.* A database connection is a resource with identity that must be released exactly once, which is why `Database` is a class with a `deinit` rather than a struct. A struct could be copied, and then two copies would each try to close the same connection. With a class, ARC ensures `sqlite3_close` runs once, when the last reference goes away. The wrapper extends Swift's memory management to the C library's resources.

*Scoped cleanup.* Every prepared statement is finalized by `defer`, and the error message that `sqlite3_exec` allocates is freed by `sqlite3_free` once it's been copied into a Swift string. Each acquisition sits next to its release.

*Strings in and out.* Passing a Swift `String` for a `const char *` parameter gives the C function a temporary copy that disappears when the call returns. SQLite's `bind` functions, however, may hold onto the pointer until the statement runs. The final argument of `sqlite3_bind_text` says how to handle that, and `SQLITE_TRANSIENT` tells SQLite to make its own copy. Because that constant is defined in C as a macro that casts -1 to a function pointer, Swift can't import it, and we re-create it with `unsafeBitCast`, one of the few places where that function is the right tool. Results come back as C strings that SQLite owns and may free at the next call, so `String(cString:)` copies each one immediately.

*Errors.* SQLite reports failure with integer result codes. The wrapper converts them to a thrown Swift error with a readable message, so callers use `try` and `catch` like anywhere else.

A wrapper like this is where unsafe code belongs: small, reviewed carefully, and presenting an interface on which ordinary Swift code can rely. One limitation is worth noting. `Database` is not `Sendable`, and shouldn't be. An SQLite connection must not be used from two threads at once without extra configuration, and leaving the class non-`Sendable` makes the compiler enforce that. To share a connection across tasks, put it inside an actor (Exercise 13.6).

### 13.4.3. Callbacks

Many C APIs call back into the caller through function pointers. A Swift closure can be passed as a C function pointer as long as it captures no context, because a C function pointer is just a code address, with nowhere to keep captured values. The C library's `qsort` makes a small example:

```swift
// qsort sorts an array using a comparison callback.
var values: [Int32] = [5, 2, 9, 1]
qsort(&values, values.count, MemoryLayout<Int32>.stride) { a, b in
    let x = a!.load(as: Int32.self), y = b!.load(as: Int32.self)
    return x < y ? -1 : (x > y ? 1 : 0)
}
print(values)  // "[1, 2, 5, 9]"
```

Most callback-based C APIs also accept a `void *` "context" or "user data" pointer that they pass back to the callback, so that it can find its state. To pass a Swift object that way, convert the reference to a raw pointer and back with `Unmanaged`:

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

`passUnretained` makes the pointer without adding a reference, so it's up to you to keep the object alive for as long as C might call back; `passRetained` adds a reference that must later be balanced with `release()`. This is the boundary where ARC's automatic bookkeeping meets C's manual rules, and the compiler can't check that you've reconciled them correctly.

### 13.4.4. Exposing Swift to C

Calls can also go the other way. The `@c` attribute (SE-0495) exports a Swift function to C, optionally under a different name, as in `@c(my_callback)`, with the C calling convention. Compilers that predate it offer the underscored, unofficial `@_cdecl("name")`, which much existing code uses. This is how Swift code can serve as a plug-in for a C program, and the compiler can generate a C header for a module with `-emit-clang-header-path`. For larger interfaces, Swift's C++ interoperability is usually the better route. Enabling it on a target lets Swift import C++ headers, including classes, instantiated templates, and standard library containers, and lets C++ call into Swift:

```swift
// Package.swift
.target(name: "Wrapper", swiftSettings: [.interoperabilityMode(.Cxx)])
```

### 13.4.5. Memory Safety Annotations in C Headers

A C pointer parameter says nothing about how many elements it refers to or how long it must remain valid, which is why C functions import with unsafe pointer types. Starting with Swift 6.2, the compiler can use Clang's *bounds-safety* and *lifetime* annotations to do better. If a header marks a parameter with, for example, `__counted_by(n)` to say that it points to `n` elements, Swift can generate a safe overload of the function, alongside the original, that takes a `Span` and passes the count for you. Libraries that adopt these annotations become safer to call from Swift with no change to their implementation. Support is new and still expanding, and may need to be enabled with a compiler setting, so check the documentation for your toolchain.

## 13.5. Another Word of Caution

Chapter 12 ended by urging restraint with reflection. The case for restraint with unsafe code is stronger, because the consequences of a mistake are worse.

When reflective code goes wrong, the result is a wrong answer or a trap. When unsafe code goes wrong, the result is *undefined behavior*: the language no longer promises anything about what the program does. It might crash immediately, or much later in unrelated code. It might quietly compute wrong results, work in debug builds and fail in release builds, work on your machine and fail on a customer's, or work for years and then fail after a compiler upgrade. In the worst case, a memory-safety bug lets an attacker take control of the program. Tests can't be relied on to find such bugs, because the behavior they cause may not show up in any particular run. A large share of the security vulnerabilities in widely used software have been memory-safety bugs of exactly this kind, which is a large part of why Swift was designed to be safe by default.

If you must write unsafe code, these habits keep the risk contained:

*Exhaust the safe options first.* `Array`, `String`, `Data`, `Span`, and generic code are fast; a better algorithm usually beats a pointer trick. Keep a safe implementation alongside any unsafe one and measure the difference (Chapter 11). If there isn't one, delete the unsafe code.

*Choose the least powerful tool that works.* A `Span` before an `UnsafeBufferPointer`, a buffer pointer before a bare `UnsafePointer`, a typed pointer before a raw one. A conversion like `bitPattern` before `withUnsafeBytes(of:)`, and either before `unsafeBitCast`. An `inout` parameter before a pointer to a variable.

*Borrow pointers; don't keep them.* Use the `withUnsafe...` functions and finish with the pointer inside the closure. Pointers stored in properties and globals are where use-after-free bugs come from.

*Wrap and isolate.* Put unsafe code behind a small type or function with a safe interface, as `Database` does, and state in comments the conditions it relies on. Then only that code needs careful review, and nothing outside it can violate its assumptions.

*Validate before you touch.* Check lengths, counts, and offsets in safe code, as `pngInfo` does, so that the unsafe part operates only on facts already established.

*Test under the sanitizers.* Run the tests with the Address Sanitizer and the Thread Sanitizer, in both debug and release builds, and turn on `-strict-memory-safety` to keep an up-to-date inventory of unsafe code.

*Rely only on promises.* Don't depend on the layout of Swift types, the internals of standard library types, or the particular behavior of the compiler version you happen to be using.

Unsafe code is the boundary where Swift's guarantees end and responsibility passes to the programmer. Most Swift programs never need to cross it. For those that must, Swift makes the crossing explicit, local, and searchable, so that the code that needs the most scrutiny is easy to find.

**Exercise 13.4:** Write a function that, given the sizes and alignments of a struct's fields in declaration order, computes the struct's size and the bytes lost to padding. Check it against `MemoryLayout` for several structs, then find the property order that minimizes each one's stride.

**Exercise 13.5:** Implement a growable array of `Int`s on top of `UnsafeMutablePointer`, with `append`, a subscript, `count`, and a `deinit` that frees the storage. Then turn it into a copy-on-write struct, using `ManagedBuffer` and `isKnownUniquelyReferenced`.

**Exercise 13.6:** Wrap `Database` in an actor so that one connection can be shared safely by many tasks. Which methods need to be `async` from the callers' point of view?

**Exercise 13.7:** Add typed column accessors to `Database`, returning `Int`, `Double`, `String`, or `[UInt8]` according to SQLite's column type (`sqlite3_column_type`), and support binding `Int` and `Double` parameters as well as strings.

**Exercise 13.8:** Add a method `transaction(_ body: () throws -> Void)` to `Database` that runs `body` between `BEGIN` and `COMMIT`, and issues `ROLLBACK` instead if `body` throws.

**Exercise 13.9:** Create a C target containing a function that sums an array of `int`s, and call it from Swift with an `[Int32]`. Look at the imported Swift signature. Then annotate the parameter with `__counted_by` and see how the signature changes.

**Exercise 13.10:** Write a Swift wrapper for C's `getenv` and `setenv`, returning `String?`. Who owns the memory `getenv` returns, and when might it change? Compare your wrapper with `ProcessInfo.processInfo.environment`.
