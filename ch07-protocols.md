# 7. Protocols

Protocol types express generalizations or abstractions about the behaviors of other types. By generalizing, protocols let us write functions that are more flexible and adaptable because they are not tied to the details of one particular implementation.

Many object-oriented languages have some notion of interfaces, and Swift's protocols descend from Objective-C's protocols, which in turn borrowed from Smalltalk. But Swift's protocols go much further. They can require properties, methods, initializers, subscripts, and operators; they can be extended with default implementations; they can have *associated types* that make them generic; and they can constrain the parameters of generic functions, so that the compiler checks, and optimizes, code written against an abstraction just as it would code written against a concrete type.

Unlike Go's interfaces, Swift's protocols are *nominal*: a type conforms to a protocol only if it says so, in its declaration or in an extension. As we'll see, this is less restrictive than it sounds, because a conformance can be added to any type at any time, even one defined in another module.

This chapter starts by looking at the mechanics of protocols and conformance. We'll then look at generics, which in Swift are inseparable from protocols, and at the dynamic side of protocols: protocol values, casts, and type switches. Along the way, we'll see several important protocols from the standard library.

## 7.1. Protocols as Contracts

All the types we've looked at so far have been *concrete types*. A concrete type specifies the exact representation of its values and exposes the intrinsic operations of that representation, such as arithmetic for numbers, or indexing, `append`, and `count` for arrays. A concrete type may also provide additional behaviors through its methods. When you have a value of a concrete type, you know exactly what it *is* and what you can *do* with it.

There is another kind of type in Swift called a *protocol type*. A protocol is an *abstract type*. It doesn't expose the representation or internal structure of its values, or the set of basic operations they support; it reveals only some of their methods. When you have a value of a protocol type, you know nothing about what it *is*; you know only what it can *do*, or more precisely, what behaviors are provided by its methods.

Throughout the book, we've been using `print` to write output. By default it writes to the standard output, but it has a sibling that writes to any *output stream*:

```swift
func print(_ items: Any..., separator: String = " ", terminator: String = "\n",
           to output: inout some TextOutputStream)
```

`TextOutputStream` is a protocol from the standard library with a single requirement:

```swift
protocol TextOutputStream {
    /// Appends the given string to the stream.
    mutating func write(_ string: String)
}
```

The `TextOutputStream` protocol defines the contract between `print` and its callers. On the one hand, the contract requires that the caller provide a value of a concrete type, like a `String` or a file stream, that has a method called `write` with the appropriate signature and behavior. On the other hand, the contract guarantees that `print` will do its job given any value that satisfies the protocol. `print` may not assume that it's writing to a file or to memory, only that it can call `write`.

Because `print` assumes nothing about the representation of the value and relies only on the behaviors guaranteed by the contract, we can safely pass a value of any concrete type that conforms to `TextOutputStream` as the argument. This freedom to substitute one type for another that satisfies the same protocol is called *substitutability*, and it's a hallmark of object-oriented programming.

`String` conforms to `TextOutputStream`, so a string can serve as an in-memory buffer:

```swift
var buffer = ""
print("hello", "world", to: &buffer)
print(buffer, terminator: "")  // "hello world\n"
```

Let's test this out using a new type. The `write` method of the `ByteCounter` type below merely counts the bytes written to it before discarding them:

```swift
// swiftpl/ch7/bytecounter
struct ByteCounter: TextOutputStream {
    var count = 0

    mutating func write(_ string: String) {
        count += string.utf8.count
    }
}
```

Since `ByteCounter` satisfies the `TextOutputStream` contract, we can pass it to `print`, which does its string formatting oblivious to this change; the `ByteCounter` correctly accumulates the length of the result.

```swift
var c = ByteCounter()
print("hello", to: &c)
print(c.count)  // "6", = len("hello\n")

c = ByteCounter()  // reset the counter
let name = "Dolly"
print("hello, \(name)", to: &c)
print(c.count)  // "13", = len("hello, Dolly\n")
```

The declaration `struct ByteCounter: TextOutputStream` states the conformance. The compiler checks that the type provides everything the protocol requires (here, a `write` method), and reports an error if it doesn't.

There is another protocol of great importance to output. Formatting the textual representation of a value, as `print` and string interpolation do, relies on the `CustomStringConvertible` protocol:

```swift
protocol CustomStringConvertible {
    var description: String { get }
}
```

We've already conformed several types to it, including `Celsius` in Section 2.5 and `IntSet` in Section 6.5. A protocol requirement can be a property as well as a method. Here `{ get }` means that conforming types must provide a readable property; a stored `let`, a stored `var`, or a computed property will all do. A requirement written `{ get set }` must be satisfied by something writable.

**Exercise 7.1:** Using the ideas from `ByteCounter`, implement counters for words and for lines. You'll find `split(whereSeparator:)` useful.

**Exercise 7.2:** Write a generic function `countingStream` with the signature below that, given a `TextOutputStream`, returns a new stream that wraps the original and keeps track of the number of bytes written to it.

```swift
struct CountingStream<Base: TextOutputStream>: TextOutputStream { ... }
func countingStream<S: TextOutputStream>(_ base: S) -> CountingStream<S>
```

**Exercise 7.3:** Write a `description` property for the `Tree` type from Section 4.4 that reveals the sequence of values in the tree.

## 7.2. Protocol Types

A protocol declaration specifies a set of requirements, the members that a concrete type must possess to be considered a conforming type.

Foundation and the standard library define many protocols, from the very small, like `Equatable` (one operator), to the substantial, like `Collection` (a dozen requirements, most with defaults). Here are some you'll meet often:

```swift
protocol Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool
}

protocol Hashable: Equatable {
    func hash(into hasher: inout Hasher)
}

protocol Comparable: Equatable {
    static func < (lhs: Self, rhs: Self) -> Bool
}

protocol Error: Sendable {}
```

The `Self` in `Equatable` refers to the conforming type, so `Int`'s `==` compares two `Int`s and `String`'s compares two `String`s.

Protocols can *inherit* from other protocols. `Hashable: Equatable` means that every `Hashable` type must also be `Equatable` and satisfy its requirements. A protocol can inherit from several others: `protocol BidirectionalCollection: Collection`, and `protocol RandomAccessCollection: BidirectionalCollection`, and so on.

Protocols can also be *composed* without declaring a new protocol, using `&`:

```swift
typealias Codable = Decodable & Encodable

func save(_ item: some Codable & Sendable) { ... }
```

This is exactly how `Codable` is defined in the standard library.

A protocol can declare requirements of many kinds. Here's a protocol that shows most of them:

```swift
protocol Shape {
    init(boundingBox: Rect)  // an initializer
    var area: Double { get }  // a read-only property
    var name: String { get set }  // a read-write property
    func contains(_ p: Point) -> Bool  // a method
    mutating func scale(by factor: Double)  // a mutating method
    static var unit: Self { get }  // a static property
    subscript(vertex i: Int) -> Point { get }  // a subscript
}
```

A `mutating` requirement may be satisfied by a mutating method of a struct, or by an ordinary method of a class. A protocol that should be adopted only by classes can say so by inheriting from `AnyObject`, which lets it be used with `weak` references:

```swift
protocol Delegate: AnyObject {
    func didFinish()
}

final class Downloader {
    weak var delegate: (any Delegate)?
}
```

**Exercise 7.4:** Define a protocol `Stack` with an associated `Element` type and requirements `push`, `pop`, and `isEmpty`. Conform `Array` to it with an extension.

## 7.3. Protocol Conformance

A type *conforms* to a protocol if it declares the conformance and possesses all the members the protocol requires. For example, `String` conforms to `TextOutputStream` and `CustomStringConvertible`, among many others. As a shorthand, we often say that a concrete type "is a" particular protocol type, meaning that it conforms to the protocol. For example, a `String` is a `TextOutputStream`, and `[Int]` is a `Collection`.

A conformance can be declared where the type is declared, or later, in an extension, which is the common style for organizing code:

```swift
struct Track {
    var title, artist, album: String
    var year: Int
    var length: Duration
}

extension Track: CustomStringConvertible {
    var description: String { "\(title) by \(artist)" }
}

extension Track: Equatable {}  // == is synthesized
```

