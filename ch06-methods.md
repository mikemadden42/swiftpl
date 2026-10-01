# 6. Methods

A *method* is a function that belongs to a type. Rather than writing `length(of: interval)`, you write `interval.length`; rather than `shift(&interval, by: 2)`, you write `interval.shift(by: 2)`. The difference looks cosmetic, but it changes how code is organized: the operations on a type are grouped with the type, a reader can discover them by typing a dot, and the type's author can keep its representation private while exposing only the operations that keep it valid. That combination of data with the operations that maintain it is the core of what's usually called *object-oriented programming*.

Swift supports object-oriented programming in the classic style, with classes, inheritance, and overriding. But its everyday style differs from Java's or C++'s in two important ways. First, methods aren't reserved for classes: structs and enums have them too, and most Swift types are structs. Second, behavior is shared between types mostly through *protocols* and *extensions* rather than through inheritance. This chapter covers methods and the related ideas of extensions, mutation, composition, and encapsulation, and then class inheritance, property wrappers, and how the memory of class instances is managed. Chapter 7 then takes up protocols.

## 6.1. Method Declarations

A method is declared inside a type's body, or in an *extension* of the type, which we'll come to shortly. Within a method, the instance it was called on is available as `self`. Here's a type representing an interval of the real number line, such as a span of time measured in hours, with a few methods:

```swift
// swiftpl/ch6/interval
/// An Interval is the closed range of real numbers from lower to upper.
struct Interval {
    var lower: Double
    var upper: Double

    /// The length of the interval.
    var length: Double { upper - lower }

    /// Reports whether x lies within the interval.
    func contains(_ x: Double) -> Bool {
        lower <= x && x <= upper
    }

    /// Reports whether the interval shares any point with another.
    func overlaps(_ other: Interval) -> Bool {
        lower <= other.upper && other.lower <= upper
    }
}
```

Inside the methods, `lower` and `upper` refer to the instance's properties, short for `self.lower` and `self.upper`. Writing `self.` explicitly is needed only to resolve an ambiguity, such as a parameter with the same name as a property, or in certain closures (Section 5.6.1).

A method is called with dot syntax, the instance first:

```swift
// swiftpl/ch6/interval (continued)
let meeting = Interval(lower: 9, upper: 10.5)
let lunch = Interval(lower: 12, upper: 13)
print(meeting.length)  // "1.5"
print(meeting.contains(10))  // "true"
print(meeting.overlaps(lunch))  // "false"
```

The expression `meeting.overlaps` *selects* the method `overlaps(_:)` of `Interval` for the instance `meeting`. Properties are selected the same way, and they share a namespace with methods: a type can't have both a property and a method named `length`.

Each type has its own namespace for methods, so different types can each have a method called `contains`, and the compiler picks the right one from the type of the instance. `contains` on an `Interval` and `contains` on an `Array` are unrelated methods that happen to share a name.

### 6.1.1. Extensions

An *extension* adds members to a type after its declaration: methods, computed properties, initializers, subscripts, nested types, and protocol conformances. Extensions can be written in another file, and even in another module. They can extend any type, including ones from the standard library.

Suppose a calendar program keeps the day's meetings in an array of intervals, and needs to know how much of the day is booked. Overlapping meetings mustn't be counted twice: two meetings from 9:00 to 10:00 and 9:30 to 11:00 occupy two hours, not two and a half. That question belongs to *arrays of intervals*, and a *constrained extension* adds a method to exactly those:

```swift
// swiftpl/ch6/interval (continued)
extension Array where Element == Interval {
    /// Returns the total length covered by the intervals, counting overlaps once.
    func coveredLength() -> Double {
        var total = 0.0
        var current: Interval? = nil  // the run of overlapping intervals being merged
        for next in sorted(by: { $0.lower < $1.lower }) {
            if var run = current, next.lower <= run.upper {
                run.upper = max(run.upper, next.upper)  // extend the run
                current = run
            } else {
                total += current?.length ?? 0  // close the previous run
                current = next
            }
        }
        return total + (current?.length ?? 0)
    }
}

let schedule = [
    Interval(lower: 9, upper: 10),
    Interval(lower: 9.5, upper: 11),
    Interval(lower: 13, upper: 14),
]
print(schedule.coveredLength())  // "3.0"
```

The `where` clause limits the extension to arrays whose elements are `Interval`s, so `coveredLength()` is available on `[Interval]` and nowhere else. Inside it, `self` is the array, and `sorted(by:)` is the array's own method. The algorithm sorts the intervals by start time and sweeps through them, merging each into the current *run* if it overlaps and starting a new run otherwise.

Extending types you don't own is routine in Swift, and it's more flexible than what some languages allow, where methods can be declared only on types defined in the same package or class. The two limits are that an extension can't add *stored* properties, since that would change the type's memory layout, and can't replace a member the type already has.

### 6.1.2. Properties or Methods?

`length` is a *computed property*: it's declared with `var` and a body, and it looks like a stored property to its users. `coveredLength()` is a method. The convention for choosing between them is that a computed property should be cheap (roughly constant time), have no side effects, and describe an attribute of the value, like `count`, `isEmpty`, or `description`. Anything that does substantial work, may fail, or changes something should be a method. `coveredLength()` sorts the whole array, so it's a method.

### 6.1.3. Static Members

Members marked `static` belong to the type itself rather than to any instance. Static constants and static *factory methods*, which construct instances in a particular way, are both common:

