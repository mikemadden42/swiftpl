# 7. Protocols

A *protocol* describes what a type can do without saying what it is. It lists requirements, such as methods, properties, initializers, and operators, and any type that provides them can declare that it *conforms*. Code written against a protocol then works with every conforming type, including types written later by other people. Protocols are how Swift expresses abstraction, and they're everywhere in the standard library: `Equatable` and `Hashable` for comparison and hashing, `Sequence` and `Collection` for iteration, `Codable` for serialization, `Error` for things that can be thrown, `Sendable` for things that can be shared between tasks.

Swift's protocols go well beyond the "interfaces" of many languages. They can carry *default implementations* in extensions, so a type gets rich behavior from a few requirements. They can have *associated types*, which makes them generic. And they work hand in hand with *generics*, so that code written against a protocol is checked, and optimized, as precisely as code written for a single concrete type.

Conformance in Swift is *nominal*: a type conforms to a protocol only if it says so, in its declaration or in an extension. Because an extension can add a conformance to any type at any time, even a type from another module, this is less restrictive than it sounds, and the explicit declaration documents intent and lets the compiler check it.

This chapter begins with protocols as contracts between code that uses a capability and code that provides it. It then covers generics, the dynamic side of protocols (values of protocol type, casts, and type switches), and some advice on choosing among these tools.

## 7.1. Protocols as Contracts

Most of the types we've used are *concrete*: an `Int`, a `[String]`, a `RingBuffer<Double>`. A concrete type determines its values' representation and exactly which operations they support. A protocol is *abstract*: it says nothing about representation and promises only that certain operations exist. Holding a value through a protocol, you know what it can *do*, not what it *is*.

A good example is the way `print` writes its output. Besides the familiar form, which writes to the standard output, `print` has a form that writes anywhere:

```swift
func print(_ items: Any..., separator: String = " ", terminator: String = "\n",
           to output: inout some TextOutputStream)
```

Its destination is anything that conforms to `TextOutputStream`, a protocol with a single requirement:

```swift
protocol TextOutputStream {
    /// Appends the given string to the stream.
    mutating func write(_ string: String)
}
```

The protocol is a contract between `print` and its callers. Callers must supply a value whose type has a `write` method that behaves as documented. In return, `print` guarantees to produce its output entirely through that method, so it works with any destination at all, whether a file, a network connection, or a string in memory, without knowing which. Being able to substitute any conforming type wherever the protocol is expected is what makes a protocol useful.

`String` conforms to `TextOutputStream`, so printing into a string is built in:

```swift
var log = ""
print("started", 42, separator: ": ", to: &log)
print(log, terminator: "")  // "started: 42\n"
```

Let's write a conforming type of our own. A `Transcript` collects whatever is written to it as a list of complete lines, which is handy for testing code that prints (Section 11.2.7):

```swift
// swiftpl/ch7/transcript
struct Transcript: TextOutputStream {
    private(set) var lines: [String] = []
    private var partial = ""

    mutating func write(_ string: String) {
        for c in string {
            if c == "\n" {
                lines.append(partial)
                partial = ""
            } else {
                partial.append(c)
            }
        }
    }
}
```

`print` doesn't know anything about `Transcript`, and doesn't need to:

```swift
// swiftpl/ch7/transcript (continued)
var t = Transcript()
print("hello", to: &t)
print("a", "b", "c", separator: "-", to: &t)
print("no newline yet", terminator: "", to: &t)
print(t.lines)  // "["hello", "a-b-c"]"
```

The declaration `struct Transcript: TextOutputStream` states the conformance. The compiler checks that `Transcript` really provides everything the protocol requires, and reports an error if it doesn't. Notice that `print` may call `write` several times for one line, once for each item, separator, and terminator, so `Transcript` can't assume each call is a whole line; it buffers until it sees a newline.

Formatting values as text relies on another protocol, which we've used several times already:

```swift
protocol CustomStringConvertible {
    var description: String { get }
}
```

A requirement can be a property as well as a method. `{ get }` means conforming types must provide a readable property; a stored `let` or `var` or a computed property all qualify. `{ get set }` would require a writable one. We've conformed `Kilometers` (Section 2.5) and `RingBuffer` (Section 6.5) to it, among others.

**Exercise 7.1:** Write a `TextOutputStream` that counts the lines and words written to it, and use it to measure the output of a function that prints a report.

**Exercise 7.2:** Write a generic `struct Tee<A: TextOutputStream, B: TextOutputStream>: TextOutputStream` that writes everything to two streams at once.

**Exercise 7.3:** Make the `FileNode` type from Section 4.4 conform to `CustomStringConvertible`, with a `description` that shows the tree with indentation, as `printTree` does.

## 7.2. Protocol Types

A protocol declaration lists requirements. The standard library's protocols range from the minimal, like `Equatable`, with a single operator, to the substantial, like `Collection`, with a dozen requirements, most of which have default implementations. A few you'll meet constantly:

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

Inside a protocol, `Self` stands for the conforming type, so `Equatable` requires an `==` that compares two `Int`s for `Int` and two `String`s for `String`.

A protocol can *inherit* from others. `Hashable: Equatable` means that every `Hashable` type must be `Equatable` as well, and meet both sets of requirements. Protocols can also be *composed* on the spot with `&`, without declaring a new protocol; that's exactly how the standard library defines `Codable`:

```swift
typealias Codable = Decodable & Encodable

func cache(_ value: some Codable & Sendable) { ... }
```

Requirements come in many forms. Here's a protocol that uses most of them:

```swift
protocol Shape {
    init(boundingBox: Rect)  // an initializer
    var area: Double { get }  // a read-only property
    var label: String { get set }  // a read-write property
    func contains(_ p: Point) -> Bool  // a method
    mutating func scale(by factor: Double)  // a mutating method
    static var unit: Self { get }  // a static property
    subscript(vertex i: Int) -> Point { get }  // a subscript
}
```

A struct satisfies a `mutating` requirement with a mutating method, and a class with an ordinary method. A protocol that only classes should adopt says so by inheriting from `AnyObject`, which lets values of the protocol type be held by `weak` references:

```swift
protocol DownloadDelegate: AnyObject {
    func downloadDidFinish(_ data: [UInt8])
}

final class Downloader {
    weak var delegate: (any DownloadDelegate)?
}
```

**Exercise 7.4:** Define a protocol `Stack` with an associated type `Element` and requirements `push(_:)`, `pop()`, and `isEmpty`, and make `Array` conform to it with an extension.

## 7.3. Protocol Conformance