Because conformances can be added by extensions, a type can be made to conform to a protocol that its author never heard of, by anyone who needs it to. You can conform your types to standard library protocols, and you can conform standard library types to your protocols. That captures much of the flexibility of Go's implicit, structural interfaces, while keeping the intent explicit.

There's one caveat. If you conform a type from one module to a protocol from another module, and neither is yours, two different modules might both do it, with different implementations. Swift 6 warns about such *retroactive conformances*. If you're confident it's what you want, acknowledge the risk by writing `@retroactive`:

```swift
import Foundation
import ArgumentParser

extension URL: @retroactive ExpressibleByArgument { ... }
```

The assignability rule (Section 2.4.2) for protocol types is simple: a value may be assigned to a protocol type only if its type conforms. So:

```swift
var w: any TextOutputStream
w = ""  // OK: String conforms
w = ByteCounter()  // OK: ByteCounter conforms
w = 42  // compile error: Int does not conform to TextOutputStream
```

A protocol type also hides the members of its underlying value that aren't part of the protocol:

```swift
var counter: any TextOutputStream = ByteCounter()
counter.write("hello")  // OK: write is a TextOutputStream requirement
print(counter.count)  // compile error: value of type 'any TextOutputStream' has no member 'count'
```

A protocol with more requirements, such as `BidirectionalCollection`, tells us more about the values it contains, and places greater demands on the types that implement it, than does a protocol with fewer, such as `Sequence`. What does the protocol `Any`, which has no requirements at all, tell us about the values that conform to it? Nothing. `Any` is a protocol composition of zero protocols, and every type conforms to it. That's why `print` can accept values of any type: its parameter is `Any...`. But we can't do much with a value of type `Any` directly, because it has no methods. We need a way to get the value back out again. We'll see how to do that using casts in Section 7.10.

A conforming type can satisfy a requirement in several ways: with a member it declares itself, with a member inherited from a superclass, or with a *default implementation* provided by an extension of the protocol. Default implementations make it possible to design rich protocols with only a few true requirements. `Sequence`, for example, requires just one method, `makeIterator()`, and its extensions supply `map`, `filter`, `reduce`, `contains`, `min`, `max`, `sorted`, `first(where:)`, `enumerated`, `zip`, `joined`, and many more. We'll implement a sequence of our own in Section 7.7.

Each grouping of concrete types based on their shared behaviors can be expressed as a protocol type. Unlike class-based languages, in which the set of interfaces a class satisfies is fixed when the class is written, in Swift we can define new abstractions or groupings of interest when we need them, without modifying the declaration of the concrete type. This is particularly useful when the concrete types come from packages written by different authors. For example, suppose we need to describe the digital media products sold by an online store:

```swift
protocol Artifact {
    var title: String { get }
    var creators: [String] { get }
    var created: Date { get }
}

protocol Text {
    var pages: Int { get }
    var words: Int { get }
    var pageSize: Int { get }
}

protocol Audio {
    func stream() -> AsyncStream<[Float]>
    var runningTime: Duration { get }
    var format: String { get }  // e.g., "MP3", "WAV"
}

protocol Video {
    func stream() -> AsyncStream<[Float]>
    var runningTime: Duration { get }
    var format: String { get }  // e.g., "MP4", "WMV"
    var resolution: (x: Int, y: Int) { get }
}
```

Suppose the store's `Song` type conforms to `Audio`, and its `Movie` type to `Video`. If we find we need to process songs and movies in the same way, we can define a `Streamer` protocol to represent their common aspects, and conform the existing types to it with extensions, without changing any of their declarations:

```swift
protocol Streamer {
    func stream() -> AsyncStream<[Float]>
    var runningTime: Duration { get }
    var format: String { get }
}

extension Movie: Streamer {}
extension Song: Streamer {}
```

The extensions are empty, because each type already has the required members.

## 7.4. Parsing Flags with a Protocol

In this section, we'll see how a protocol lets a library work with types it has never heard of. In Section 2.3.2, we used the `ArgumentParser` package to parse command-line flags. It knows how to parse `String`, `Int`, `Double`, `Bool`, and a handful of other types, but it can parse a value of *any* type that conforms to its protocol `ExpressibleByArgument`:

```swift
protocol ExpressibleByArgument {
    /// Creates a new instance of this type from a command-line-specified argument.
    init?(argument: String)
    // ...and a few other requirements with default implementations...
}
```

The initializer is *failable*, written `init?`: it may return `nil` if the argument string can't be converted to the type, in which case `ArgumentParser` reports a usage error.

To begin, consider this program, which sleeps for a specified period of time:

```swift
// swiftpl/ch7/sleep
import ArgumentParser

@main
struct Sleep: AsyncParsableCommand {
    @Option(help: "sleep period")
    var period: Duration = .seconds(1)

    func run() async throws {
        print("Sleeping for \(period)...")
        try await Task.sleep(for: period)
    }
}
```

`Duration` isn't one of the types `ArgumentParser` knows about, so the program doesn't compile until we teach it how to parse a duration:

```swift
extension Duration: @retroactive ExpressibleByArgument {
    /// Parses a duration like "50ms", "1.5s", or "2m".
    public init?(argument: String) {
        let units: [(suffix: String, seconds: Double)] = [
            ("ms", 0.001), ("s", 1), ("m", 60), ("h", 3600),
        ]
        for (suffix, scale) in units where argument.hasSuffix(suffix) {
            guard let value = Double(argument.dropLast(suffix.count)) else {
                return nil
            }
            self = .seconds(value * scale)
            return
        }
        return nil
    }
}
```

(The order of the units matters, since `"50ms"` also ends with `"s"`; we check `"ms"` first.) With the conformance in place, the program works:

```
$ swift run sleep
Sleeping for 1.0 seconds...
$ swift run sleep --period 50ms
Sleeping for 0.05 seconds...
$ swift run sleep --period 2m
Sleeping for 120.0 seconds...
$ swift run sleep --period "1 day"
Error: The value '1 day' is invalid for '--period <period>'
Usage: sleep [--period <period>]
```

It's equally easy to define new flag types for our own data types. Here's a flag for the `Celsius` type from Section 2.6, which allows a temperature to be specified in either Celsius or Fahrenheit, such as `"100C"` or `"212F"`, with the conversion done in the initializer:

```swift
// swiftpl/ch7/tempflag
import ArgumentParser
import TempConv

extension Celsius: @retroactive ExpressibleByArgument {
    public init?(argument: String) {
        guard let unit = argument.last, let value = Double(argument.dropLast()) else {
            return nil
        }
        switch unit {
        case "C", "c":
            self.init(value)
        case "F", "f":
            self = fToC(Fahrenheit(value))
        default:
            return nil
        }
    }
}

@main
struct TempFlag: ParsableCommand {
    @Option(help: "the temperature")
    var temp = Celsius(20)

    func run() {
        print(temp)
    }
}
```

Here's a test session:

```
$ swift run tempflag
20.0°C
$ swift run tempflag --temp -18C
-18.0°C
$ swift run tempflag --temp 212F
100.0°C
$ swift run tempflag --temp 273.15K
Error: The value '273.15K' is invalid for '--temp <temp>'
```

`ArgumentParser` doesn't know anything about `Celsius`; it knows only that it can call `init?(argument:)` on any `ExpressibleByArgument` type. This is the same kind of decoupling that `print` gets from `TextOutputStream`, but in the other direction: a protocol requirement that's an initializer lets a library *create* values of types it has never seen.

**Exercise 7.5:** Add support for Kelvin temperatures to `tempflag`.

**Exercise 7.6:** `ExpressibleByArgument` has another requirement, `static var allValueStrings: [String]`, with a default of `[]`. Find out what it's for, and use it to make a flag that accepts the name of a `Weekday` (Section 3.6). Is there an easier way to do this for a `CaseIterable` enum?

## 7.5. Existential Values: `any` and `some`

Conceptually, a value of a protocol type such as `any TextOutputStream` has two components, a concrete type and a value of that type. These are called the protocol value's *dynamic type* and *dynamic value*. Swift calls a value of protocol type an *existential*, from logic: it says "there exists some type `T` conforming to `TextOutputStream`, and this is a value of `T`." The keyword `any` marks an existential type.