```swift
// swiftpl/ch6/interval (continued)
extension Interval {
    static let unit = Interval(lower: 0, upper: 1)

    /// Returns the interval centered on center that extends radius in each direction.
    static func around(_ center: Double, radius: Double) -> Interval {
        Interval(lower: center - radius, upper: center + radius)
    }
}

let tolerance = Interval.around(5, radius: 0.1)
print(tolerance.contains(5.05))  // "true"
print(tolerance.overlaps(.unit))  // "false"
```

In the last line, `.unit` is an *implicit member expression*: where the expected type is known, a static member can be named with just a leading dot.

## 6.2. Mutating Methods and Reference Types

Because a struct is a value, its methods can't change it unless they're marked `mutating`. A mutating method receives `self` as an implicit `inout` parameter (Section 2.3.2), so its changes are written back to the variable it was called on:

```swift
// swiftpl/ch6/interval (continued)
extension Interval {
    /// Moves the interval by delta.
    mutating func shift(by delta: Double) {
        lower += delta
        upper += delta
    }
}

var slot = Interval(lower: 14, upper: 15)
slot.shift(by: 1.5)
print(slot)  // "Interval(lower: 15.5, upper: 16.5)"
```

A mutating method can be called only on a variable:

```swift
let fixed = Interval(lower: 0, upper: 1)
fixed.shift(by: 1)  // compile error: cannot use mutating member on immutable value: 'fixed' is a 'let' constant
```

That's what makes `let` meaningful for structs. The compiler guarantees that a non-mutating method doesn't change the value, and that a `let` struct never changes at all, so readers and the optimizer can both rely on it.

The API Design Guidelines recommend naming mutating and non-mutating versions of an operation consistently. When the operation is a verb, the mutating method uses the plain verb and the non-mutating one uses its "-ed" or "-ing" form: `sort()` and `sorted()`, `shift(by:)` and `shifted(by:)`. When it's naturally a noun, the non-mutating method uses the noun and the mutating one adds the prefix "form": `union(_:)` and `formUnion(_:)`. A non-mutating version is often easiest to write in terms of the mutating one:

```swift
// swiftpl/ch6/interval (continued)
extension Interval {
    func shifted(by delta: Double) -> Interval {
        var copy = self
        copy.shift(by: delta)
        return copy
    }
}
```

A mutating method can even assign a whole new value to `self`, which is the natural way to write state transitions for enums:

```swift
enum TrafficLight {
    case red, green, yellow

    mutating func next() {
        switch self {
        case .red: self = .green
        case .green: self = .yellow
        case .yellow: self = .red
        }
    }
}
```

### 6.2.1. Classes

Class instances are reference types (Section 2.3.2). A variable of class type refers to a shared object, and a method can change the object's stored properties without being marked `mutating`, since it changes the object, not the variable referring to it:

```swift
final class Connection {
    let host: String
    private(set) var bytesSent = 0

    init(host: String) {
        self.host = host
    }

    func send(_ bytes: [UInt8]) {
        // ...write the bytes to the network...
        bytesSent += bytes.count  // no 'mutating' needed
    }
}

let conn = Connection(host: "example.com")
conn.send(Array("hello".utf8))  // fine, although conn is a let
print(conn.bytesSent)  // "5"
```

`final` prevents subclassing. We recommend marking classes `final` unless they're designed to be subclassed: it makes the code easier to reason about and lets the compiler call methods directly instead of through a table.

When should a type be a class rather than a struct? Start with a struct, and use a class when the thing being modeled has *identity*, when everyone holding a reference should see the same changes. A network connection, an open file, a shared cache, and a node in a graph with several incoming edges all have identity. Use a class also when you need `deinit` to release a resource, or when a framework requires subclassing. Large data isn't by itself a reason, since copy-on-write makes copying values cheap.

### 6.2.2. Representing Absence

Swift references can't be `nil` unless their type is optional, so "no value" is always explicit. There are two common ways to give it behavior.

The first is to make emptiness one of the cases of an enum, as the `List` type of Section 4.4 does with its `end` case. Methods on the enum then handle the empty case like any other:

```swift
extension List {
    /// The number of elements in the list.
    var count: Int {
        switch self {
        case .end: 0
        case .node(_, let rest): 1 + rest.count
        }
    }
}

print(List.node("a", .node("b", .end)).count)  // "2"
print(List<String>.end.count)  // "0"
```

Here `switch` is used as an expression, each case producing the property's value directly.

The second is to extend `Optional` itself, giving methods to a value that might be `nil`:

```swift
extension Optional where Wrapped == String {
    /// The string, or "" if there is none.
    var orEmpty: String { self ?? "" }
}

let middleName: String? = nil
print(middleName.orEmpty.isEmpty)  // "true"
```

For everything else, *optional chaining* applies a method only when a value is present: `connection?.send(bytes)` sends only if `connection` isn't `nil`, and `user?.address?.city` is `nil` if any link of the chain is.

## 6.3. Composing Types with Extensions and Protocols

Building a bigger type out of smaller ones is called *composition*, and Swift offers three ways to share code between the bigger type and the smaller ones.

### 6.3.1. Plain Composition

The simplest is to store the smaller value as a property and reach its methods through the property. A note attached to a stretch of a recording, say, has an interval and some text:

```swift
// swiftpl/ch6/annotation (builds on interval)
struct Annotation {
    var span: Interval
    var note: String
}

var a = Annotation(span: Interval(lower: 62, upper: 75), note: "guitar solo")
var b = Annotation(span: Interval(lower: 70, upper: 90), note: "crowd noise")
print(a.span.overlaps(b.span))  // "true"
a.span.shift(by: 30)
print(a.span.overlaps(b.span))  // "false"
```

This is explicit and clear, and it's what most Swift code does. An `Annotation` *has* an interval; it isn't one, and a function expecting an `Interval` won't accept an `Annotation`.