A type *conforms* to a protocol when it declares the conformance and meets the requirements. We often say a type "is a" protocol: a `String` is a `TextOutputStream`, an `[Int]` is a `Collection`.

Conformances can be declared with the type or added later in extensions, and Swift style favors one extension per conformance, which keeps related code together:

```swift
struct Finisher {
    var name: String
    var country: String
    var time: Duration
}

extension Finisher: CustomStringConvertible {
    var description: String { "\(name) (\(country))" }
}

extension Finisher: Equatable {}  // == is synthesized from the stored properties
```

Because extensions can be added anywhere, conformances can be added after the fact, by anyone: your types can adopt the standard library's protocols, and the standard library's types can adopt yours. That gives much of the flexibility of languages where conformance is implicit, while keeping each conformance explicit and checked.

There's one hazard. If two modules that you don't own both add the same conformance (say, of a Foundation type to a protocol from some package), they may disagree, and the program can't use both. Swift 6 warns about such *retroactive* conformances; when you're sure, acknowledge the risk with `@retroactive`:

```swift
import Foundation
import ArgumentParser

extension URL: @retroactive ExpressibleByArgument { ... }
```

A value can be assigned to a variable of protocol type only if its type conforms:

```swift
var out: any TextOutputStream
out = ""  // OK: String conforms
out = Transcript()  // OK: Transcript conforms
out = 42  // compile error: 'Int' does not conform to 'TextOutputStream'
```

And through a protocol type, only the protocol's members are visible:

```swift
var stream: any TextOutputStream = Transcript()
stream.write("hello\n")  // OK: write is a requirement
print(stream.lines)  // compile error: value of type 'any TextOutputStream' has no member 'lines'
```

The more a protocol requires, the more it tells you about its values and the fewer types can conform. `Any`, the protocol composition with no protocols at all, requires nothing, so every type conforms to it, and that's why `print` can take arguments of type `Any`. But with an `Any` you can do almost nothing until you find out what's inside, which is what casts are for (Section 7.10).

Requirements can be met by a type's own members, by members inherited from a superclass, or by *default implementations* in an extension of the protocol. Defaults are what make rich protocols practical. `Sequence` requires just `makeIterator()`, and its extensions supply `map`, `filter`, `reduce`, `contains`, `min`, `max`, `sorted`, `first(where:)`, `enumerated()`, `joined`, and many more. We'll write a sequence of our own in Section 7.7.

### 7.3.1. New Abstractions for Existing Types

Because conformance can be added after the fact, you can define a protocol whenever you notice that existing types share a role, without touching their declarations. Consider a home-automation program with types for its devices, each conforming to protocols for its capabilities:

```swift
// swiftpl/ch7/home
protocol Switchable {
    var isOn: Bool { get set }
}

protocol Dimmable {
    var brightness: Double { get set }  // 0...1
}

protocol Thermostatic {
    var setpoint: Double { get set }  // degrees Celsius
}

struct Lamp: Switchable, Dimmable {
    var name: String
    var isOn = false
    var brightness = 1.0
}

struct Heater: Switchable, Thermostatic {
    var name: String
    var isOn = false
    var setpoint = 20.0
}

struct Plug: Switchable {
    var name: String
    var isOn = false
    var load: Double  // watts drawn by whatever is plugged in
}
```

Later, someone wants to estimate the household's power use. That's a new role, cutting across the existing ones, and it can be added without modifying any of the device types:

```swift
// swiftpl/ch7/home (continued)
protocol PowerConsumer {
    var watts: Double { get }
}

extension Lamp: PowerConsumer {
    var watts: Double { isOn ? 9 * brightness : 0 }
}

extension Heater: PowerConsumer {
    var watts: Double { isOn ? 1500 : 0 }
}

extension Plug: PowerConsumer {
    var watts: Double { isOn ? load : 0 }
}

let devices: [any PowerConsumer] = [
    Lamp(name: "desk", isOn: true, brightness: 0.5),
    Heater(name: "office", isOn: true),
    Plug(name: "kettle", isOn: false, load: 2200),
]
print(devices.reduce(0) { $0 + $1.watts })  // "1504.5"
```

Each extension supplies the one requirement in terms of the type's existing properties. The array's element type, `any PowerConsumer`, can hold values of all three device types side by side; Section 7.5 explains what `any` means.

## 7.4. Parsing Flags with a Protocol

Protocols also let a library *create* values of types it has never seen. Section 2.3.2 used the `ArgumentParser` package to fill a struct's properties from the command line. It knows how to parse strings, numbers, and Booleans out of the box, and any other type can join in by conforming to its `ExpressibleByArgument` protocol:

```swift
protocol ExpressibleByArgument {
    /// Creates a new instance of this type from a command-line-specified argument.
    init?(argument: String)
    // ...and a few other requirements with default implementations...
}
```

The requirement is a *failable* initializer, `init?`, which returns `nil` for text it can't parse; `ArgumentParser` turns that into a usage error.

As an example, let's write `pace`, a command for runners that computes the time per kilometer from a distance and a duration. Both options need types `ArgumentParser` doesn't know. Here's `Duration`, accepting forms like `90s`, `25m`, and `1.5h`:

```swift
// swiftpl/ch7/pace
import ArgumentParser

extension Duration: @retroactive ExpressibleByArgument {
    /// Parses a duration like "250ms", "90s", "25m", or "1.5h".
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

The order of the table matters, since `"250ms"` also ends with `"s"`; checking `"ms"` first handles it correctly. The distance uses the `Kilometers` type from the `Distance` module of Section 2.6, accepting either kilometers or miles:

```swift
// swiftpl/ch7/pace (continued)
import Distance

extension Kilometers: @retroactive ExpressibleByArgument {
    /// Parses a distance like "5km" or "3.1mi".
    public init?(argument: String) {
        guard let value = Double(argument.dropLast(2)) else { return nil }
        switch argument.suffix(2) {
        case "km": self.init(value)
        case "mi": self = milesToKm(Miles(value))
        default: return nil
        }
    }
}
```

With those two conformances, the command itself is short:

```swift
// swiftpl/ch7/pace (continued)
import Foundation

@main
struct Pace: ParsableCommand {
    @Option(help: "the distance, like 5km or 3.1mi")
    var distance = Kilometers(5)

    @Option(help: "the time taken, like 25m or 1.5h")
    var time: Duration