In the statements below, the variable `w` takes on three different values, each with its own dynamic type:

```swift
var w: any TextOutputStream = ""
w = ByteCounter()
w = FileStream.standardError  // suppose this is a class conforming to TextOutputStream
```

Internally, an existential is stored in an *existential container*, a fixed-size box that holds the dynamic type's metadata, a table of its conformance (the *witness table*, which lists the implementations of each requirement), and the value itself. Small values (up to three words) are stored inline in the container; larger values are boxed on the heap. Calling a method on an existential looks up the implementation in the witness table and calls it with the stored value, which is called *dynamic dispatch*.

Existentials are flexible: an `[any Shape]` can hold circles, squares, and triangles together, and the dynamic type of each element can be different. But that flexibility has costs. Dynamic dispatch prevents inlining, larger values may need heap allocation, and, most important, information about the type is erased. Once a value is in an `any Equatable` box, for example, you can't compare it with another `any Equatable`, because the two might have different dynamic types:

```swift
let a: any Equatable = 1
let b: any Equatable = "one"
print(a == b)  // compile error: binary operator '==' cannot be applied to two 'any Equatable' operands
```

### 7.5.1. Opaque Types and `some`

The alternative to an existential is a *generic* parameter, which Swift lets you write concisely with the keyword `some`:

```swift
func printTwice(_ x: some CustomStringConvertible) {
    print(x.description, x.description)
}
```

This looks almost the same as a function taking `any CustomStringConvertible`, but it means something quite different. `some CustomStringConvertible` says that the function works for *some specific type* that conforms to the protocol, determined by each caller. The compiler knows the concrete type at each call site: `printTwice(42)` calls the function with `Int`, and `printTwice("hi")` with `String`. Nothing is boxed, and the optimizer can generate a specialized copy of the function for each type, with the method calls inlined. The `some` form is shorthand for an explicit generic parameter, which we'll discuss in Section 7.7:

```swift
func printTwice<T: CustomStringConvertible>(_ x: T) { ... }
```

`some` can also be used for a function's *result* type, where it's called an *opaque result type*. The function returns a value of one specific type, which the caller is not told:

```swift
func evens(upTo n: Int) -> some Sequence<Int> {
    stride(from: 0, to: n, by: 2)
}
```

The caller knows only that the result is a sequence of `Int`s; the function's author is free to change the implementation later to return a different type of sequence. But the compiler knows the actual type, so there's no boxing or dynamic dispatch.

The general advice, which the Swift compiler's own diagnostics encourage, is to write `some` by default and use `any` only when you need a value whose dynamic type can vary at run time, such as an element of a heterogeneous array.

### 7.5.2. Caveat: An Existential Containing `nil`

In Go, an interface value containing a nil pointer is itself non-nil, a notorious trap. Swift's optionals make the analogous situation explicit, but it can still surprise. An existential can't be `nil` (the type `any P` has no `nil` value), but `(any P)?` can. And because every type conforms to `Any`, *an optional can itself be stored in an `Any`*:

```swift
let maybeName: String? = nil
let x: Any = maybeName  // warning: expression implicitly coerced from 'String?' to 'Any'
print(x)  // "nil"
if x is String {
    print("a string")
} else {
    print("not a string")  // printed
}
```

The value `x` is not `nil`; it's an `Any` whose dynamic type is `Optional<String>` and whose dynamic value happens to be `.none`. The compiler warns about the implicit conversion, and the warning is worth heeding: you almost never want to put an optional into an `Any`. Write `maybeName as Any` if you mean it, or unwrap first.

## 7.6. Sorting with `Comparable`

Like string formatting, sorting is a frequently used operation in many programs. Swift's standard library sorts any mutable, random-access collection in place with `sort()`, or produces a sorted array from any sequence with `sorted()`. The algorithm is a hybrid merge sort (Timsort-like), and it's *stable*: equal elements keep their relative order.

How does `sort` know how to compare elements? The no-argument form works for elements that conform to `Comparable`, using `<`:

```swift
var names = ["Robert", "Ken", "Rob", "Ian"]
names.sort()
print(names)  // "["Ian", "Ken", "Rob", "Robert"]"
```

Go's `sort.Interface` requires a type to implement `Len`, `Less`, and `Swap` methods. Swift doesn't need that, because `sort` is defined in an extension of `MutableCollection` constrained to `RandomAccessCollection`: any such collection already knows its length and how to swap elements. All that remains is the comparison. For elements that aren't `Comparable`, or to sort in a different order, pass a function that implements "less than":

```swift
names.sort(by: >)  // descending
names.sort { $0.count < $1.count }  // by length
```

Let's work through a more substantial example: the playlist of a music player, displayed as a table. Each track is a row, and each column is an attribute of that track, like artist, title, and running time. Imagine that a graphical user interface presents the table, and that clicking the head of a column causes the playlist to be sorted by that attribute; clicking the same column head again reverses the order. Let's look at what might happen in response to each click.

The variable `tracks` below contains a playlist. Each element is a `Track`, as declared in Section 7.3.

```swift
// swiftpl/ch7/sorting
let tracks = [
    Track(title: "Fly", artist: "Delia", album: "Morning Light", year: 2012, length: length(3, 38)),
    Track(title: "Fly", artist: "Moby Grape", album: "Static", year: 1992, length: length(3, 37)),
    Track(title: "Fly Away", artist: "Alex K", album: "As I Am", year: 2007, length: length(4, 36)),
    Track(title: "Ready to Fly", artist: "M. Solveig", album: "Smash", year: 2011, length: length(4, 24)),
]

func length(_ minutes: Int, _ seconds: Int) -> Duration {
    .seconds(minutes * 60 + seconds)
}
```

(The artists and albums are invented for this example.) `Duration.seconds(_:)` is a static factory method that creates a duration from a number of seconds.

The `printTracks` function prints the playlist as a table, with padded columns:

```swift
func printTracks(_ tracks: [Track]) {
    func row(_ cols: String...) -> String {
        zip(cols, [16, 12, 16, 6, 8])
            .map { $0.padding(toLength: $1, withPad: " ", startingAt: 0) }
            .joined()
    }
    print(row("Title", "Artist", "Album", "Year", "Length"))
    print(row("-----", "------", "-----", "----", "------"))
    for t in tracks {
        print(row(t.title, t.artist, t.album, String(t.year), "\(t.length.formatted(.time(pattern: .minuteSecond)))"))
    }
}
```

`zip` pairs each column with its width, and Foundation's `padding(toLength:withPad:startingAt:)` pads or truncates each to the right width. To sort the playlist by the `artist` field:

```swift
printTracks(tracks.sorted { $0.artist < $1.artist })
```

```
Title           Artist      Album           Year  Length
-----           ------      -----           ----  ------
Fly Away        Alex K      As I Am         2007  4:36
Fly             Delia       Morning Light   2012  3:38
Ready to Fly    M. Solveig  Smash           2011  4:24
Fly             Moby Grape  Static          1992  3:37
```

If the user requests "sort by artist" a second time, we'll sort the tracks in reverse:

```swift
printTracks(tracks.sorted { $0.artist > $1.artist })
```

Reversing the comparison is simplest. When the order is reversed often, you might instead sort once and then iterate over `.reversed()`, which is a lazy view and costs nothing.

### 7.6.1. Sorting by Several Keys

Many sorts need several keys, consulted in order. For example, we might sort by title, then by year, then by running time, to break ties. Swift's tuples are `Comparable` lexicographically (for up to six `Comparable` elements), which makes multi-tier comparisons a one-liner:

```swift
printTracks(tracks.sorted {
    ($0.title, $0.year, $0.length) < ($1.title, $1.year, $1.length)
})
```

```
Title           Artist      Album           Year  Length
-----           ------      -----           ----  ------
Fly             Moby Grape  Static          1992  3:37
Fly             Delia       Morning Light   2012  3:38
Fly Away        Alex K      As I Am         2007  4:36
Ready to Fly    M. Solveig  Smash           2011  4:24
```

When the keys are chosen at run time, perhaps from a sequence of clicks on column headers, Foundation's `SortComparator` and `KeyPathComparator` let you represent each key as a value and sort with an array of them:

