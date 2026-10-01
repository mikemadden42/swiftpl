# 6. Methods

A *method* is a function that belongs to a type. Rather than writing `length(of: interval)`, you write `interval.length`; rather than `shift(&interval, by: 2)`, you write `interval.shift(by: 2)`. The difference looks cosmetic, but it changes how code is organized: the operations on a type are grouped with the type, a reader can discover them by typing a dot, and the type's author can keep its representation private while exposing only the operations that keep it valid. That combination of data with the operations that maintain it is the core of what's usually called *object-oriented programming*.

Swift supports object-oriented programming in the classic style, with classes, inheritance, and overriding. But its everyday style differs from Java's or C++'s in two important ways. First, methods aren't reserved for classes: structs and enums have them too, and most Swift types are structs. Second, behavior is shared between types mostly through *protocols* and *extensions* rather than through inheritance. This chapter covers methods and the related ideas of extensions, mutation, composition, and encapsulation. Chapter 7 then takes up protocols.

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

`override` is mandatory, so a method can't override another by accident, and a subclass's initializer must set its own properties and then call `super.init` to set the inherited ones. Swift's full rules for class initialization (designated and convenience initializers, required initializers, two-phase initialization) are considerably more intricate than this example suggests, which is one reason everyday Swift favors structs and protocols.

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
extension RingBuffer: CustomStringConvertible {
    var description: String {
        "[" + elements.map { "\($0)" }.joined(separator: ", ") + "]"
    }
}
```

Here it keeps the last three commands of a shell session:

```swift
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