### 6.3.2. Protocol Extensions

When several types share a capability, Swift's preferred approach is to name the capability with a *protocol* and implement the shared behavior once, in a *protocol extension*. Here's a protocol for anything that occupies a span of time:

```swift
// swiftpl/ch6/annotation (continued)
protocol HasSpan {
    var span: Interval { get set }
}

extension HasSpan {
    var duration: Double { span.length }

    func overlaps(_ other: some HasSpan) -> Bool {
        span.overlaps(other.span)
    }

    mutating func shift(by delta: Double) {
        span.shift(by: delta)
    }
}
```

The protocol requires one thing, a readable and writable `span`. The extension then gives *every* conforming type a `duration` property and `overlaps` and `shift` methods, all written in terms of that one requirement. A type acquires them simply by conforming:

```swift
// swiftpl/ch6/annotation (continued)
extension Annotation: HasSpan {}  // its stored 'span' property satisfies the requirement

struct Meeting: HasSpan {
    var span: Interval
    var title: String
    var attendees: [String]
}

var standup = Meeting(span: Interval(lower: 9, upper: 9.25), title: "standup", attendees: ["Ada", "Grace"])
let review = Annotation(span: Interval(lower: 9, upper: 10), note: "code review")
print(standup.overlaps(review))  // "true": a meeting and an annotation, compared directly
standup.shift(by: 1)
print(standup.duration)  // "0.25"
```

`Annotation` already had a stored property named `span` of the right type, so its conformance needs no code at all. A type whose data is arranged differently could satisfy the requirement with a computed property instead.

Protocol extensions are the foundation of *protocol-oriented programming*, the style that runs throughout Swift's standard library. The `Sequence` protocol, for example, requires little more than a way to iterate, and its extensions supply `map`, `filter`, `reduce`, `contains`, `sorted`, `min`, `max`, and dozens of other methods to every conforming type. Chapter 7 explores protocols in depth.

### 6.3.3. Class Inheritance

Classes can also share code through *inheritance*. A subclass inherits its superclass's stored properties and methods, can add more, and can *override* methods; an overridden method is chosen at run time according to the object's actual class:

```swift
class Shape {
    var name: String
    init(name: String) { self.name = name }
    func area() -> Double { 0 }
    func summary() -> String { "\(name) with area \(area())" }
}

final class Square: Shape {
    var side: Double
    init(side: Double) {
        self.side = side
        super.init(name: "square")
    }
    override func area() -> Double { side * side }
}

let shapes: [Shape] = [Shape(name: "point"), Square(side: 3)]
for s in shapes {
    print(s.summary())
}
// point with area 0.0
// square with area 9.0
```

`override` is mandatory, so a method can't override another by accident, and a subclass's initializer must set its own properties and then call `super.init` to set the inherited ones. Swift's full rules for class initialization (designated and convenience initializers, required initializers, two-phase initialization) are more intricate than this example suggests; Section 6.7 covers them.

Inheritance is the right tool for a genuine "is-a" hierarchy whose members share both data and behavior, and for working with frameworks designed around subclassing. For sharing behavior among types in general, protocols with extensions provide the same reuse with less coupling, and they work for structs and enums too.

## 6.4. Method Values and Key Paths

A method is usually selected and called in one expression, as in `meeting.contains(10)`. The two steps can be separated. Selecting a method from an instance without calling it gives a *method value*: a function bound to that instance, which can be called later with just the remaining arguments:

```swift
let workingHours = Interval(lower: 9, upper: 17)
let isDuringWork = workingHours.contains  // a method value of type (Double) -> Bool

let alarms = [6.5, 9.25, 12.0, 18.75, 23.0]
print(alarms.filter(isDuringWork))  // "[9.25, 12.0]"
```

Method values are handy wherever an API wants a function and the behavior needed is "call this method on that instance." Passing `workingHours.contains` directly to `filter` reads better than wrapping it in a closure.

A method can also be referenced through its *type*, as `Interval.contains`. This *unapplied method reference* is a *curried* function: given an instance, it returns a method value bound to that instance:

```swift
let containment = Interval.contains
print(type(of: containment))  // "(Interval) -> (Double) -> Bool"
print(containment(.unit)(0.5))  // "true"
```

That's occasionally useful for applying the same method to many instances. Operators, which are static functions, can be chosen dynamically the same way, which is often clearer than branching inside a loop. Here a report finds either the earliest start or the latest finish in a list of intervals, depending on a flag:

```swift
func extreme(of times: [Double], latest: Bool) -> Double? {
    let pick: (Double, Double) -> Double = latest ? max : min
    guard let first = times.first else { return nil }
    return times.dropFirst().reduce(first, pick)
}

print(extreme(of: schedule.map(\.lower), latest: false) as Any)  // "Optional(9.0)"
print(extreme(of: schedule.map(\.upper), latest: true) as Any)  // "Optional(14.0)"
```

The library functions `min` and `max` are generic, but the type annotation on `pick` tells the compiler which specialization to use.

### 6.4.1. Key Paths

A *key path* is to a property what an unapplied method reference is to a method: a reference to a property that isn't yet tied to an instance. It's written with a backslash, the type, and the path of properties, as in `\Interval.lower` or `\Meeting.span.upper`; when the type can be inferred, it may be omitted, as in `\.lower`.

A key path is applied to an instance with the `[keyPath:]` subscript, and if it leads to a mutable property, it can be used to change it:

```swift
var block = Interval(lower: 13, upper: 14)
let end = \Interval.upper
print(block[keyPath: end])  // "14.0"
block[keyPath: end] = 15
print(block)  // "Interval(lower: 13.0, upper: 15.0)"
```