```swift
import Foundation

var order: [KeyPathComparator<Track>] = []
order.insert(KeyPathComparator(\.year), at: 0)  // user clicked "Year"
order.insert(KeyPathComparator(\.title), at: 0)  // then "Title"
printTracks(tracks.sorted(using: order))
```

Each comparator also has an `order` property, `.forward` or `.reverse`, which a user interface can toggle when a column header is clicked again.

### 7.6.2. Making a Type `Comparable`

When a type has a natural order, conform it to `Comparable` by implementing `<`. For example, we might want tracks ordered by artist and then title, by default:

```swift
extension Track: Comparable {
    static func < (a: Track, b: Track) -> Bool {
        (a.artist, a.title) < (b.artist, b.title)
    }
}

let sortedTracks = tracks.sorted()
let earliest = tracks.min()
```

Conforming to `Comparable` gives the type not only `sort()` and `sorted()` but also `min()`, `max()`, `<=`, `>`, `>=`, range formation with `...` and `..<`, and the `clamped(to:)` method, all from protocol extensions.

The `Comparable` protocol requires that `<` be a *strict weak ordering*: irreflexive (`a < a` is false), transitive, and with incomparability transitive too. The sort algorithm relies on these properties. A broken comparison won't crash the program, but it can produce results that aren't sorted.

Checking whether a sequence is already sorted is easy with `zip`:

```swift
func isSorted<T: Comparable>(_ values: [T]) -> Bool {
    zip(values, values.dropFirst()).allSatisfy { $0 <= $1 }
}
print(isSorted([1, 2, 2, 5]))  // "true"
```

**Exercise 7.7:** Extend the `Duration` parser from Section 7.4 to accept compound durations like `"3m38s"` and `"1h30m"`.

**Exercise 7.8:** Many GUIs provide a table widget with a stateful multi-tier sort: the primary sort key is the most recently clicked column head, the secondary sort key is the second-most recently clicked column head, and so on. Define an implementation of this using `[KeyPathComparator<Track>]`, in which clicking a column moves its comparator to the front and clicking it twice in a row reverses its order. Compare that approach with repeated stable sorting by each key, in reverse order.

**Exercise 7.9:** Use the `HTML` type from Section 4.6 to produce an HTML table of the tracks in which clicking a column header makes an HTTP request to sort the table by that column. Serve it with Hummingbird.

## 7.7. Generics and Associated Types

Go's original design omitted generics, and Go programs used interfaces where other languages would use generic code; generics arrived in Go 1.18, years after the Go book was written. In Swift, generics have been central from the beginning, and protocols and generics are two halves of a single design. This section introduces the essentials.

### 7.7.1. Generic Functions

A *generic function* has one or more *type parameters*, written in angle brackets after the function's name. Each type parameter can be *constrained* to conform to one or more protocols:

```swift
func largest<T: Comparable>(_ values: [T]) -> T? {
    guard var best = values.first else { return nil }
    for v in values.dropFirst() where v > best {
        best = v
    }
    return best
}

print(largest([3, 1, 4, 1, 5]) as Any)  // "Optional(5)"
print(largest(["fig", "apple", "kiwi"]) as Any)  // "Optional("kiwi")"
```

The type parameter `T` stands for whatever element type the caller passes, and the constraint `T: Comparable` lets the body use `>` on values of type `T`. At each call site, the compiler infers `T` from the argument (`Int` in the first call, `String` in the second) and checks that it satisfies the constraint. Inside the function, *only* the operations guaranteed by the constraints are available; the body is type-checked once, against the constraints, not separately for each use. That's the essential difference between Swift generics and C++ templates.

Constraints that are too complicated for the angle brackets go in a `where` clause:

```swift
func allEqual<C: Collection>(_ c: C) -> Bool where C.Element: Equatable {
    guard let first = c.first else { return true }
    return c.allSatisfy { $0 == first }
}
```

At run time, generic code can work in one of two ways. Without optimization, there is one compiled copy of the function, and it receives a hidden parameter describing the type `T` and its conformances, through which it calls the right `<` or `==`. With optimization, the compiler usually *specializes* the function, generating a separate copy for each concrete type it's used with, so generic code runs just as fast as code written by hand for each type. The library author doesn't have to choose; the compiler decides.

### 7.7.2. Generic Types

Types can be generic too. All of the standard collections are: `Array<Element>`, `Dictionary<Key, Value>`, `Set<Element>`, `Optional<Wrapped>`. Here is a simple stack:

```swift
// swiftpl/ch7/stack
struct Stack<Element> {
    private var items: [Element] = []

    var isEmpty: Bool { items.isEmpty }

    mutating func push(_ item: Element) {
        items.append(item)
    }

    mutating func pop() -> Element? {
        items.popLast()
    }
}

var s = Stack<String>()
s.push("a")
s.push("b")
print(s.pop() ?? "empty")  // "b"
```

An extension of a generic type can add members only for certain type arguments; this is called a *conditional conformance* when it adds a protocol conformance:

```swift
extension Stack: Equatable where Element: Equatable {}  // == is synthesized
extension Stack: Sendable where Element: Sendable {}
```

Now two stacks of strings can be compared with `==`, but a stack of closures, which aren't `Equatable`, can't. The standard library uses conditional conformance throughout: `Array` is `Equatable` when its elements are, `Hashable` when its elements are, `Codable` when its elements are, and so on.

### 7.7.3. Associated Types

Some protocols need to talk about *related* types. A `Sequence`, for instance, has an element type, but the element type is different for each conforming type: `Int` for `[Int]`, `Character` for `String`, `(key: K, value: V)` for a dictionary. A protocol declares such a type with an `associatedtype` requirement:

```swift
protocol IteratorProtocol {
    associatedtype Element
    mutating func next() -> Element?
}

protocol Sequence {
    associatedtype Element
    associatedtype Iterator: IteratorProtocol where Iterator.Element == Element
    func makeIterator() -> Iterator
    // ...plus many requirements with default implementations
}
```

An iterator produces the elements of a sequence one at a time, returning `nil` at the end. A `for`-`in` loop is shorthand for calling `makeIterator()` and then `next()` until it returns `nil`:

```swift
for x in sequence {
    print(x)
}
// is equivalent to:
var it = sequence.makeIterator()
while let x = it.next() {
    print(x)
}
```

Let's make a sequence of our own: a countdown from some number to 1.

```swift
// swiftpl/ch7/countdown
struct Countdown: Sequence, IteratorProtocol {
    var count: Int

    mutating func next() -> Int? {
        guard count > 0 else { return nil }
        defer { count -= 1 }
        return count
    }
}

for n in Countdown(count: 3) {
    print(n, terminator: " ")
}
print("Liftoff!")  // "3 2 1 Liftoff!"
```

The compiler infers the associated type `Element` as `Int` from the return type of `next()`. Because the type is its own iterator, the default implementation of `makeIterator()` returns `self`. And now that `Countdown` conforms to `Sequence`, it gets every sequence algorithm for free:

```swift
let c = Countdown(count: 5)
print(c.map { $0 * $0 })  // "[25, 16, 9, 4, 1]"
print(c.filter { $0.isMultiple(of: 2) })  // "[4, 2]"
print(c.reduce(0, +))  // "15"
print(c.contains(3))  // "true"
print(Array(c.prefix(2)))  // "[5, 4]"
```

A protocol with associated types can name its most important ones as *primary associated types* in angle brackets: `protocol Sequence<Element>`, `protocol Collection<Element>`. That's what lets us write constrained `some` and `any` types like `some Sequence<Int>`, which we used for `evens(upTo:)` in Section 7.5, and `any Collection<String>`.

### 7.7.4. Generics versus Existentials

We now have two ways to write a function that works with many types: with a protocol existential, `any P`, or with a generic type parameter, `T: P` or `some P`. How should you choose?

Generic code preserves type information. In `largest<T: Comparable>`, the result is a `T`, so a caller passing `[Int]` gets back an `Int?`, not an `any Comparable`. Generic code can relate several values to each other (two `T`s can be compared with `<`), and it's specialized and inlined by the optimizer. Prefer it in most cases.