    func run() {
        let seconds = Double(time.components.seconds)
            + Double(time.components.attoseconds) / 1e18
        let perKm = Int((seconds / distance.value).rounded())
        print(String(format: "%d:%02d per km", perKm / 60, perKm % 60))
    }
}
```

```
$ swift run pace --time 25m
5:00 per km
$ swift run pace --distance 10mi --time 1.5h
5:36 per km
$ swift run pace --distance 3furlongs --time 1m
Error: The value '3furlongs' is invalid for '--distance <distance>'
Usage: pace [--distance <distance>] --time <time>
```

`ArgumentParser` knows nothing about kilometers or durations. It knows only that it can call `init?(argument:)` on any `ExpressibleByArgument` type. Where `TextOutputStream` let `print` *consume* any destination, a protocol whose requirement is an initializer lets a library *produce* values of any conforming type.

Note `@retroactive` on both extensions: neither the types nor the protocol belong to this module.

The `pace` package depends on two others: `swift-argument-parser`, as in Section 2.3.2, and the `Distance` package from Section 2.6, which can be referenced by its location on disk. Its manifest's dependencies are:

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.0"),
    .package(path: "../distance"),
],
```

and its target depends on the products `ArgumentParser` and `Distance`.

**Exercise 7.5:** Extend the `Duration` parser to accept compound durations like `1h20m` and `3m30s`.

**Exercise 7.6:** `ExpressibleByArgument` has another requirement, `static var allValueStrings: [String]`, with a default of `[]`. Find out what it's for, and use it to make an option that accepts the name of a `Suit` (Section 3.6). Is there an easier way for a `CaseIterable` enum?

## 7.5. Existential Values: `any` and `some`

A value of protocol type, such as `any TextOutputStream`, is called an *existential*, a term from logic: it asserts that *there exists* some type conforming to the protocol, and holds a value of that type. The word `any` marks such types. An existential has two parts: its *dynamic type*, the concrete type of the value currently inside, and its *dynamic value*.

```swift
var out: any TextOutputStream = ""  // dynamic type String
out = Transcript()  // dynamic type Transcript
```

Swift stores an existential in a fixed-size *container* holding the dynamic type's metadata, a table of the type's implementations of the protocol's requirements (its *witness table*), and the value itself, inline if it's small (three words or fewer) and in a separate heap allocation if not. Calling a method through an existential looks up the implementation in the witness table: *dynamic dispatch*.

Existentials are flexible. An `[any PowerConsumer]` holds lamps, heaters, and plugs side by side. But the flexibility costs something. Dynamic dispatch can't be inlined, large values are boxed on the heap, and, most importantly, the concrete type is *erased*. Two `any Equatable` values can't even be compared, because they might hold different types:

```swift
let a: any Equatable = 1
let b: any Equatable = "one"
print(a == b)  // compile error: binary operator '==' cannot be applied to two 'any Equatable' operands
```

### 7.5.1. Opaque Types and `some`

The alternative is *generic* code, which Swift lets you write very concisely with the keyword `some`:

```swift
func logTwice(_ x: some CustomStringConvertible) {
    print(x.description, x.description)
}
```

`some CustomStringConvertible` means "a single specific type that conforms, chosen by each caller." At each call site, the compiler knows that type: `logTwice(42)` calls the function with `Int`, and `logTwice(Kilometers(5))` with `Kilometers`. Nothing is boxed, and the optimizer can produce a specialized copy for each type, with the calls inlined. It's shorthand for an explicit generic parameter (Section 7.7):

```swift
func logTwice<T: CustomStringConvertible>(_ x: T) { ... }
```

`some` can appear in a function's result type too, as an *opaque result type*. The function returns one specific type, which callers aren't told:

```swift
func multiples(of n: Int, upTo limit: Int) -> some Sequence<Int> {
    stride(from: n, through: limit, by: n)
}
```

Callers know they get a sequence of `Int`s, and the author can change the actual sequence type later without breaking them; but the compiler knows the real type, so there's no boxing or dynamic dispatch.

Swift's own guidance, which its diagnostics reinforce, is to use `some` by default and `any` only when you need a value whose concrete type can vary at run time, as in a heterogeneous array.

### 7.5.2. Caveat: Optionals Inside `Any`

An existential can't itself be `nil`; there's no `nil` value of type `any P`, though there is one of `(any P)?`. But since *every* type conforms to `Any`, an optional can be stored *inside* an `Any`, with confusing results:

```swift
let nickname: String? = nil
let boxed: Any = nickname  // warning: expression implicitly coerced from 'String?' to 'Any'
print(boxed)  // "nil"
if boxed is String {
    print("a string")
} else {
    print("not a string")  // printed
}
```

`boxed` isn't `nil`. It's an `Any` whose dynamic type is `Optional<String>` and whose dynamic value happens to be `.none`. The compiler warns about the conversion, and the warning is worth heeding: it's almost never what you meant. Unwrap first, or write `nickname as Any` to make the conversion explicit.

## 7.6. Sorting with `Comparable`

Sorting is one of the most common operations in programming. Swift sorts any mutable random-access collection in place with `sort()`, or any sequence into a new array with `sorted()`. Its algorithm is *stable*, so elements that compare equal keep their original relative order.

The parameterless versions require the element type to be `Comparable`, and use `<`:

```swift
var cities = ["Oslo", "Accra", "Lima", "Hanoi"]
cities.sort()
print(cities)  // "["Accra", "Hanoi", "Lima", "Oslo"]"
```

Notice what *isn't* required. A collection already knows its length and how to swap two elements, so sorting needs only the comparison. For any other order, or for elements that aren't `Comparable`, pass a function that implements "comes before":

```swift
cities.sort(by: >)  // reverse alphabetical
cities.sort { $0.count < $1.count }  // shortest first
```

Let's sort something more interesting: the results of a marathon. We'll extend the `Finisher` struct of Section 7.3 with an age (the names are invented):

```swift
// swiftpl/ch7/results
struct Finisher {
    var name: String
    var country: String
    var age: Int
    var time: Duration
}

func hms(_ h: Int, _ m: Int, _ s: Int) -> Duration {
    .seconds(h * 3600 + m * 60 + s)
}

let finishers = [
    Finisher(name: "Lena Ortiz", country: "ESP", age: 34, time: hms(2, 31, 12)),
    Finisher(name: "Tomas Berg", country: "NOR", age: 41, time: hms(2, 28, 45)),
    Finisher(name: "Ayumi Sato", country: "JPN", age: 29, time: hms(2, 31, 12)),
    Finisher(name: "Kofi Mensah", country: "GHA", age: 38, time: hms(2, 45, 3)),
]
```