Key paths are values: they can be stored, passed as arguments, compared with `==`, and used as dictionary keys. A key-path literal can also stand in for a function from a type to one of its properties, which makes code using the standard library's higher-order functions concise:

```swift
let starts = schedule.map(\.lower)  // [9.0, 9.5, 13.0]
let longest = schedule.max(by: { $0.length < $1.length })
let byStart = schedule.sorted(using: KeyPathComparator(\.lower))  // Foundation
```

Key paths are statically typed. `\Interval.upper` has type `WritableKeyPath<Interval, Double>`, so misspelled paths and mismatched types are compile-time errors. Section 4.4 used them with `@dynamicMemberLookup`, and Chapter 12 uses them as a type-safe alternative to setting properties by name at run time.

## 6.5. Example: A Ring Buffer

A *ring buffer*, or circular buffer, holds a fixed number of elements and, once full, makes room for each new element by discarding the oldest. It's the natural structure for keeping "the last *n*" of anything, such as recent log lines, the latest sensor readings for a moving average, or a shell's command history, using a fixed amount of memory however long the program runs.

The implementation keeps the elements in a fixed-size array used circularly: `head` is the position of the oldest element, the others follow it, and positions wrap around from the end of the array to the start. Making it *generic* over the element type (the `<Element>` after its name) lets one implementation hold strings, numbers, or anything else. Generics get a full treatment in Section 7.7:

```swift
// swiftpl/ch6/ringbuffer
/// A RingBuffer holds up to `capacity` of the most recently appended elements.
struct RingBuffer<Element> {
    private var storage: [Element?]
    private var head = 0  // position of the oldest element
    private(set) var count = 0

    init(capacity: Int) {
        precondition(capacity > 0, "capacity must be positive")
        storage = Array(repeating: nil, count: capacity)
    }

    var capacity: Int { storage.count }
    var isFull: Bool { count == capacity }

    /// Appends an element, discarding the oldest one if the buffer is full.
    mutating func append(_ element: Element) {
        storage[(head + count) % capacity] = element
        if isFull {
            head = (head + 1) % capacity  // the oldest element was just overwritten
        } else {
            count += 1
        }
    }

    /// Removes and returns the oldest element, or nil if the buffer is empty.
    mutating func popFirst() -> Element? {
        guard count > 0 else { return nil }
        defer {
            storage[head] = nil
            head = (head + 1) % capacity
            count -= 1
        }
        return storage[head]
    }

    /// The elements, from oldest to newest.
    var elements: [Element] {
        (0..<count).map { storage[(head + $0) % capacity]! }
    }
}
```

The storage is an array of *optionals*, so that unused positions can hold `nil` instead of some arbitrary placeholder value. `append` writes the new element just past the newest one, wrapping around with the remainder operator `%`. If the buffer was already full, that position held the oldest element, which has now been overwritten, so `head` advances. `popFirst` uses `defer` (Section 5.8) to update the bookkeeping *after* the return value has been read. The `!` in `elements` is safe because every position in the occupied range holds a value.

A ring buffer is far more useful if it can be printed, so let's give it a `description`:

```swift
// swiftpl/ch6/ringbuffer (continued)
extension RingBuffer: CustomStringConvertible {
    var description: String {
        "[" + elements.map { "\($0)" }.joined(separator: ", ") + "]"
    }
}
```

Here it keeps the last three commands of a shell session:

```swift
// swiftpl/ch6/ringbuffer (continued)
var history = RingBuffer<String>(capacity: 3)
for command in ["ls", "cd src", "swift build", "swift test"] {
    history.append(command)
}
print(history)  // "[cd src, swift build, swift test]": "ls" was discarded
print(history.popFirst()!)  // "cd src"
history.append("git status")
print(history, history.count)  // "[swift build, swift test, git status] 3"
```

Because `RingBuffer` is a struct whose only storage is an array, it has value semantics for free: `var snapshot = history` makes an independent copy, and later appends to `history` don't affect `snapshot`, with copy-on-write keeping the copy cheap until one of them changes.

**Exercise 6.1:** Add a `last` property that returns the newest element, a `removeAll()` method, and a subscript in which index 0 is the oldest element and `count - 1` the newest.

**Exercise 6.2:** Add `append(contentsOf:)`, which appends every element of a sequence. Make sure it works correctly when the sequence is longer than the buffer's capacity.

**Exercise 6.3:** Add a constrained extension, `extension RingBuffer where Element == Double`, with an `average` property, and use it to print a moving average of the last ten numbers read from the standard input.

**Exercise 6.4:** Add a mutating method that changes the buffer's capacity, keeping the newest elements when shrinking.

## 6.6. Encapsulation

A property or method is *encapsulated* when clients of the type can't reach it, and *information hiding* of this kind is one of the main purposes of methods and types. Swift controls it with access levels, from the most restrictive to the least:

```
private       visible within the enclosing declaration and its extensions in the same file
fileprivate   visible within the same source file
internal      visible within the same module (the default)
package       visible within the modules of the same package (Swift 5.9)
public        visible to any module that imports this one
open          like public, and also allows subclassing and overriding from other modules
```

`RingBuffer` keeps its `storage` and `head` `private`. Clients can append, pop, and read the elements, but they can't reach into the array, so they can't put the buffer into a state its methods don't expect, such as a `head` past the end of the array, or a `count` that disagrees with the occupied positions.