Existential code erases type information, which is useful precisely when you need to store or pass values whose types vary at run time: a heterogeneous array of shapes, a property that can hold any `Error`, a plugin registry. Existential values have a uniform size and representation, so they can be stored side by side regardless of the dynamic type.

In practice, most Swift code uses concrete types most of the time, generics when it's abstracting over algorithms, and existentials at the boundaries where truly dynamic values occur, such as `any Error`.

**Exercise 7.10:** Write a generic function `isPalindrome` that takes any `BidirectionalCollection` whose elements are `Equatable` and reports whether it reads the same forwards and backwards. Test it with arrays, strings, and a string's `utf8` view.

**Exercise 7.11:** Make the `IntSet` from Section 6.5 conform to `Sequence` by writing an iterator type.

**Exercise 7.12:** Write a generic `struct LRUCache<Key: Hashable, Value>` with a fixed capacity, and methods to get and set values.

## 7.8. The `Error` Protocol

Since the beginning of this book, we've been throwing and catching errors without saying much about what an error is. As we saw in Section 7.2, `Error` is a protocol with no requirements (other than `Sendable`, which we'll discuss in Chapter 9):

```swift
protocol Error: Sendable {}
```

Any type can be an error by conforming, and enums, structs, and classes all are, in different libraries. A function declared with plain `throws` can throw a value of any `Error` type, so a caught error has the existential type `any Error`. Its dynamic type might be an enum from your own module, a `DecodingError` from the standard library, a `URLError` or `CocoaError` from Foundation, or a `CancellationError` from the concurrency library.

The simplest possible error is a struct carrying a message, much like Go's `errors.New`:

```swift
struct SimpleError: Error, CustomStringConvertible {
    let description: String
    init(_ description: String) { self.description = description }
}

throw SimpleError("something went wrong")
```

It's tempting to define one such general-purpose error type and use it everywhere, but it's better to define error types that describe the failures of each module: an enum whose cases name the ways an operation can fail, or a struct that carries the details of one kind of failure. Callers can then make decisions based on the error's type and contents, as we'll see in Section 7.11.

Errors are not `Equatable` by default, so you can't compare two `any Error` values. An enum error without associated values, or whose associated values are `Equatable`, can be made `Equatable` simply by declaring the conformance, which is convenient for tests:

```swift
enum ParseError: Error, Equatable {
    case unexpectedEnd
    case unexpectedCharacter(Character, at: Int)
}

func parseNumber(_ s: String) throws(ParseError) -> Int {
    guard let first = s.first else { throw .unexpectedEnd }
    guard let n = Int(s) else { throw .unexpectedCharacter(first, at: 0) }
    return n
}

#expect(throws: ParseError.unexpectedEnd) { try parseNumber("") }  // Swift Testing (Chapter 11)
```

At the lowest level, system calls report failures with an integer error code, `errno`. The `swift-system` package provides a type, `Errno`, that wraps these codes and conforms to `Error`, with names like `.noSuchFileOrDirectory` and `.permissionDenied`, and Foundation's `POSIXError` does the same.

## 7.9. Example: Expression Evaluator

In this section, we'll build an evaluator for simple arithmetic expressions. We'll use an enum, `Expr`, to represent any expression in this language. Unlike Go, which would represent the different kinds of expression as different struct types satisfying a common interface, Swift's natural representation for a closed set of alternatives is an enum with associated values. For now, the enum needs no methods, but we'll add some later.

Our expression language consists of floating-point literals; the binary operators `+`, `-`, `*`, and `/`; the unary operators `-x` and `+x`; function calls `pow(x, y)`, `sin(x)`, and `sqrt(x)`; variables such as `x` and `pi`; and of course parentheses and standard operator precedence. All values are of type `Double`. Here's the type:

```swift
// swiftpl/ch7/eval
/// An Expr is an arithmetic expression.
indirect enum Expr: Equatable {
    case literal(Double)  // a numeric constant, e.g., 3.141
    case variable(String)  // a variable, e.g., x
    case unary(Character, Expr)  // a unary operator expression, e.g., -x; op is '+' or '-'
    case binary(Character, Expr, Expr)  // a binary operator expression, e.g., x+y; op is '+', '-', '*', or '/'
    case call(String, [Expr])  // a function call expression, e.g., sin(x); fn is "pow", "sin", or "sqrt"
}
```

To evaluate an expression containing variables, we'll need an *environment* that maps variable names to values:

```swift
typealias Env = [String: Double]
```

We'll also need an `eval` method that returns the value of the expression in a given environment. Since every expression must provide this method, and the set of kinds of expression is fixed, a single method with a `switch` over all the cases is the most direct implementation:

```swift
import Foundation

extension Expr {
    func eval(_ env: Env) -> Double {
        switch self {
        case .literal(let value):
            return value
        case .variable(let name):
            return env[name] ?? 0
        case .unary(let op, let x):
            switch op {
            case "+": return +x.eval(env)
            case "-": return -x.eval(env)
            default: fatalError("unsupported unary operator: \(op)")
            }
        case .binary(let op, let x, let y):
            switch op {
            case "+": return x.eval(env) + y.eval(env)
            case "-": return x.eval(env) - y.eval(env)
            case "*": return x.eval(env) * y.eval(env)
            case "/": return x.eval(env) / y.eval(env)
            default: fatalError("unsupported binary operator: \(op)")
            }
        case .call(let fn, let args):
            switch fn {
            case "pow": return pow(args[0].eval(env), args[1].eval(env))
            case "sin": return sin(args[0].eval(env))
            case "sqrt": return sqrt(args[0].eval(env))
            default: fatalError("unsupported function call: \(fn)")
            }
        }
    }
}
```

The evaluation of a literal simply returns its value. A variable is looked up in the environment, and a variable that isn't defined evaluates to zero. The unary and binary cases recursively evaluate their operands, then apply the operator. We don't consider division by zero or infinity to be errors, since they produce a result, albeit non-finite. Finally, a call evaluates the arguments to the `pow`, `sin`, or `sqrt` function and then calls the corresponding function from the math library.

Several of these cases can fail, if an `Expr` has an unknown operator or function, or a call has the wrong number of arguments. Since these are errors in the expression, not in the evaluator, we'll check for them once, ahead of time, in a separate method, so that `eval` can safely trap if a bad expression slips past.

First, though, we need a parser to turn strings into expressions. Here's a compact recursive-descent parser, with a lexer that splits the input into tokens:

```swift
// swiftpl/ch7/eval
struct SyntaxError: Error, CustomStringConvertible {
    var description: String
}

enum Token: Equatable {
    case number(Double)
    case ident(String)
    case symbol(Character)
    case eof
}

/// Parses the input string as an arithmetic expression.
///
///     expr = num                         a literal number, e.g., 3.14159
///          | id                          a variable name, e.g., x
///          | id '(' expr ',' ... ')'     a function call
///          | '-' expr                    a unary operator (+-)
///          | expr '+' expr               a binary operator (+-*/)
func parse(_ input: String) throws(SyntaxError) -> Expr {
    var p = try Parser(input)
    let e = try p.parseBinary(minPrecedence: 1)
    guard p.token == .eof else {
        throw SyntaxError(description: "unexpected \(p.token)")
    }
    return e
}

struct Parser {
    private let chars: [Character]
    private var pos = 0
    private(set) var token = Token.eof

    init(_ input: String) throws(SyntaxError) {
        chars = Array(input)
        try next()
    }

    /// Advances to the next token.
    mutating func next() throws(SyntaxError) {
        while pos < chars.count, chars[pos].isWhitespace {
            pos += 1
        }
        guard pos < chars.count else {
            token = .eof
            return
        }
        let start = pos
        let c = chars[pos]
        if c.isNumber || c == "." {
            while pos < chars.count, chars[pos].isNumber || chars[pos] == "." {
                pos += 1
            }
            let text = String(chars[start..<pos])
            guard let value = Double(text) else {
                throw SyntaxError(description: "invalid number \(text)")
            }
            token = .number(value)
        } else if c.isLetter {
            while pos < chars.count, chars[pos].isLetter || chars[pos].isNumber {
                pos += 1
            }
            token = .ident(String(chars[start..<pos]))
        } else {
            pos += 1
            token = .symbol(c)
        }
    }

    private func precedence(_ t: Token) -> Int {
        switch t {
        case .symbol("*"), .symbol("/"): 2
        case .symbol("+"), .symbol("-"): 1
        default: 0
        }
    }

    /// Parses a sequence of binary operators whose precedence is at least minPrecedence.
    mutating func parseBinary(minPrecedence: Int) throws(SyntaxError) -> Expr {
        var lhs = try parseUnary()
        while case .symbol(let op) = token, precedence(token) >= minPrecedence {
            let prec = precedence(token)
            try next()
            let rhs = try parseBinary(minPrecedence: prec + 1)  // left-associative
            lhs = .binary(op, lhs, rhs)
        }
        return lhs
    }

    mutating func parseUnary() throws(SyntaxError) -> Expr {
        if case .symbol(let op) = token, op == "+" || op == "-" {
            try next()
            return .unary(op, try parseUnary())
        }
        return try parsePrimary()
    }

    mutating func parsePrimary() throws(SyntaxError) -> Expr {
        switch token {
        case .number(let value):
            try next()
            return .literal(value)

        case .ident(let id):
            try next()
            guard token == .symbol("(") else {
                return .variable(id)
            }
            try next()  // consume '('
            var args: [Expr] = []
            if token != .symbol(")") {
                while true {
                    args.append(try parseBinary(minPrecedence: 1))
                    if token == .symbol(",") {
                        try next()
                        continue
                    }
                    guard token == .symbol(")") else {
                        throw SyntaxError(description: "got \(token), want ')'")
                    }
                    break
                }
            }
            try next()  // consume ')'
            return .call(id, args)

        case .symbol("("):
            try next()
            let e = try parseBinary(minPrecedence: 1)
            guard token == .symbol(")") else {
                throw SyntaxError(description: "got \(token), want ')'")
            }
            try next()  // consume ')'
            return e

        default:
            throw SyntaxError(description: "unexpected \(token)")
        }
    }
}
```