A helper prints them as a table, padding each column with Foundation's `padding(toLength:withPad:startingAt:)` and formatting the times as hours, minutes, and seconds:

```swift
// swiftpl/ch7/results (continued)
import Foundation

func printResults(_ finishers: [Finisher]) {
    for f in finishers {
        let time = f.time.formatted(.time(pattern: .hourMinuteSecond))
        print(f.name.padding(toLength: 13, withPad: " ", startingAt: 0), f.country, f.age, time)
    }
}
```

Sorting by time gives the finishing order:

```swift
// swiftpl/ch7/results (continued)
printResults(finishers.sorted { $0.time < $1.time })
```

```
Tomas Berg    NOR 41 2:28:45
Lena Ortiz    ESP 34 2:31:12
Ayumi Sato    JPN 29 2:31:12
Kofi Mensah   GHA 38 2:45:03
```

Lena Ortiz and Ayumi Sato finished in the same time, and the stable sort left them in their original order. A results table would usually break ties by some other field, say alphabetically by name. That calls for a sort on *two* keys, and Swift's tuples make it a one-liner, because tuples of `Comparable` values (up to six of them) compare lexicographically, element by element:

```swift
// swiftpl/ch7/results (continued)
printResults(finishers.sorted { ($0.time, $0.name) < ($1.time, $1.name) })
```

```
Tomas Berg    NOR 41 2:28:45
Ayumi Sato    JPN 29 2:31:12
Lena Ortiz    ESP 34 2:31:12
Kofi Mensah   GHA 38 2:45:03
```

When the sort keys are chosen at run time, perhaps by a user clicking column headings, it helps to represent each key as a value. Foundation's `KeyPathComparator` does that, pairing a key path with a direction, and `sorted(using:)` sorts by a list of them in priority order:

```swift
// swiftpl/ch7/results (continued)
var order = [KeyPathComparator(\Finisher.time)]
order.insert(KeyPathComparator(\Finisher.age, order: .reverse), at: 0)  // the user chose "oldest first"
printResults(finishers.sorted(using: order))
```

### 7.6.1. Making a Type `Comparable`

When a type has one natural ordering, make it `Comparable` by implementing `<`:

```swift
// swiftpl/ch7/results (continued)
extension Finisher: Comparable {
    static func < (a: Finisher, b: Finisher) -> Bool {
        (a.time, a.name) < (b.time, b.name)
    }
}

let podium = finishers.sorted().prefix(3)
let winner = finishers.min()!
```

In return for that one operator, the type gets `sort()`, `sorted()`, `min()`, `max()`, the other comparison operators, ranges, and `clamped(to:)`, all from protocol extensions.

`<` must be a *strict weak ordering*: never true of an element and itself, transitive, and consistent in what it treats as ties. Sorting depends on it, and a comparison that breaks these rules can produce output that isn't sorted, without any error.

A sorted check is a neat use of `zip`, which pairs each element with its successor:

```swift
// swiftpl/ch7/results (continued)
func isSorted<T: Comparable>(_ values: [T]) -> Bool {
    zip(values, values.dropFirst()).allSatisfy { $0 <= $1 }
}
print(isSorted(finishers.sorted()))  // "true"
```

**Exercise 7.7:** Print the results grouped by country, with the countries in alphabetical order and each country's runners in finishing order, using a single sort.

**Exercise 7.8:** Implement a results table that remembers the order in which a user clicked column headings, sorting first by the most recent click, then by the one before, and so on, and reversing a column's direction when it's clicked twice in a row. Represent the state as an array of `KeyPathComparator<Finisher>`.

**Exercise 7.9:** Use the `HTML` type from Section 4.6 to produce the results as an HTML table whose column headings are links that re-sort the table. Serve it with Hummingbird.

## 7.7. Generics and Associated Types

Generics and protocols are two halves of one design in Swift. Protocols describe capabilities; generics let code be written once for every type with those capabilities.

### 7.7.1. Generic Functions

A *generic function* has *type parameters*, written in angle brackets after its name, and each type parameter can be *constrained* to conform to protocols:

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

`T` stands for whatever element type the caller uses, and because `T` is constrained to `Comparable`, the body may use `>` on `T` values. The compiler infers `T` at each call, as `Int` for the first and `String` for the second, and checks that it meets the constraint. The body is type-checked once, against the constraint alone; it can use only what `Comparable` guarantees. (That's a key difference from C++ templates, which are checked separately for each type they're used with.)

More complex constraints go in a `where` clause:

```swift
func allEqual<C: Collection>(_ c: C) -> Bool where C.Element: Equatable {
    guard let first = c.first else { return true }
    return c.allSatisfy { $0 == first }
}
```

How does a single compiled function work with every type? Unoptimized, it receives hidden arguments describing `T` and its conformances, and calls `<` or `==` through them. Optimized, it's usually *specialized*: the compiler generates a separate copy for each type it's used with, with all those calls inlined, so generic code runs as fast as code written for one type.

### 7.7.2. Generic Types

Types can have type parameters too. All the standard collections are generic (`Array<Element>`, `Dictionary<Key, Value>`, `Set<Element>`, `Optional<Wrapped>`), and so is the `RingBuffer` of Section 6.5. Here's a minimal stack:

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

An extension can add members or conformances that apply only when the type parameters meet extra constraints, a *conditional conformance*:

```swift
// swiftpl/ch7/stack (continued)
extension Stack: Equatable where Element: Equatable {}  // == is synthesized
extension Stack: Sendable where Element: Sendable {}
```

A stack of strings can now be compared with `==`, but a stack of closures, which aren't `Equatable`, can't. The standard library relies on this throughout: arrays are `Equatable` when their elements are, `Hashable` when their elements are, `Codable` when their elements are, and so on.

### 7.7.3. Associated Types

Some protocols involve related types that differ from one conforming type to another. A sequence has an element type, for example, which is `Int` for `[Int]`, `Character` for `String`, and a key-value pair for a dictionary. A protocol declares such a type with `associatedtype`:

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

An *iterator* produces a sequence's elements one at a time, returning `nil` when there are no more. A `for`-`in` loop is shorthand for exactly that:

```swift
for x in sequence {
    print(x)
}
// means:
var it = sequence.makeIterator()
while let x = it.next() {
    print(x)
}
```

Here's a sequence that counts down from a given number to 1:

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

The compiler infers `Element` to be `Int` from the result type of `next()`. Since `Countdown` serves as its own iterator, the default `makeIterator()` simply returns a copy of it. And by conforming to `Sequence`, it acquires every sequence algorithm:

```swift
// swiftpl/ch7/countdown (continued)
let c = Countdown(count: 5)
print(c.map { $0 * $0 })  // "[25, 16, 9, 4, 1]"
print(c.filter { $0.isMultiple(of: 2) })  // "[4, 2]"
print(c.reduce(0, +))  // "15"
print(c.contains(3))  // "true"
print(Array(c.prefix(2)))  // "[5, 4]"
```

A protocol can name its most important associated types as *primary associated types*, in angle brackets after its name, as in `protocol Sequence<Element>` and `protocol Collection<Element>`. That's what makes constrained types like `some Sequence<Int>`, used for `multiples(of:upTo:)` in Section 7.5, and `any Collection<String>` possible.

### 7.7.4. Generics versus Existentials

There are now two ways to write code that works with many types: with an existential, `any P`, or with a generic parameter, `<T: P>` or `some P`. Which should you use?

Generics preserve type information. `largest` returns a `T`, so a caller passing `[Int]` gets back an `Int?`, not some boxed value. Generic code can relate several values to each other, as when two `T`s are compared with `>`, and the optimizer can specialize it. Generics should be your default.

Existentials *erase* type information, and that's exactly what's needed when values of different types must be stored or passed together: an array of devices, a property that can hold any error, a registry of plug-ins. Every existential has the same size and layout, whatever is inside.

In practice, most Swift code uses concrete types most of the time, generics when writing algorithms that apply to many types, and existentials at the points where values of varying types genuinely meet, of which `any Error` is the most common.

**Exercise 7.10:** Write a generic `isPalindrome` that accepts any `BidirectionalCollection` with `Equatable` elements, and test it on arrays, strings, and a string's `utf8` view.

**Exercise 7.11:** Make the `RingBuffer` from Section 6.5 conform to `Sequence`, with an iterator that yields the elements from oldest to newest. Which methods does it gain?

**Exercise 7.12:** Write a generic `struct LRUCache<Key: Hashable, Value>` with a fixed capacity, which evicts the least recently used entry when a new one is added to a full cache.

## 7.8. The `Error` Protocol

Every thrown value conforms to `Error`, which, as Section 7.2 showed, requires nothing except `Sendable`:

```swift
protocol Error: Sendable {}
```

So any type can be an error, and different libraries use enums, structs, and classes. A plain `throws` function can throw any of them, which is why a caught error has the existential type `any Error`. Its dynamic type might be one of your own enums, a standard library `DecodingError`, a Foundation `URLError` or `CocoaError`, or a `CancellationError` from the concurrency library.

The simplest error type carries just a message:

```swift
struct MessageError: Error, CustomStringConvertible {
    let description: String
    init(_ description: String) { self.description = description }
}

throw MessageError("configuration file is empty")
```

A single catch-all type like this is tempting, but callers can't do much with it except print it. Errors that describe their failures, such as an enum whose cases name the ways an operation fails or a struct carrying the details of one failure, let callers make decisions, as Section 7.11 shows.

Errors aren't `Equatable` by default, so two `any Error` values can't be compared. An enum error whose associated values are `Equatable` can declare the conformance and get `==` for free, which is useful in tests:

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

At the lowest level, operating-system calls report errors as integer codes. The `swift-system` package wraps them in a type, `Errno`, with names like `.noSuchFileOrDirectory`, and Foundation's `POSIXError` does the same.

## 7.9. Example: Expression Evaluator

In this section, we'll build an evaluator for arithmetic expressions such as `price * (1 + tax)` or `sqrt(x * x + y * y)`, and use it in an interactive calculator. It's a compact example of how Swift models a structured language: the expressions become values of an `enum`, a parser builds them from text, and functions over the enum evaluate and check them.

The language has floating-point numbers; variables; the binary operators `+`, `-`, `*`, and `/` with the usual precedence; unary `+` and `-`; parentheses; and four functions, `min(a, b)`, `max(a, b)`, `abs(x)`, and `sqrt(x)`. Every value is a `Double`. An expression is one of five kinds of thing, which makes it a natural enum:

```swift
// swiftpl/ch7/eval
/// An Expr is an arithmetic expression.
indirect enum Expr: Equatable {
    case literal(Double)  // a numeric constant, e.g., 2.5
    case variable(String)  // a variable, e.g., x
    case unary(Character, Expr)  // a unary operator expression, e.g., -x; op is '+' or '-'
    case binary(Character, Expr, Expr)  // a binary operator expression, e.g., x+y; op is '+', '-', '*', or '/'
    case call(String, [Expr])  // a function call, e.g., max(a, b); fn is "min", "max", "abs", or "sqrt"
}
```

The enum is `indirect` because its cases contain other `Expr` values directly, so their payloads must be stored on the heap (Section 4.4.1).

To evaluate an expression with variables, we need an *environment* giving their values:

```swift
// swiftpl/ch7/eval (continued)
typealias Env = [String: Double]
```

Evaluation is a single method that switches over the kinds of expression, recursively evaluating the parts:

```swift
// swiftpl/ch7/eval (continued)
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
            case "min": return min(args[0].eval(env), args[1].eval(env))
            case "max": return max(args[0].eval(env), args[1].eval(env))
            case "abs": return abs(args[0].eval(env))
            case "sqrt": return sqrt(args[0].eval(env))
            default: fatalError("unsupported function call: \(fn)")
            }
        }
    }
}
```

A literal is its own value, and a variable is looked up in the environment. An operator evaluates its operands and combines them. A function call evaluates its arguments and applies the function. Division by zero isn't treated as an error: IEEE arithmetic gives it a result, an infinity or NaN.

Some expressions can't be evaluated: those with an unknown function, a function called with the wrong number of arguments, or an operator outside the language. `eval` traps on them, on the assumption that they'll have been rejected earlier. We'll write a checking pass for that shortly. First we need a parser, to turn text into `Expr` values. Here's a compact recursive-descent parser, with a lexer that splits the text into tokens:

```swift
// swiftpl/ch7/eval (continued)
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

The parser follows a classic design. `parseBinary` handles binary operators by *precedence climbing*: it parses one operand, and then, as long as the next token is an operator that binds at least as tightly as `minPrecedence`, consumes the operator and parses a right operand that may contain only operators binding *more* tightly. That's how `1 + 2 * 3` becomes `1 + (2 * 3)` and `10 - 4 - 3` becomes `(10 - 4) - 3`. Each function's typed `throws(SyntaxError)` says that a syntax error is the only thing that can go wrong.

Note the nested patterns in `precedence` and `parsePrimary`: `case .symbol("*")` matches a token that is a symbol whose character is `*`, all in one pattern.

The checking pass walks the tree, rejecting anything `eval` couldn't handle, and collects the variables the expression uses, so that a caller can make sure they're all defined:

```swift
// swiftpl/ch7/eval (continued)
extension Expr {
    static let arity = ["min": 2, "max": 2, "abs": 1, "sqrt": 1]

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

Separating checking from evaluation means an expression can be validated once and then evaluated any number of times, with different environments, knowing that `eval` won't trap.

Now the calculator. For now, put it in the same package as the evaluator, as the `main.swift` of a single executable; Section 10.3 shows how the evaluator becomes a library module, `Eval`, with `calc` as a separate executable that imports it. (The evaluator's declarations would then need to be `public`.) The calculator reads lines from the standard input. A line of the form `name = expression` defines a variable; any other line is an expression to evaluate:

```swift
// swiftpl/ch7/calc (builds on eval)
// Calc evaluates expressions read from standard input.
// A line "name = expression" defines a variable for later lines to use.
import Foundation

var env: Env = [:]
while let line = readLine() {
    let (target, text) = splitAssignment(line)
    do {
        let expr = try parse(text)
        var vars = Set<String>()
        try expr.check(&vars)
        if let undefined = vars.sorted().first(where: { env[$0] == nil }) {
            print("error: undefined variable \(undefined)")
            continue
        }
        let value = expr.eval(env)
        if let target {
            env[target] = value
        }
        print(String(format: "%.10g", value))
    } catch {
        print("error: \(error)")
    }
}

/// Splits "name = expression" into its parts; for any other line,
/// returns nil and the whole line.
func splitAssignment(_ line: String) -> (String?, String) {
    let parts = line.split(separator: "=", maxSplits: 1)
    guard parts.count == 2 else { return (nil, line) }
    let name = parts[0].trimmingCharacters(in: .whitespaces)
    guard let first = name.first, first.isLetter, name.allSatisfy({ $0.isLetter || $0.isNumber }) else {
        return (nil, line)
    }
    return (name, String(parts[1]))
}
```

Here's a session:

```
$ swift run calc
price = 24.5
24.5
tax = 0.08
0.08
price * (1 + tax)
26.46
max(price, 30) - min(price, 30)
5.5
sqrt(3 * 3 + 4 * 4)
5
total / 2
error: undefined variable total
x % 2
error: unexpected symbol("%")
log(10)
error: unknown function "log"
max(1)
error: call to max has 1 args, want 2
```

Each result is formatted with `%.10g`, which prints up to ten significant digits and drops trailing zeros, so `26.46` doesn't appear as `26.460000000000001`, the nearest `Double`'s full expansion. The errors come from three different places: the parser (`%` isn't a token the grammar knows), the checker (unknown functions and wrong argument counts), and the calculator itself (undefined variables).

**Exercise 7.13:** Make `Expr` conform to `CustomStringConvertible`, with a `description` that prints the expression with only the parentheses that are necessary. Check that parsing the description gives back an equal `Expr`.

**Exercise 7.14:** Add a new kind of expression, such as comparison operators producing 1 or 0, or a conditional `if(c, a, b)` function. Update `parse`, `eval`, `check`, and `description`. How does the compiler help you find every place that needs to change?

**Exercise 7.15:** Let `calc` accept several assignments and an expression on one line, separated by semicolons.

**Exercise 7.16:** Combine the evaluator with the `heatmap` program of Section 3.2.1, so that `heatmap 'sin(x) * sin(y)'` plots any expression in `x` and `y`, after checking that it uses no other variables.

### 7.9.1. Enums or Protocols?

We represented expressions as an enum and wrote each operation, `eval` and `check`, as a `switch` over all the cases. The other natural design represents each kind of expression as its own type conforming to a protocol, with each operation implemented separately by each type:

```swift
protocol ExprNode {
    func eval(_ env: Env) -> Double
    func check(_ vars: inout Set<String>) throws(SyntaxError)
}

struct Literal: ExprNode { ... }
struct Binary: ExprNode { var op: Character; var x, y: any ExprNode; ... }
```

The two designs have opposite strengths. With an enum, adding a new *operation*, such as a pretty-printer, is easy: write one function with a `switch`, and the compiler ensures it handles every case. But adding a new *kind* of expression means updating every existing operation. With a protocol, adding a new kind is easy, even from another module (just write a new conforming type), but adding an operation means touching every type. (This is sometimes called *the expression problem*.)

For a closed set of alternatives, like the grammar of a small language, Swift programmers generally choose enums, and exhaustiveness checking makes them robust as the code evolves. For an open set, such as a plug-in interface or shapes that clients define, protocols are the right fit.

## 7.10. Type Casting

A *type cast* converts a value to a related type. Swift has three cast operators.

The *upcast* `as` converts to a more general type that the value is statically known to conform to or inherit from. It can't fail, so it's usually implicit; it's needed only to resolve ambiguity, as in `x as Any` or `1 as Double`.

The *conditional downcast* `as?` attempts a conversion to a more specific type, based on the value's *dynamic* type, and returns an optional: the converted value if the dynamic type fits, `nil` otherwise. The *forced downcast* `as!` does the same but traps on failure. And `is` simply tests whether a cast would succeed.

The target of a downcast can be a concrete type, in which case the cast succeeds if the dynamic type is that type, or, for classes, a subclass of it:

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

Or the target can be a protocol type, in which case the cast succeeds if the dynamic type *conforms* to the protocol, and produces an existential of that protocol type. That's how code asks, at run time, whether a value supports a capability its static type doesn't promise:

```swift
func describe(_ v: Any) -> String {
    if let c = v as? any CustomStringConvertible {
        return c.description
    }
    return "<\(type(of: v))>"
}
```

Swift's casts do more than simple type tests. They see through optionals (a cast from an `Int?` holding 5 to `Int` succeeds), convert between Swift and Foundation types on Apple platforms (`NSString` to `String`), and convert collections element by element (`[Any]` to `[Int]` succeeds if every element is an `Int`). Reserve `as!` for casts whose failure would mean a bug.

## 7.11. Discriminating Errors with Casts

A caught error has the type `any Error`, so deciding what to do about it often starts with a cast. A common question in network code is whether a failure is *transient*, likely to go away if the operation is retried, such as a timeout or a dropped connection, or permanent, such as a bad URL or a "not found" response. Section 5.4 wrote `withRetries`, which retried any failure. It's better to retry only the transient ones, and a function can sort errors into the two groups by examining their types:

```swift
// swiftpl/ch7/transient
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

/// An HTTP response with an unsuccessful status code.
struct HTTPError: Error {
    var status: Int
}

/// Reports whether a failed request is worth retrying.
func isTransient(_ error: any Error) -> Bool {
    switch error {
    case let e as URLError:
        let transient: [URLError.Code] = [
            .timedOut, .networkConnectionLost, .notConnectedToInternet, .cannotConnectToHost,
        ]
        return transient.contains(e.code)
    case let e as POSIXError:
        return e.code == .ECONNRESET || e.code == .ETIMEDOUT
    case let e as HTTPError:
        return e.status == 429 || (500...599).contains(e.status)  // rate-limited, or server trouble
    default:
        return false
    }
}
```

Each `case let e as T` pattern matches errors whose dynamic type is `T` and binds them with that type, so the case's code can examine type-specific details: `URLError`'s `code`, `POSIXError`'s error number, our own `HTTPError`'s status. With this function, `withRetries` can retry selectively, by adding it to the condition on its `catch` clause:

```swift
} catch where attempt < attempts && isTransient(error) {
```

The same kind of discrimination can be written directly in `catch` clauses, combining a cast with a condition:

```swift
do {
    let data = try Data(contentsOf: configURL)
    apply(data)
} catch let error as CocoaError where error.code == .fileReadNoSuchFile {
    useDefaultConfiguration()
} catch {
    printError("can't read configuration: \(error)")
}
```

Wrapping errors to add context (Section 5.4) can hide the original type from such tests: a cast to `URLError` fails if the `URLError` is inside a wrapper. A well-designed wrapper exposes what it wraps, like the `underlying` property of Section 5.4's `LoadError`, so that a classifier can look inside. As a rule, though, decide what an error means close to where it happened, before it's wrapped and passed up.

## 7.12. Querying Behaviors with Conditional Casts

Casts to protocol types let a function adapt to what a value can do. A logging function, for instance, wants the most informative description it can get of whatever it's given, and different protocols offer different levels of detail:

```swift
import Foundation

/// Returns the most useful available description of value for a log message.
func logDescription(_ value: Any) -> String {
    switch value {
    case let e as any LocalizedError:
        return e.errorDescription ?? "\(e)"  // a human-readable error message
    case let v as any CustomDebugStringConvertible:
        return v.debugDescription  // a description meant for developers
    case let v as any CustomStringConvertible:
        return v.description
    default:
        return String(reflecting: value)  // fall back on reflection (Chapter 12)
    }
}
```

The cases are tried in order, from the most specific to the most general, which is the same strategy `print` and string interpolation use when they turn a value into text.

The same technique lets a generic algorithm take a faster path when its input supports one. A function that accepts any sequence can check whether it's actually a random-access collection, which can reach any position directly:

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

Passing the existential `c` to the generic `middleElement` *opens* it: the compiler unboxes the value and calls the function with its dynamic type as `C`.

Such run-time checks are less common in Swift than you might expect, because the compiler can usually choose the right implementation statically. The standard library's algorithms are defined at several levels, on `Sequence`, `Collection`, `BidirectionalCollection`, and `RandomAccessCollection`, and when a more capable type has a faster implementation of a requirement, it's selected automatically. `count`, for example, walks a general collection but is constant-time for an array.

## 7.13. Type Switches

Protocols get used in two quite different styles. In the first, illustrated by `TextOutputStream`, `Sequence`, and `Comparable`, a protocol captures what its conforming types have *in common*, and code using it deliberately ignores which type it has. In the second, a protocol type (often `Any`) is treated as a *union* of specific types, and code examines each value to find out which one it has, handling each differently. A `switch` with type-casting patterns expresses the second style directly.

Suppose we need to write values of a few basic types as JSON literals:

```swift
// swiftpl/ch7/json
func jsonLiteral(_ x: Any?) -> String {
    switch x {
    case nil:
        return "null"
    case let b as Bool:
        return b ? "true" : "false"
    case let i as Int:
        return String(i)
    case let d as Double:
        return d.isFinite ? String(d) : "null"  // JSON has no infinities or NaN
    case let s as String:
        return quoted(s)
    default:
        fatalError("unsupported type \(type(of: x!))")
    }
}

/// Returns s as a JSON string literal, with quotes and backslashes escaped.
func quoted(_ s: String) -> String {
    "\"" + s.replacing("\\", with: "\\\\").replacing("\"", with: "\\\"") + "\""
}

print(jsonLiteral("say \"hi\""), jsonLiteral(3), jsonLiteral(nil))
// "say \"hi\"" 3 null
```

Each `case let x as T` matches values whose dynamic type is `T`, binding them as a `T`; `case nil` matches an empty optional; and `case is T` would test the type without binding. Cases are tried in order, and the first match runs. (A real JSON writer would also escape control characters; that's left for Exercise 7.20.)

This works, but Swift has a better tool for a closed set of alternatives: an enum. JSON's values are exactly null, Booleans, numbers, strings, arrays, and objects, so they can be modeled precisely:

```swift
// swiftpl/ch7/json (continued)
enum JSONValue {
    case null
    case bool(Bool)
    case number(Double)
    case string(String)
    case array([JSONValue])
    case object([String: JSONValue])

    var literal: String {
        switch self {
        case .null: "null"
        case .bool(let b): b ? "true" : "false"
        case .number(let n): n.isFinite ? String(n) : "null"
        case .string(let s): quoted(s)
        case .array(let items): "[" + items.map(\.literal).joined(separator: ",") + "]"
        case .object(let fields):
            "{" + fields.sorted(by: { $0.key < $1.key })
                .map { quoted($0.key) + ":" + $0.value.literal }
                .joined(separator: ",") + "}"
        }
    }
}

let doc = JSONValue.object(["name": .string("Ada"), "tags": .array([.string("math"), .null])])
print(doc.literal)  // "{"name":"Ada","tags":["math",null]}"
```

Now the `default` case and its `fatalError` are gone: the compiler knows the `switch` covers every case, and a caller *can't* pass an unsupported type such as a `Date` by mistake. Nested arrays and objects work too, by recursion, which the `Any` version didn't attempt. An enum like this is sometimes called a *discriminated union* or *sum type*; in Swift it's simply an enum, checked by the type system. Reserve `Any` and type switches for data whose types really can't be enumerated in advance.

## 7.14. Example: Reading a News Feed with XMLParser

Section 4.5 decoded JSON into Swift types with `Decodable`. XML can be handled the same way with third-party packages, but sometimes it's better to process a document *as it's read*, without building a complete data structure, because the document is large, or because you need only a little of it. Foundation's `XMLParser` does that. It reads a document from start to finish and reports each start tag, end tag, and run of text as it goes, by calling methods on a *delegate*, an object conforming to the `XMLParserDelegate` protocol. The protocol's methods are all optional; a delegate implements those for the events it cares about.

News sites and blogs publish their latest articles as *feeds* in one of two XML formats, RSS and Atom. In RSS, each article is an `<item>` with `<title>` and `<link>` elements, the link's URL written as text. In Atom, each article is an `<entry>` with a `<title>` and a `<link href="...">` whose URL is an attribute. The program below reads a feed in either format and lists its articles:

```swift
// swiftpl/ch7/feed
// Feed lists the titles and links of the articles in an RSS or Atom feed.
import Foundation
#if canImport(FoundationXML)
import FoundationXML
#endif

final class FeedReader: NSObject, XMLParserDelegate {
    struct Article {
        var title = ""
        var link = ""
    }

    private(set) var articles: [Article] = []
    private var current: Article?  // the article being read, if any
    private var text = ""  // text accumulated since the last tag

    func parser(_ parser: XMLParser, didStartElement name: String, namespaceURI: String?,
                qualifiedName: String?, attributes: [String: String] = [:]) {
        switch name {
        case "item", "entry":
            current = Article()
        case "link" where current != nil:
            if let href = attributes["href"] {
                current?.link = href  // Atom: the URL is an attribute
            }
        default:
            break
        }
        text = ""
    }

    func parser(_ parser: XMLParser, foundCharacters string: String) {
        text += string
    }

    func parser(_ parser: XMLParser, didEndElement name: String, namespaceURI: String?,
                qualifiedName: String?) {
        let value = text.trimmingCharacters(in: .whitespacesAndNewlines)
        switch name {
        case "title":
            current?.title = value
        case "link" where !value.isEmpty:
            current?.link = value  // RSS: the URL is the element's text
        case "item", "entry":
            if let article = current {
                articles.append(article)
            }
            current = nil
        default:
            break
        }
        text = ""
    }
}

let data = FileHandle.standardInput.readDataToEndOfFile()
let parser = XMLParser(data: data)
let reader = FeedReader()
parser.delegate = reader
guard parser.parse() else {
    let reason = parser.parserError.map { "\($0)" } ?? "parse failed"
    FileHandle.standardError.write(Data("feed: \(reason)\n".utf8))
    exit(1)
}
for article in reader.articles {
    print(article.title)
    print("    \(article.link)")
}
```

```
$ download https://www.swift.org/atom.xml | swift run feed
Announcing Swift 6.2
    https://www.swift.org/blog/swift-6.2-released/
Swift on Embedded Devices: A Progress Report
    https://www.swift.org/blog/embedded-progress/
...
```

(The articles shown are illustrative; a live feed shows whatever has been published recently.)

On Linux, `XMLParser` is in a separate module, `FoundationXML`, so that programs that don't parse XML needn't link the XML library, hence the conditional import, as with `FoundationNetworking`.

The delegate is a small *state machine*. `current` is non-`nil` while the parser is inside an article, so the feed's own `<title>`, which appears outside any article, is ignored: `current?.title = value` does nothing when `current` is `nil`. Text arrives through `foundCharacters`, possibly in several pieces for one element, so it's accumulated in `text` and used when the element ends. The two formats differ only in where the link's URL is found, which the two `link` cases handle.

`FeedReader` must be a class descended from `NSObject`, because `XMLParserDelegate` comes from Objective-C, where every object descends from `NSObject`. The parser holds its delegate weakly, to avoid a reference cycle, so the program keeps its own reference, `reader`, until parsing is done.

The delegate style is a *push* interface: the parser is in control and calls our code. The alternative is a *pull* interface, in which our code asks for the next event, as with an iterator. Pull interfaces are generally easier to compose, and in modern Swift they'd usually be provided as an `AsyncSequence` (Chapter 8) of parsing events, consumed with `for try await`.

**Exercise 7.17:** Include each article's publication date (`<pubDate>` in RSS, `<updated>` in Atom), and print the articles newest first.

**Exercise 7.18:** Some feeds wrap titles in CDATA sections, which `XMLParser` reports through a different delegate method, `parser(_:foundCDATA:)`. Handle it.

**Exercise 7.19:** Use `XMLParser` to build a tree for an arbitrary XML document, with nodes of two kinds: `indirect enum XMLNode { case text(String); case element(name: String, attributes: [String: String], children: [XMLNode]) }`.

**Exercise 7.20:** Complete `quoted` so that it escapes control characters as JSON requires (`\n`, `\t`, and `\u00XX` for the others), and test it on strings containing every character below U+0020.

## 7.15. Choosing Between Protocols, Generics, and Enums

Swift gives you many ways to write code that works with more than one type, and the most common design mistake is reaching for more abstraction than a problem needs. Some guidelines:

**Start concrete.** Write the type you need. Introduce a protocol when a second type needs to be used in the same way, not before. A protocol with a single conforming type is an extra layer that readers must look through, and, used as an existential, it costs performance as well. Access control (Section 6.6) can hide a concrete type's details perfectly well without one.

There are good reasons to break this rule occasionally: a protocol can decouple two modules when the concrete type can't live in the same module as its users, and it can make room for a test double. (Even then, passing a closure is often simpler than defining a protocol.)

**Keep protocols small.** A protocol should ask only for what its users need. Small protocols, like `TextOutputStream`, `Error`, and `Equatable`, are easy for new types to adopt and easy to compose. Larger capabilities can be built by inheritance and composition from small ones.

**Prefer `some` to `any`.** Generic code (`some P` or `<T: P>`) keeps type information, can relate values to each other, and is optimized as if written for each type. Use existentials (`any P`) where values of different types genuinely meet: in heterogeneous collections, in properties that hold "any error" or "any delegate," and at the boundaries of a system.

**Prefer enums for closed sets.** When you can list all the alternatives, as with the kinds of expression, JSON values, or the states of a connection, an enum with associated values gives you exhaustiveness checking and makes impossible values unrepresentable. Choose protocols when the set of alternatives must stay open to extension by others.

**Remember plain functions and types.** Not everything needs a protocol, a generic parameter, or a class hierarchy. A free function that does one job, or a struct with no conformances at all, is often the clearest design, and the best foundation for anything more elaborate built later.