Its `count` illustrates another common need: a property that clients can read but not write. Declaring it `private(set)` makes the getter `internal` (or whatever the property's access level is) and the setter `private`. Other languages often achieve the same effect with a private field and a public getter method. In Swift, that's unnecessary, since a stored property can later be turned into a computed one without affecting the code that uses it.

Encapsulation pays off in three ways. Because only the type's own code can modify its private state, understanding how that state can change means reading only that code. Because clients can't depend on hidden details, the type's author can change its representation, perhaps replacing the array of optionals with a raw buffer for speed, without breaking anyone. And, most important, because every change to the state goes through the type's own methods, those methods can ensure that the state always satisfies the type's invariants.

*Property observers* give a stored property invariants of its own. `willSet` and `didSet` blocks run before and after every assignment, keeping the convenience of property syntax while enforcing a rule:

```swift
// swiftpl/ch6/thermostat
struct Thermostat {
    enum Mode { case heating, cooling, idle }

    /// The target temperature, kept within 10...30 °C.
    var target = 20.0 {
        didSet { target = min(max(target, 10), 30) }
    }
    private(set) var mode = Mode.idle

    mutating func update(current: Double) {
        if current < target - 0.5 {
            mode = .heating
        } else if current > target + 0.5 {
            mode = .cooling
        } else {
            mode = .idle
        }
    }
}

var t = Thermostat()
t.target = 45
print(t.target)  // "30.0": clamped
t.update(current: 18)
print(t.mode)  // "heating"
```

Assigning to a property inside its own `didSet`, as here, doesn't trigger the observer again.

Not every type needs to hide anything. `Interval` exposes its bounds, and that's fine, because any pair of numbers is a valid interval, so there's no invariant to protect, and its bounds *are* its meaning. (Should `lower` be allowed to exceed `upper`? If you decide not, that's an invariant, and the bounds should become `private(set)` with an initializer that checks them.) Deciding what to expose is a judgment about which details are essential to a type and which are incidental.

For code in libraries, there's a further consideration: anything `public` is a promise. Once other modules depend on it, changing or removing it breaks them. Swift's default of `internal` means nothing becomes public by accident, and each public declaration should be a deliberate part of the module's interface.

**Exercise 6.5:** Change `Interval` so that `lower <= upper` always holds: make the bounds `private(set)`, add a failable initializer that rejects reversed bounds, and adjust `shift(by:)` and the extensions accordingly. Which earlier examples still compile?

**Exercise 6.6:** Write a `BankBalance` type that stores its value as an integer number of cents, can be read as a `Decimal`, and can be changed only by `deposit(_:)` and `withdraw(_:) throws`, which refuses withdrawals that would make the balance negative.

## 6.7. Classes and Inheritance

Most of this chapter has been about structs, because most Swift types are structs. But classes, and the inheritance they support, are a full part of the language, and some problems call for them: modeling things with identity, sharing a resource that must be released exactly once, and using frameworks built around subclassing. Section 6.3.3 gave a short example. This section covers the rules properly.

### 6.7.1. A Class Hierarchy

Inheritance works best when a base class defines the overall shape of an operation and subclasses fill in the details, a design known as the *template method* pattern. As an example, here's a small family of document exporters. The base class fixes the order in which a document is produced (header, paragraphs, footer), and subclasses decide what each part looks like:

```swift
// swiftpl/ch6/exporters
class Exporter {
    let title: String

    /// The designated initializer: sets every stored property.
    init(title: String) {
        self.title = title
    }

    /// A convenience initializer: delegates to another initializer of this class.
    convenience init() {
        self.init(title: "Untitled")
    }

    /// Exports the document. Subclasses customize the parts, not their order.
    final func export(_ paragraphs: [String]) -> String {
        var out = header()
        for p in paragraphs {
            out += paragraph(p)
        }
        return out + footer()
    }

    func header() -> String { title + "\n\n" }
    func paragraph(_ text: String) -> String { text + "\n\n" }
    func footer() -> String { "" }
}

final class MarkdownExporter: Exporter {
    override func header() -> String { "# \(title)\n\n" }
}

final class HTMLExporter: Exporter {
    let stylesheet: String?

    init(title: String, stylesheet: String?) {
        self.stylesheet = stylesheet  // phase 1: this class's own properties first...
        super.init(title: title)  // ...then the superclass's
        // phase 2: self is fully initialized, so methods may be called here
    }

    override convenience init(title: String) {
        self.init(title: title, stylesheet: nil)
    }

    override func header() -> String {
        var head = "<html><head><title>\(title)</title>"
        if let stylesheet {
            head += "<link rel=\"stylesheet\" href=\"\(stylesheet)\">"
        }
        return head + "</head><body>\n<h1>\(title)</h1>\n"
    }

    override func paragraph(_ text: String) -> String { "<p>\(text)</p>\n" }
    override func footer() -> String { "</body></html>\n" }
}
```

A variable of type `Exporter` can refer to an instance of any subclass, and calls to overridden methods go to the subclass's version, chosen while the program runs according to the object's actual class:

```swift
// swiftpl/ch6/exporters (continued)
let exporters: [Exporter] = [
    Exporter(title: "Notes"),
    MarkdownExporter(title: "Notes"),
    HTMLExporter(),  // inherited convenience initializer: title "Untitled"
]
for exporter in exporters {
    print(exporter.export(["First point.", "Second point."]))
}
```

```
Notes

First point.

Second point.

# Notes

First point.

Second point.

<html><head><title>Untitled</title></head><body>
<h1>Untitled</h1>
<p>First point.</p>
<p>Second point.</p>
</body></html>
```

(A real HTML exporter would escape its text, as Section 4.6 showed.) Here's what each part of the declarations does.

`override` is required on every method, property, or initializer that replaces an inherited one. You can't override by accident by happening to choose an existing name, and you can't misspell an override without the compiler noticing that nothing is being overridden.

`super.method()` calls the superclass's version of a method from an override, for when a subclass wants to extend inherited behavior rather than replace it.

`final` on a method, like `export`, forbids overriding it. Here it guarantees that every exporter produces its parts in the same order. `final` on a class, like `MarkdownExporter`, forbids subclassing it at all. Besides documenting intent, `final` lets the compiler call methods directly instead of looking them up at run time.

*Dynamic dispatch* is how an overridden method is found: each class has a table of its overridable methods, and a call through a reference looks up the right entry for the object's actual class. That's slightly slower than a direct call, and it prevents inlining, which is another reason to mark classes `final` unless they're meant to be subclassed.

Static methods and properties can't be overridden. A class can declare `class func` or `class var` instead of `static` to make a type-level member overridable.

### 6.7.2. Initialization

A class's initializers must leave every stored property, including inherited ones, with a value before the object can be used. Swift enforces this with rules that are worth learning precisely, since the compiler applies them precisely.

A class has two kinds of initializers. A *designated* initializer, the ordinary kind, initializes all the properties the class itself introduces and then calls a designated initializer of its superclass. A *convenience* initializer, marked `convenience`, doesn't initialize anything directly; it must call another initializer of the same class, `self.init(...)`, eventually reaching a designated one. In short: designated initializers delegate *up*, and convenience initializers delegate *across*.

Initialization happens in two phases. In *phase 1*, each class in the hierarchy, from the most derived up to the root, initializes the stored properties it introduced. That's why `HTMLExporter`'s initializer sets `stylesheet` *before* calling `super.init`. Until phase 1 completes, the object isn't fully formed, so the compiler forbids calling methods, reading inherited properties, or otherwise using `self`. Once the root class's initializer has run, *phase 2* begins: control returns down the chain, and each initializer may now use `self` freely, calling methods and customizing inherited properties. This ordering guarantees that no method, not even an overridden one called from a superclass's initializer, ever sees an uninitialized property. Languages that skip such a rule allow exactly that bug.

Subclasses don't automatically inherit their superclass's initializers, since an inherited initializer wouldn't know how to set the subclass's new properties. Two rules say when they do. If a subclass adds no designated initializers of its own (and gives all its new properties default values), it inherits all of the superclass's designated initializers, as `MarkdownExporter` inherits `init(title:)`. And if a subclass provides every one of its superclass's designated initializers, by inheriting or overriding them, it also inherits all of the superclass's convenience initializers. `HTMLExporter` overrides `init(title:)` (as a convenience initializer that supplies a default stylesheet), so it inherits `Exporter`'s convenience `init()`, and `HTMLExporter()` works.

An initializer marked `required` must be implemented by every subclass. That matters when code creates instances through a type that might be a subclass, for example in a factory method that returns `Self`:

```swift
class Plugin {
    let name: String

    required init(name: String) {
        self.name = name
    }

    class func make(named name: String) -> Self {
        Self(name: name)  // works only because every subclass must have init(name:)
    }
}
```

Protocols that require an initializer impose the same obligation: a non-final class conforming to such a protocol must mark the implementation `required`, so that its subclasses conform too.

### 6.7.3. Deinitialization

A class can define a `deinit`, which runs when the last strong reference to an instance disappears (Section 2.3.4). Deinitializers are called automatically, in the reverse order of initialization: the subclass's `deinit` runs first, then its superclass's, up to the root:

```swift
class Resource {
    deinit { print("Resource released") }
}

final class FileResource: Resource {
    deinit { print("FileResource closing file") }
}

do {
    _ = FileResource()
}
// FileResource closing file
// Resource released
```

A `deinit` can't be called directly, takes no parameters, and has full access to the instance's properties, so it can release whatever the object owns.

### 6.7.4. When to Use Inheritance

Inheritance couples a subclass tightly to its superclass. The subclass depends on which methods the superclass calls, in what order, and from where, which is why changes to a base class can break subclasses in surprising ways. Swift's defaults reflect that caution: classes outside their module can't be subclassed unless they're `open` (Section 10.5), and the language offers protocols with default implementations (Section 6.3.2) as a looser form of sharing.

Inheritance is the right tool when there's a genuine "is-a" relationship and subclasses share both data and the overall shape of their behavior, as in the template method pattern above, and when you're working with frameworks designed around subclassing. For sharing behavior among otherwise unrelated types, prefer protocols. For reusing an implementation, prefer composition, storing one value inside another (Section 6.3.1).

**Exercise 6.7:** Add a `PlainTextExporter` subclass that wraps paragraphs at 72 characters, overriding only `paragraph(_:)`.

**Exercise 6.8:** Give `Exporter` a designated initializer that takes the title and an `author: String?`, and keep `init(title:)` as a convenience initializer. Which of the subclasses need changes to keep compiling, and why?

**Exercise 6.9:** Rewrite the exporters with a protocol, `Exporting`, with default implementations of `header()`, `paragraph(_:)`, and `footer()` in an extension, and structs instead of classes. What do you gain and lose? (Hint: what happens if a default implementation of `export` calls `header()`?)

## 6.8. Property Wrappers

Some patterns recur in property declarations: clamping a number into a range, trimming whitespace from a string, recording a property's history, logging every change. Writing the getter and setter logic by hand for every such property is repetitive, and the repetition hides the intent. A *property wrapper* packages the logic once, as a type, and lets any property adopt it with an attribute.

### 6.8.1. Defining a Property Wrapper

A property wrapper is a struct (or class or enum) marked `@propertyWrapper` that has a property named `wrappedValue`. Here's one that trims leading and trailing whitespace from any string stored in it:

```swift
// swiftpl/ch6/wrappers
import Foundation

@propertyWrapper
struct Trimmed {
    private var value = ""

    var wrappedValue: String {
        get { value }
        set { value = newValue.trimmingCharacters(in: .whitespacesAndNewlines) }
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue  // trims the initial value too
    }
}

struct SignUp {
    @Trimmed var name: String
    @Trimmed var email: String
}

var form = SignUp(name: "  Ada Lovelace ", email: "ada@example.com\n")
print("[\(form.name)] [\(form.email)]")  // "[Ada Lovelace] [ada@example.com]"
form.name = "\tGrace Hopper  "
print("[\(form.name)]")  // "[Grace Hopper]"
```

The attribute `@Trimmed` tells the compiler to store the property inside a `Trimmed` value and route all access through it. In effect, it rewrites the declaration of `name` into this:

```swift
private var _name: Trimmed
var name: String {
    get { _name.wrappedValue }
    set { _name.wrappedValue = newValue }
}
```

The struct's memberwise initializer still takes plain `String`s, which it passes to `init(wrappedValue:)`. To users of `SignUp`, `name` is an ordinary `String` property that happens never to have surrounding whitespace.

### 6.8.2. Wrapper Arguments

A property wrapper can take arguments, written after the attribute's name, which are passed to its initializer along with the property's initial value. This wrapper keeps a value within a range:

```swift
// swiftpl/ch6/wrappers (continued)
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    let range: ClosedRange<Value>

    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }

    var wrappedValue: Value {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

struct Speaker {
    @Clamped(0...11) var volume = 5
    @Clamped(-1.0...1.0) var balance = 0.0
}

var s = Speaker()
s.volume = 20
s.balance = -3
print(s.volume, s.balance)  // "11 -1.0"
```

`Clamped` is generic, so it works for any `Comparable` type; the compiler infers `Value` from the property's type. The declaration `@Clamped(0...11) var volume = 5` becomes a call to `Clamped(wrappedValue: 5, 0...11)`.

### 6.8.3. Projected Values

A wrapper can expose additional functionality through a *projected value*: a property named `projectedValue`, which users reach by prefixing the wrapped property's name with `$`. This wrapper remembers every value a property has held, and projects the history:

```swift
// swiftpl/ch6/wrappers (continued)
@propertyWrapper
struct History<Value> {
    private var values: [Value]

    init(wrappedValue: Value) {
        values = [wrappedValue]
    }

    var wrappedValue: Value {
        get { values.last! }
        set { values.append(newValue) }
    }

    /// Every value the property has held, oldest first.
    var projectedValue: [Value] { values }
}

struct Thermostat {
    @History var target = 20.0
}

var t = Thermostat()
t.target = 21
t.target = 19.5
print(t.target)  // "19.5"
print(t.$target)  // "[20.0, 21.0, 19.5]"
```

`t.target` is the current value, a `Double`; `t.$target` is the projected value, here an array of `Double`s. A projected value can be of any type, chosen by the wrapper's author. SwiftUI's `@State` uses one to give access to a *binding*, a reference that lets another view read and write the same state; ArgumentParser's `@Option` uses wrappers to attach parsing information to properties, which the library reads back through reflection (Section 12.7.1).

### 6.8.4. Using Wrappers Well

Property wrappers can be applied to stored properties of structs, classes, and enums, and to local variables. They can't be applied to computed properties or used in protocol requirements, and a wrapped property can't also be `lazy`.

Their power is also their risk: a wrapper changes what assignment *means*, invisibly at the point of use. `s.volume = 20` followed by `print(s.volume)` printing `11` is surprising unless you've seen the declaration. Wrappers work best for behavior that's simple, predictable, and clearly named, such as clamping, trimming, or recording, and for frameworks whose wrappers are widely understood. Complicated business logic is usually clearer as an ordinary method or a property observer (Section 6.6).

**Exercise 6.10:** Write a `@Lowercased` property wrapper for strings. Then try to apply both wrappers to one property, as `@Trimmed @Lowercased var email: String`. When wrappers are composed like this, the outer wrapper wraps the *inner wrapper*, not the string. Why doesn't that compile with these two wrappers, and how could you change them so that it does?

**Exercise 6.11:** Give `Clamped` a projected value that reports whether the most recent assignment had to be clamped, so that callers can write `if s.$volume { print("volume limited") }`.

**Exercise 6.12:** Write a `@Logged` wrapper that prints a line whenever the property changes, showing the old and new values. Why might you want it to take the property's name as an argument?

## 6.9. Automatic Reference Counting

Section 2.3.4 introduced automatic reference counting (ARC), the way Swift manages the memory of class instances, and actors too. Each instance keeps a count of the *strong* references to it. Storing a reference in a variable, property, collection, or closure capture increments the count; overwriting or dropping one decrements it; and when the count reaches zero, the instance's `deinit` runs (Section 6.7.3) and its memory is freed. There's no garbage collector and no collection pause. Values of struct and enum types aren't counted at all, although a struct that contains references to class instances holds those references strongly, like any other variable.

For most code, ARC is invisible. It matters in two situations: when objects refer to each other, and when the moment an object is freed matters.

### 6.9.1. Reference Cycles

Here's a document that can be shown in several windows. The document keeps track of its windows, and each window refers back to its document:

```swift
// swiftpl/ch6/documents
final class Document {
    let title: String
    var windows: [Window] = []

    init(title: String) { self.title = title }
    deinit { print("closing document \(title)") }

    func openWindow() {
        windows.append(Window(document: self))
    }
}

final class Window {
    let document: Document  // strong: completes a cycle
    init(document: Document) { self.document = document }
    deinit { print("closing a window") }
}

do {
    let report = Document(title: "Report")
    report.openWindow()
    report.openWindow()
}
print("done")
```

The program prints only `done`. When the `do` block ends, the constant `report` goes away, but the document's count doesn't reach zero, because each window still holds a strong reference to it, and the windows' counts don't reach zero, because the document's array holds them. No `deinit` ever runs, and the three objects stay in memory, unreachable, until the program exits. This is a *reference cycle*, and it's the one kind of leak that ARC can't prevent on its own.

The fix is to decide which side owns which. A document owns its windows; a window merely refers to the document it shows. The reference back to the owner should therefore not keep it alive. Swift offers two kinds of non-owning reference:

- A `weak` reference doesn't increment the count, and is set to `nil` automatically when its object is freed. It must be a `var` of optional type.
- An `unowned` reference doesn't increment the count either, but it's non-optional, because it promises that the object will outlive the reference. Using an `unowned` reference after its object has been freed is a programming error, and traps (Section 5.9).

A window can't exist without its document, so `unowned` expresses the relationship exactly:

```swift
final class Window {
    unowned let document: Document
    init(document: Document) { self.document = document }
    deinit { print("closing a window") }
}
```

Now the program prints:

```
closing document Report
closing a window
closing a window
done
```

When `report` disappears, the document's count reaches zero. Its `deinit` runs, then its stored properties are destroyed, releasing the array, which releases the windows.

Choose `weak` when the referenced object can legitimately disappear first and the code holding the reference should cope, as with a delegate, an observer, or a cache entry. Choose `unowned` when the lifetimes are tied, as here, and a dangling reference would be a bug. When in doubt, `weak` is the safe choice: its worst case is a `nil` you have to handle rather than a trap.

### 6.9.2. Closures and Capture Lists

Closures are reference types, and a closure holds strong references to the class instances it captures. An object that stores a closure that uses `self` therefore creates a cycle, from the object to the closure and back:

```swift
// swiftpl/ch6/ticker
final class Ticker {
    var count = 0
    var onTick: (() -> Void)?

    init() {
        onTick = { self.count += 1 }  // cycle: self -> onTick -> self
    }
    deinit { print("ticker freed") }
}
```

This is the situation described in Section 5.6.1, and a *capture list* breaks the cycle. Capture lists accept the same `weak` and `unowned` modifiers as properties, with the same meanings. Since `onTick` belongs to the ticker, it can't be called after the ticker is gone, so `unowned` fits:

```swift
onTick = { [unowned self] in self.count += 1 }
```

A capture list can also capture a *value* instead of the object it came from. `[title = self.title]` copies the title into the closure when the closure is created, so the closure doesn't need `self` at all. That's often the cleanest fix, when the closure needs only a property or two.

Not every closure that mentions `self` causes a cycle. A non-escaping closure, like the argument to `map` or `sorted(by:)`, is gone by the time the call returns. An escaping closure that the object doesn't store, directly or indirectly, such as the body of a `Task` that finishes, keeps the object alive only until the closure itself is released. That might delay the object's `deinit`, which is sometimes exactly what's wanted, but it doesn't leak. The cycles to look for are closures stored in properties of the objects they capture, or in something those objects own, such as a timer or a notification observer.

### 6.9.3. When Objects Are Freed

Because counting is deterministic, a `deinit` runs as soon as the last strong reference disappears, which makes it a reasonable place to release resources such as file descriptors and locks. But "the last strong reference disappears" means when the reference is last *used*, not necessarily at the end of the variable's scope. The optimizer is free to release an object right after its last use:

```swift
do {
    let session = Session()
    let token = session.token
    send(token)  // session may already have been freed here
}
```

Usually this doesn't matter. It does matter when an object's `deinit` undoes something that other code still depends on, such as a lock file that another process checks, or a handle that a C library uses (Chapter 13). `withExtendedLifetime(session) { ... }` guarantees that `session` stays alive until the closure returns. Better still, give such resources an explicit `close()` method, and treat `deinit` as a safety net.

Reference counting has a cost. Each increment and decrement is an atomic operation, since references can be shared between threads, and code that copies references in a tight loop can spend noticeable time on them. The optimizer removes many of these operations, and profiling (Section 11.5) shows when the rest matter. Value types avoid the cost entirely, which is one more reason Swift code starts with structs.

To find leaks, Xcode's memory graph debugger shows the objects alive in a running program and the references between them, and the Instruments Leaks template reports unreachable cycles. A test can check for leaks directly, by holding only a weak reference to an object and expecting it to be `nil` once the strong references are gone:

```swift
@Test func documentIsFreed() {
    weak var weakDocument: Document?
    do {
        let document = Document(title: "Test")
        document.openWindow()
        weakDocument = document
    }
    #expect(weakDocument == nil)
}
```

**Exercise 6.13:** Build a three-level tree from the `TreeNode` type of Section 2.3.4 and write a test like `documentIsFreed` that checks that the whole tree is freed. Then make `parent` a strong reference and confirm that the test fails.

**Exercise 6.14:** Write a program that keeps a strong reference to one of a document's windows after the document itself has been freed, and then reads the window's `document` property. What happens when `Window` holds `unowned let document: Document`? Change it to `weak var document: Document?`, and decide what a window should display when its document is gone.

**Exercise 6.15:** Delegate properties are conventionally `weak`. Write a `Downloader` class with a `delegate` property of protocol type that reports progress, and explain why the protocol must be declared `protocol DownloaderDelegate: AnyObject` for the property to be `weak`.