The parser is a typical recursive-descent design. `parseBinary` handles binary operators by *precedence climbing*: it parses a unary expression, then, as long as the next token is an operator with sufficient precedence, consumes it and parses a right-hand side whose operators must bind more tightly. So `1 + 2 * 3` parses as `1 + (2 * 3)`, and `1 - 2 - 3` as `(1 - 2) - 3`. Every parsing function uses typed throws, since the only possible failure is a `SyntaxError`.

Notice the patterns in `precedence` and `parsePrimary`. A case like `.symbol("*")` matches a token that is a symbol whose character is `*`, and patterns can be nested to any depth.

Now let's test the evaluator. The program below evaluates a few expressions in several environments:

```swift
let tests: [(expr: String, env: Env, want: String)] = [
    ("sqrt(A / pi)", ["A": 87616, "pi": .pi], "167"),
    ("pow(x, 3) + pow(y, 3)", ["x": 12, "y": 1], "1729"),
    ("pow(x, 3) + pow(y, 3)", ["x": 9, "y": 10], "1729"),
    ("5 / 9 * (F - 32)", ["F": -40], "-40"),
    ("5 / 9 * (F - 32)", ["F": 32], "0"),
    ("5 / 9 * (F - 32)", ["F": 212], "100"),
]

for test in tests {
    let expr = try parse(test.expr)
    let got = String(format: "%.6g", expr.eval(test.env))
    print("\(test.expr)\t\(test.env)\t=> \(got)", got == test.want ? "" : "(want \(test.want))")
}
```

For each entry in the table, the program parses the expression, evaluates it in the environment, and prints the result. The output (with the environments abbreviated) shows that all the results are as expected:

```
sqrt(A / pi)            ["A": 87616.0, "pi": 3.141592653589793]   => 167
pow(x, 3) + pow(y, 3)   ["x": 12.0, "y": 1.0]                     => 1729
pow(x, 3) + pow(y, 3)   ["x": 9.0, "y": 10.0]                     => 1729
5 / 9 * (F - 32)        ["F": -40.0]                              => -40
5 / 9 * (F - 32)        ["F": 32.0]                               => 0
5 / 9 * (F - 32)        ["F": 212.0]                              => 100
```

In Chapter 11 we'll turn tables like this into proper unit tests.

So far, all our inputs have been well formed, but we might not be so lucky in the future. The parser rejects syntax errors, but an expression like `log(10)` or `sqrt(1, 2)` is syntactically valid yet would make `eval` trap. So we'll add a `check` method that reports such *static* errors, and also collects the set of variables the expression uses, which is useful for validating the environment:

```swift
extension Expr {
    static let arity = ["pow": 2, "sin": 1, "sqrt": 1]

    /// Reports errors in the expression and collects the set of variables it uses.
    func check(_ vars: inout Set<String>) throws(SyntaxError) {
        switch self {
        case .literal:
            return
        case .variable(let name):
            vars.insert(name)
        case .unary(let op, let x):
            guard "+-".contains(op) else {
                throw SyntaxError(description: "unexpected unary op \(op)")
            }
            try x.check(&vars)
        case .binary(let op, let x, let y):
            guard "+-*/".contains(op) else {
                throw SyntaxError(description: "unexpected binary op \(op)")
            }
            try x.check(&vars)
            try y.check(&vars)
        case .call(let fn, let args):
            guard let arity = Self.arity[fn] else {
                throw SyntaxError(description: "unknown function \"\(fn)\"")
            }
            guard args.count == arity else {
                throw SyntaxError(
                    description: "call to \(fn) has \(args.count) args, want \(arity)")
            }
            for arg in args {
                try arg.check(&vars)
            }
        }
    }
}
```

Here are some faulty inputs and the errors they elicit, some from `parse` and some from `check`:

```
x % 2               unexpected symbol("%")
math.Pi             invalid number .
!true               unexpected symbol("!")
"hello"             unexpected symbol("\"")
log(10)             unknown function "log"
sqrt(1, 2)          call to sqrt has 2 args, want 1
```

Separating checking from evaluation means that an expression can be checked once and then evaluated many times. That's exactly what we'll do next: we'll use the evaluator to plot arbitrary functions with the `surface` program from Section 3.2. The program takes an expression on the command line, parses and checks it, and then plots it, using an environment that binds `x` and `y` to the coordinates of each point and `r` to the distance from the origin:

```swift
// swiftpl/ch7/surface
// Surface plots the function given on the command line as an SVG surface.
import Foundation

guard CommandLine.arguments.count == 2 else {
    printError("usage: surface 'expression in x, y, and r'")
    exit(2)
}
let expr = parseAndCheck(CommandLine.arguments[1])
plot { x, y in
    expr.eval(["x": x, "y": y, "r": hypot(x, y)])
}

/// Parses and checks the expression, exiting the program if it is invalid.
func parseAndCheck(_ input: String) -> Expr {
    do {
        let expr = try parse(input)
        var vars = Set<String>()
        try expr.check(&vars)
        for v in vars where !["x", "y", "r"].contains(v) {
            throw SyntaxError(description: "undefined variable: \(v)")
        }
        return expr
    } catch {
        printError("bad expr: \(error)")
        exit(1)
    }
}
```

Here `plot` is the main loop of the original `surface` program, refactored into a function that takes the surface function as a parameter, `(Double, Double) -> Double`. We pass it a closure that evaluates the parsed expression. Because `check` has already validated the expression, `eval` won't trap.

```
$ swift run -c release surface 'sin(-x) * pow(1.5, -r)' > ripple.svg
$ swift run -c release surface 'pow(2, sin(y)) * pow(2, sin(x)) / 12' > eggbox.svg
$ swift run -c release surface 'sin(x * y / 10) / 10' > saddle.svg
```

**Exercise 7.13:** Add a `description` property to `Expr` that pretty-prints the syntax tree. Check that the results, when parsed again, yield an equivalent tree.

**Exercise 7.14:** Add a new case to `Expr`, such as `min(_:_:)`-style variadic calls or a ternary `?:` operator, and update `parse`, `eval`, `check`, and `description`. How does the compiler help you find all the places that must change?

**Exercise 7.15:** Write a program that reads a single expression from the standard input, prompts the user to provide values for any variables, then evaluates the expression in the resulting environment. Handle all errors gracefully.

**Exercise 7.16:** Write a web-based calculator program.

### 7.9.1. Enums or Protocols?

We implemented `Expr` as an enum, with each operation (`eval`, `check`) written as a `switch` over all the cases. Go's version of this program does the opposite: each kind of expression is a separate type, and each operation is a method implemented separately by each type, called through an interface. Swift could do it that way too:

```swift
protocol ExprNode {
    func eval(_ env: Env) -> Double
    func check(_ vars: inout Set<String>) throws(SyntaxError)
}

struct Literal: ExprNode { ... }
struct Binary: ExprNode { var op: Character; var x, y: any ExprNode; ... }
```

The two designs have complementary strengths. With an enum, it's easy to add a new *operation*: write one new function with a `switch`, and the compiler checks that it handles every case. But adding a new *kind of expression* means touching every operation. With a protocol, it's easy to add a new *kind*, even from another module, by writing one new conforming type; but adding a new operation means touching every type. (This trade-off is sometimes called *the expression problem*.)

For a closed set of alternatives, like the syntax of a small language, Swift programmers usually choose enums, and the compiler's exhaustiveness checking makes them robust. For an open set, such as a plugin interface or a set of shapes that clients can extend, protocols are the right choice.

## 7.10. Type Casting

A *type cast* converts a value from one type to another type to which it's related. Swift has three cast operators.

The *upcast* operator `as` converts a value to a more general type, one that it's statically known to conform to or inherit from. It always succeeds, so it's usually implicit; it's needed only to disambiguate, as in `x as Any` or `1 as Double`.

The *conditional downcast* operator `as?` attempts to convert a value to a more specific type, and produces an optional: the converted value if the value's *dynamic* type matches, or `nil` if it doesn't. The *forced downcast* `as!` does the same but traps on failure. And the `is` operator just tests whether a cast would succeed.

There are two possibilities. First, if the target type is a concrete type, the cast checks whether the dynamic type of the value is that concrete type, or, for classes, a subclass of it:

```swift
let values: [Any] = [42, "hello", 3.14, [1, 2, 3]]
for v in values {
    if let i = v as? Int {
        print("an Int: \(i)")
    } else if let s = v as? String {
        print("a String with \(s.count) characters")
    } else {
        print("something else: \(type(of: v))")
    }
}
// an Int: 42
// a String with 5 characters
// something else: Double
// something else: Array<Int>
```

Second, if the target type is a protocol type, the cast checks whether the dynamic type of the value *conforms* to the protocol, and if so, produces an existential of the new protocol type. Casting to a protocol type is the way to find out, at run time, whether a value supports some behavior that its static type doesn't guarantee:

```swift
func describe(_ v: Any) -> String {
    if let c = v as? any CustomStringConvertible {
        return c.description
    }
    return "<\(type(of: v))>"
}
```

Swift's casts are richer than Go's type assertions. They look through optionals (casting an `Int?` holding 5 to `Int` succeeds), convert between bridged Foundation and Swift types on Apple platforms (`NSString` to `String`), and can cast collections element by element (`[Any]` to `[Int]` succeeds if every element is an `Int`). The forced form `as!` should be used only when failure would indicate a bug.

## 7.11. Discriminating Errors with Casts

Consider the set of errors returned by file operations in Foundation. I/O can fail for any number of reasons, but three kinds of failure often must be handled differently: file already exists (for create operations), file not found (for read operations), and permission denied. Foundation's file APIs report errors of type `CocoaError`, a struct with a `code` property that identifies the kind of failure. Since a thrown error has static type `any Error`, discriminating among them requires a cast, either with `as?` or in a `catch` pattern:

```swift
do {
    let data = try Data(contentsOf: URL(filePath: "/no/such/file"))
    use(data)
} catch let error as CocoaError where error.code == .fileReadNoSuchFile {
    print("not found: \(error.filePath ?? "?")")
} catch let error as CocoaError where error.code == .fileReadNoPermission {
    print("permission denied")
} catch {
    print("other error: \(error)")
}
```

A `catch` clause pattern combines a cast (`let error as CocoaError`) with a condition (`where ...`), so each clause handles exactly the errors it's interested in.

When such tests are needed often, it helps to wrap them in a function. Here's a helper that recognizes "file not found" from several error domains, since different APIs report the same condition with different error types:

```swift
import Foundation
import SystemPackage

/// Reports whether the error indicates that a file does not exist.
func isNotExist(_ error: any Error) -> Bool {
    switch error {
    case let e as CocoaError:
        return e.code == .fileReadNoSuchFile || e.code == .fileNoSuchFile
    case let e as POSIXError:
        return e.code == .ENOENT
    case let e as Errno:
        return e == .noSuchFileOrDirectory
    default:
        return false
    }
}
```

The wrapping of errors described in Section 5.4 can defeat such tests, if an error is wrapped in another error, the cast to the original type fails. A well-designed wrapper error exposes its underlying error, as our `ParseFailure` did with its `underlying` property, so that a test like `isNotExist` can be applied to it. In general, the discrimination of errors should be done immediately after the failing operation, before an error is propagated to the caller.

## 7.12. Querying Behaviors with Conditional Casts

Sometimes a function can do its job more efficiently if a value supports some optional behavior. A cast to a protocol type lets the function ask.

For example, the standard library's `print` has to turn each argument into text. If the value's type conforms to `TextOutputStreamable`, a protocol for types that can write themselves directly to a stream, `print` asks it to do so, which avoids building an intermediate string. Otherwise, if the type conforms to `CustomStringConvertible`, `print` uses its `description`. Otherwise, it falls back on reflection (Chapter 12) to show the type's name and properties. We can write the same logic ourselves:

```swift
func write(_ value: Any, to out: inout some TextOutputStream) {
    switch value {
    case let v as any TextOutputStreamable:
        v.write(to: &out)  // fastest: write directly
    case let v as any CustomStringConvertible:
        out.write(v.description)
    default:
        out.write(String(reflecting: value))  // slowest: uses reflection
    }
}
```

The same technique lets generic algorithms choose faster strategies for more capable types. A function that receives `some Sequence` might check whether it's actually a `RandomAccessCollection` so that it can jump directly to the middle:

```swift
func middle<S: Sequence>(_ s: S) -> S.Element? {
    if let c = s as? any RandomAccessCollection<S.Element> {
        return middleElement(of: c)  // O(1)
    }
    let all = Array(s)  // O(n)
    return all.isEmpty ? nil : all[all.count / 2]
}

func middleElement<C: RandomAccessCollection>(of c: C) -> C.Element? {
    c.isEmpty ? nil : c[c.index(c.startIndex, offsetBy: c.count / 2)]
}
```

Passing the existential `c` to the generic function `middleElement` *opens* it: the compiler unpacks the box and calls the function with the dynamic type as `C`.

In idiomatic Swift, however, such checks are less common than in Go, because the protocol hierarchy and generics usually let the compiler choose the right implementation statically. The standard library's algorithms are written as extensions on `Sequence`, `Collection`, `BidirectionalCollection`, and `RandomAccessCollection`, and when a protocol requirement has a more efficient implementation for a more capable type, the more specific implementation is chosen automatically. For example, `count` is O(*n*) for a general `Collection` but O(1) for an `Array`.

## 7.13. Type Switches

Protocols are used in two distinct styles. In the first style, exemplified by `TextOutputStream`, `Sequence`, and `Comparable`, a protocol's methods express the similarities of the concrete types that conform to it but hide the representation details and intrinsic operations of those concrete types. The emphasis is on the methods, not on the concrete types.

The second style exploits the ability of an `Any` or other existential to hold values of a variety of concrete types, and considers the protocol to be the *union* of those types. Casts are used to discriminate among these types dynamically and treat each case differently. In this style, the emphasis is on the concrete types that satisfy the protocol, not on its methods.

Swift's `switch` statement supports this second style directly, with *type-casting patterns*. Consider a function that formats a value as a literal for an SQL query, used to build queries with arguments of different types:

```swift
func sqlQuote(_ x: Any?) -> String {
    switch x {
    case nil:
        return "NULL"
    case let x as Int:
        return String(x)
    case let x as UInt:
        return String(x)
    case let x as Bool:
        return x ? "TRUE" : "FALSE"
    case let x as String:
        return "'" + x.replacing("'", with: "''") + "'"
    default:
        fatalError("unexpected type \(type(of: x!)): \(x!)")
    }
}
```

Each `case let x as T` pattern succeeds if the value's dynamic type is `T`, and binds `x` to the value with that type. As with an ordinary `switch`, cases are considered in order and, when a match is found, the case's body is executed. The `case nil` pattern matches if the optional is `nil`. A pattern of the form `case is T` tests the type without binding a value.

That said, a Swift programmer would rarely write `sqlQuote` this way, because Swift has a better tool for a closed set of alternatives: an enum with associated values.

```swift
enum SQLValue {
    case null
    case int(Int)
    case bool(Bool)
    case string(String)

    var quoted: String {
        switch self {
        case .null: "NULL"
        case .int(let x): String(x)
        case .bool(let x): x ? "TRUE" : "FALSE"
        case .string(let x): "'" + x.replacing("'", with: "''") + "'"
        }
    }
}
```

With the enum, the `default` case and its `fatalError` disappear, because the compiler knows that the `switch` is exhaustive. Callers can't pass an unsupported type, such as a `Date`, by accident; to support one, you add a case, and the compiler shows you every `switch` that must handle it. Go calls the `Any`-based approach a *discriminated union*; in Swift, the enum *is* a discriminated union, checked by the type system. Use `Any` with type switches for genuinely open-ended data, such as values decoded from an untyped format, and enums whenever you can enumerate the possibilities.

(Of course, real SQL code should never build queries by quoting values into strings; it should pass them as parameters to a prepared statement. The `HTML` type from Section 4.6 shows how the type system can enforce such disciplines.)

## 7.14. Example: Token-Based XML Decoding

Section 4.5 showed how to decode JSON documents into Swift data structures with the `Decodable` protocol. That approach works for XML, too, with third-party packages like XMLCoder, but sometimes we need to process a document without knowing its structure in advance, or one too large to load into memory at once. For that, Foundation provides `XMLParser`, a *streaming* parser that reads the input sequentially and reports each piece of the document (start tag, end tag, text) as it's found.

`XMLParser` reports these *events* by calling methods on a *delegate*, an object that conforms to the `XMLParserDelegate` protocol. The delegate pattern, inherited from Objective-C, is one of the most common uses of protocols in Apple's frameworks. All of the protocol's methods are optional, so a delegate implements only the events it cares about.

The `xmlselect` program below extracts and prints the text found beneath certain elements in an XML document tree. Using the delegate API, it can do its job in a single pass over the input without ever materializing the tree.

```swift
// swiftpl/ch7/xmlselect
// Xmlselect prints the text of selected elements of an XML document.
import Foundation
#if canImport(FoundationXML)
import FoundationXML
#endif

final class Selector: NSObject, XMLParserDelegate {
    let path: [String]
    var stack: [String] = []  // stack of element names

    init(path: [String]) {
        self.path = path
    }

    func parser(_ parser: XMLParser, didStartElement elementName: String,
                namespaceURI: String?, qualifiedName qName: String?,
                attributes attributeDict: [String: String] = [:]) {
        stack.append(elementName)  // push
    }

    func parser(_ parser: XMLParser, didEndElement elementName: String,
                namespaceURI: String?, qualifiedName qName: String?) {
        stack.removeLast()  // pop
    }

    func parser(_ parser: XMLParser, foundCharacters string: String) {
        if containsAll(stack, path) {
            print("\(stack.joined(separator: " ")): \(string)")
        }
    }
}

/// Reports whether x contains the elements of y, in order.
func containsAll(_ x: [String], _ y: [String]) -> Bool {
    var y = y[...]
    for name in x {
        if y.isEmpty { return true }
        if name == y.first { y = y.dropFirst() }
    }
    return y.isEmpty
}

let data = FileHandle.standardInput.readDataToEndOfFile()
let parser = XMLParser(data: data)
let selector = Selector(path: Array(CommandLine.arguments.dropFirst()))
parser.delegate = selector
if !parser.parse() {
    FileHandle.standardError.write(
        Data("xmlselect: \(parser.parserError.map { "\($0)" } ?? "parse failed")\n".utf8))
    exit(1)
}
```

On Linux, `XMLParser` lives in a separate module, `FoundationXML`, so that programs that don't need XML don't have to link the XML library, which is the same arrangement we saw for `FoundationNetworking`.

Each time the parser encounters a start tag, the delegate pushes the element's name onto the stack, and at each end tag it pops it. The stack thus always holds the sequence of names of the open elements. Each time text is found, the delegate prints it if the stack contains all the names listed on the command line, in the same order.

`Selector` must be a class, and a subclass of `NSObject`, because the delegate protocol comes from Objective-C, where every object descends from `NSObject`. The parser holds its delegate weakly, to avoid a reference cycle, so we keep our own strong reference in the constant `selector` while the parse runs.

The program below uses `xmlselect` to extract the text of the `h2` headings in the XML 1.1 specification, a well-formed XHTML document:

```
$ swift build
$ fetch https://www.w3.org/TR/2006/REC-xml11-20060816 | xmlselect div div h2
html body div div h2: 1 Introduction
html body div div h2: 2 Documents
html body div div h2: 3 Logical Structures
html body div div h2: 4 Physical Structures
html body div div h2: 5 Conformance
html body div div h2: 6 Notation
html body div div h2: A References
html body div div h2: B Definitions for Character Normalization
...
```

(Since the parser may report the text of a single element in several pieces, real code would accumulate the text until the end tag. Exercise 7.17 asks you to fix this.)

Notice the difference in style between this delegate API and a *pull* API like Go's `xml.Decoder.Token`. With a delegate, the parser is in control and calls our code. With a pull API, our code is in control and asks for the next token. Pull APIs are often easier to compose, and Swift's `AsyncSequence` (Chapter 8) is the modern Swift way to provide one: a parser could expose its events as an asynchronous sequence of an enum `XMLEvent`, which a client would consume with `for try await event in parser.events`.

**Exercise 7.17:** Modify `Selector` to accumulate the text of each element, printing it at the end tag, and to ignore whitespace-only text.

**Exercise 7.18:** Extend `xmlselect` so that elements may be selected not just by name, but by their attributes too, in the manner of CSS, so that, for instance, an element like `<div id="page" class="wide">` could be selected by a matching `id` or `class` as well as its name.

**Exercise 7.19:** Using `XMLParser`, write a program that reads an arbitrary XML document and constructs a tree of generic nodes that represents it. Nodes are of two kinds: text nodes and element nodes. Use an `indirect enum Node { case text(String); case element(name: String, attributes: [String: String], children: [Node]) }`.

## 7.15. A Few Words of Advice

When designing a new module, novice Swift programmers often start by creating a set of protocols and only later define the concrete types that conform to them. This approach can result in many protocols, each of which has only a single conformance. Don't do that. Such protocols are unnecessary abstractions; they also have a run-time cost when used as existentials. You can restrict which methods of a type or members of a struct are visible outside a module using access control (Section 6.6). Protocols are needed only when there are two or more concrete types that must be dealt with in a uniform way.

We make an exception to this rule when a protocol is satisfied by a single concrete type but that type cannot live in the same module as the protocol because of its dependencies. In that case, a protocol is a good way to decouple two modules. The same goes for testing: a protocol with a real implementation and a test double is a reasonable design, though in Swift, passing closures is often simpler still.

Because protocols are used in Swift only when they are satisfied by two or more types, they necessarily abstract away from the details of any particular implementation. The result is smaller protocols with fewer, simpler requirements, often just one as with `TextOutputStream` and `Error`. Small protocols are easier to satisfy when new types come along. A good rule of thumb for protocol design is *ask only for what you need*.

Prefer generics to existentials. A function that takes `some Sequence<Int>` is faster, and its type information is more precise, than one that takes `any Sequence<Int>`. Use `any` for storage of heterogeneous values and at the boundaries of the system, where dynamic types are unavoidable.

Prefer enums to protocols for closed sets of alternatives, and protocols to enums for open ones.

And finally, remember that not everything need be an object. Free functions have their place, as do structs with no protocol conformances at all. A concrete type that does one thing well is the best foundation for everything else.
