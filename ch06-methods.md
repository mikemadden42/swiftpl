# 6. Methods

Since the early 1990s, object-oriented programming (OOP) has been the dominant programming paradigm in industry and education, and nearly all widely used languages developed since then have included support for it. Swift is no exception.

Although there is no universally accepted definition of object-oriented programming, for our purposes, an *object* is simply a value or variable that has methods, and a *method* is a function associated with a particular type. An object-oriented program is one that uses methods to express the properties and operations of each data structure so that clients need not access the object's representation directly.

Swift supports classes with inheritance, in the tradition of Smalltalk, Objective-C, and Java. But you'll find that idiomatic Swift uses classes much less than those languages do. Methods may be declared on structs and enums as well as classes, and most Swift types are structs. Behavior is shared among types not primarily by inheritance but through protocols and extensions, and much of the expressive power of Swift's type system comes from the interplay between value types, methods, and protocols.

In this chapter, we'll show how to define and use methods effectively. We'll also cover two key principles of object-oriented programming, *encapsulation* and *composition*.

## 6.1. Method Declarations

A method is declared inside the body of a type, or in an *extension* of the type. Inside the method, the instance on which it was called is available as `self`. Here's our first example, a method on a simple plane geometry type:

```swift
// swiftpl/ch6/geometry
import Foundation

struct Point {
    var x, y: Double

    /// Same thing as distance(_:_:), but as a method of the Point type.
    func distance(to q: Point) -> Double {
        hypot(q.x - x, q.y - y)
    }
}

/// Traditional function.
func distance(_ p: Point, _ q: Point) -> Double {
    hypot(q.x - p.x, q.y - p.y)
}
```

Within a method, the instance's properties and other methods can be used without the `self.` prefix: `x` means `self.x`. You need to write `self` explicitly only to disambiguate, as when a parameter has the same name as a property, and in escaping closures in classes, as we saw in Section 5.6.

There's no conflict between the two declarations of functions called `distance` above. The first declares a module-level function. The second declares a method of the type `Point`, so its name is really `Point.distance(to:)`.

In a method call, the instance comes before the method name, separated by a dot:

```swift
let p = Point(x: 1, y: 2)
let q = Point(x: 4, y: 6)
print(distance(p, q))  // "5.0", function call
print(p.distance(to: q))  // "5.0", method call
```

The expression `p.distance(to:)` is called a *selector*, because it selects the appropriate method for the instance `p` of type `Point`. Selectors are also used to select properties of struct types, as in `p.x`. Since methods and properties inhabit the same namespace, declaring a method called `x` in the struct type `Point` would be an error, since it would be ambiguous.

Since each type has its own namespace for methods, we can use the name `distance` for other methods so long as they belong to different types. Let's define a type `Path` that represents a sequence of line segments and give it a `distance` method too. Rather than wrapping an array in a new struct, we'll declare `Path` as a type alias for `[Point]`, and add the method to arrays of points with a *constrained extension*:

```swift
/// A Path is a journey connecting the points with straight lines.
typealias Path = [Point]

extension Array where Element == Point {
    /// Returns the distance traveled along the path.
    func distance() -> Double {
        var sum = 0.0
        for i in indices.dropFirst() {
            sum += self[i - 1].distance(to: self[i])
        }
        return sum
    }
}
```

An *extension* adds new members to an existing type, which can be one you declared, one from another module, or one from the standard library. The `where` clause restricts this extension to arrays whose elements are `Point`s, so `distance()` is available on `[Point]` but not on `[Int]`. Inside the method, `self` is the array, `indices` is its range of valid indices, and `self[i]` is an element.

This is a key difference from Go. In Go, you can declare methods only on types defined in your own package, so to give a slice of points a method you must declare a new named type `Path`. In Swift, you can extend *any* type, even `Array` and `Int`, with methods, computed properties, initializers, subscripts, nested types, and protocol conformances. The only restrictions are that an extension can't add stored properties (they would change the type's layout) and can't override existing members.

Let's call the new method to compute the perimeter of a right triangle:

```swift
let perim: Path = [
    Point(x: 1, y: 1),
    Point(x: 5, y: 1),
    Point(x: 5, y: 4),
    Point(x: 1, y: 1),
]
print(perim.distance())  // "12.0"
```

In the two calls to methods named `distance` above, the compiler determines which function to call based on both the method name and the type of the instance. In the first, `self[i - 1]` has type `Point`, so `Point.distance(to:)` is called; in the second, `perim` has type `[Point]`, so the array method is called.

Should a value with no arguments be a method or a *computed property*? A computed property is declared with `var` and a body, and looks like a stored property to its users:

```swift
extension Array where Element == Point {
    var length: Double { distance() }
}

print(perim.length)  // "12.0"
```

The Swift convention is that a computed property should be cheap (roughly O(1)), have no side effects, and describe a characteristic of the value, like `count`, `isEmpty`, or `description`. Anything that does significant work or has side effects should be a method. By that standard, `distance()`, which walks the whole path, is better as a method.

Types can also have *static* members, which belong to the type itself rather than to an instance. We used static constants in Section 2.6; static methods are often used as factory functions:

```swift
extension Point {
    static let origin = Point(x: 0, y: 0)

    static func polar(r: Double, theta: Double) -> Point {
        Point(x: r * cos(theta), y: r * sin(theta))
    }
}

let p = Point.polar(r: 2, theta: .pi / 2)
print(p.distance(to: .origin))  // "2.0"
```

Notice the implicit member expression `.origin`. Wherever the type is known from context, a static member of that type can be named with a leading dot.

## 6.2. Mutating Methods and Reference Types

Because a struct is a value, its methods can't modify `self` unless they're explicitly marked `mutating`. A mutating method receives `self` as an implicit `inout` parameter:

```swift
extension Point {
    mutating func scale(by factor: Double) {
        x *= factor
        y *= factor
    }
}
```

The name of this method is `Point.scale(by:)`, and calling it modifies the variable on which it was called:

```swift
var r = Point(x: 1, y: 2)
r.scale(by: 2)
print(r)  // "Point(x: 2.0, y: 4.0)"
```

A mutating method can be called only on a variable, because only a variable can be modified:

```swift
let p = Point(x: 1, y: 2)
p.scale(by: 2)  // compile error: cannot use mutating member on immutable value: 'p' is a 'let' constant
Point(x: 1, y: 2).scale(by: 2)  // compile error: cannot use mutating member on immutable value
```

So `mutating` plays the role that a pointer receiver does in Go, and the `let`/`var` distinction plays the role of whether you can take an address. But the Swift rules are stricter and more visible. A non-mutating method is guaranteed not to change the value, which the compiler enforces. And a `let` struct is guaranteed never to change at all.

The API Design Guidelines recommend naming mutating and non-mutating pairs consistently. When an operation is naturally described by a verb, use the imperative verb for the mutating method and the "-ed" or "-ing" form for the non-mutating one that returns a new value: `sort()` and `sorted()`, `reverse()` and `reversed()`, `scale(by:)` and `scaled(by:)`. When an operation is naturally described by a noun, use the noun for the non-mutating method and the prefix "form" for the mutating one: `union(_:)` and `formUnion(_:)`.

```swift
extension Point {
    func scaled(by factor: Double) -> Point {
        var copy = self
        copy.scale(by: factor)
        return copy
    }
}
```

A mutating method can even replace `self` completely: `self = Point.origin`. This is especially useful in enums, where a mutating method can change the case:

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

Classes are reference types. A variable of class type holds a reference to an object, and methods of a class may modify the object's stored properties without being marked `mutating`, because they modify the shared object, not the variable that refers to it:

```swift
final class Counter {
    private(set) var count = 0

    func increment() {
        count += 1  // no 'mutating' needed
    }
}

let c = Counter()
c.increment()  // fine, even though c is a let
print(c.count)  // "1"
```

`final` means that the class can't be subclassed. We recommend marking classes `final` unless they're specifically designed for inheritance, since it makes the code easier to reason about and lets the compiler call methods directly rather than through a dispatch table.

When should you use a class rather than a struct? Use a struct by default. Use a class when the thing you're modeling has *identity*, that is, when two references to "the same" object should see each other's changes: a network connection, a file handle, a cache shared across a program, a node in a graph that several other nodes point to. Use a class, too, when you need inheritance to interoperate with a framework designed around it, or when you need a `deinit`. Some Swift programmers would add "when the value is very large," but copy-on-write makes that less compelling than it seems.

### 6.2.2. Representing Absence

In Go, `nil` is a valid value for a pointer receiver, so a method can be called on a `nil` linked list to mean "the empty list." Swift's references are never `nil` unless their type is optional, so the question doesn't arise in the same way. Instead, there are two common designs.

The first is to make emptiness a case of an enum, as the `List` type of Section 4.4 does with its `end` case:

```swift
/// An IntList is a linked list of integers.
indirect enum IntList {
    case empty
    case cons(Int, IntList)

    /// Returns the sum of the list elements.
    var sum: Int {
        switch self {
        case .empty: 0
        case let .cons(value, tail): value + tail.sum
        }
    }
}

let list = IntList.cons(1, .cons(2, .cons(3, .empty)))
print(list.sum)  // "6"
print(IntList.empty.sum)  // "0"
```

Here the `switch` is used as an expression, so each case produces the property's value without a `return`.

The second is to extend `Optional` itself, which gives methods to a value that might be `nil`. For example, it's often convenient to treat a `nil` string as empty:

```swift
extension Optional where Wrapped == String {
    var orEmpty: String { self ?? "" }
}

let name: String? = nil
print(name.orEmpty.count)  // "0"
```

Optional chaining handles the more general case: `node?.next?.value` evaluates to `nil` if any link in the chain is `nil`, and calls methods only on values that are present.

## 6.3. Composing Types with Extensions and Protocols

Go composes types by *struct embedding*: declaring a `ColoredPoint` with an anonymous `Point` field promotes all of `Point`'s methods to `ColoredPoint`, so a colored point can be scaled and measured without any forwarding code. As we saw in Section 4.4, Swift has no direct equivalent. It offers three alternatives, each suited to different circumstances.

### 6.3.1. Plain Composition

The simplest is just to store a `Point` in a named property, and access its methods through the property:

```swift
// swiftpl/ch6/coloredpoint
struct Color {
    var r, g, b, a: UInt8
    static let red = Color(r: 255, g: 0, b: 0, a: 255)
    static let blue = Color(r: 0, g: 0, b: 255, a: 255)
}

struct ColoredPoint {
    var point: Point
    var color: Color
}

var p = ColoredPoint(point: Point(x: 1, y: 1), color: .red)
var q = ColoredPoint(point: Point(x: 5, y: 4), color: .blue)
print(p.point.distance(to: q.point))  // "5.0"
p.point.scale(by: 2)
q.point.scale(by: 2)
print(p.point.distance(to: q.point))  // "10.0"
```

This is explicit and clear, and it's what most Swift programmers would write. Note that `ColoredPoint` *has a* `Point`; it isn't one. A function that takes a `Point` can't be passed a `ColoredPoint`:

```swift
p.point.distance(to: q)  // compile error: cannot convert value of type 'ColoredPoint' to expected argument type 'Point'
```

### 6.3.2. Protocol Extensions

When several types share some behavior, the Swift approach is to describe what they have in common with a *protocol*, and implement the behavior once in a *protocol extension*. Here's a protocol for anything that has a position:

```swift
protocol Positioned {
    var position: Point { get set }
}

extension Positioned {
    func distance(to other: some Positioned) -> Double {
        position.distance(to: other.position)
    }

    mutating func scale(by factor: Double) {
        position.scale(by: factor)
    }
}
```

The protocol requires a single property, `position`, that can be read and written. The extension then provides methods to *every* type that conforms to `Positioned`, implemented in terms of that requirement. A type gets all of these methods simply by conforming:

```swift
extension ColoredPoint: Positioned {
    var position: Point {
        get { point }
        set { point = newValue }
    }
}

struct Sprite: Positioned {
    var position: Point
    var image: String
}

var p = ColoredPoint(point: Point(x: 1, y: 1), color: .red)
let s = Sprite(position: Point(x: 4, y: 5), image: "ship.png")
print(p.distance(to: s))  // "5.0"
p.scale(by: 2)
```

`Sprite` satisfies the requirement with a stored property of the right name and type, so its conformance needs no other code. `ColoredPoint` satisfies it with a computed property. Now *any* two positioned things can measure the distance between them, whatever their types.

Protocol extensions are the heart of *protocol-oriented programming*, a style that Swift's designers introduced in 2015 and that permeates the standard library. Sequence algorithms like `map`, `filter`, `contains`, and `sorted` are all defined in extensions of the `Sequence` protocol, which requires only a way to make an iterator. Any type that provides one gets dozens of algorithms for free. We'll study protocols in depth in Chapter 7.

### 6.3.3. Class Inheritance

Finally, Swift classes support single inheritance. A subclass inherits the stored properties and methods of its superclass, can add its own, and can *override* methods, which are then dispatched dynamically according to the object's run-time class:

```swift
class Shape {
    var origin: Point
    init(origin: Point) { self.origin = origin }
    func area() -> Double { 0 }
    func describe() -> String { "shape with area \(area())" }
}

final class Circle: Shape {
    var radius: Double
    init(origin: Point, radius: Double) {
        self.radius = radius
        super.init(origin: origin)
    }
    override func area() -> Double { .pi * radius * radius }
}

let shapes: [Shape] = [Shape(origin: .origin), Circle(origin: .origin, radius: 1)]
for s in shapes {
    print(s.describe())
}
// shape with area 0.0
// shape with area 3.141592653589793
```

The `override` keyword is required, so you can't accidentally override a method by giving it the same name, and `super.init` must be called to initialize the superclass's properties. Swift's class initialization rules are notoriously intricate (designated and convenience initializers, two-phase initialization, required initializers), which is one more reason the community leans toward structs and protocols.

Inheritance is appropriate when you're working with a framework designed around it, or when you have a genuine "is-a" hierarchy whose members share both data and behavior. For most code, protocols with extensions give the same reuse with less coupling, and they work with structs and enums too.

### 6.3.4. Example: A Cache with a Lock

Go programs sometimes embed a `sync.Mutex` in an unnamed struct so that the struct's `Lock` and `Unlock` methods protect its other fields. Swift's `Mutex` takes a different approach: rather than sitting beside the data it protects, it *contains* it, and the only way to reach the data is through `withLock`:

```swift
import Synchronization

let cache = Mutex<[String: String]>([:])

func lookup(_ key: String) -> String? {
    cache.withLock { mapping in
        mapping[key]
    }
}

func store(_ key: String, _ value: String) {
    cache.withLock { $0[key] = value }
}
```

This design makes it impossible to forget to take the lock, since the data simply isn't accessible otherwise. We'll return to `Mutex` and its alternatives in Chapter 9.

## 6.4. Method Values and Key Paths

Usually we select and call a method in the same expression, as in `p.distance(to: q)`, but it's possible to separate these two operations. The selector `p.distance(to:)` yields a *method value*, a function that binds a method to a specific instance `p`. This function can then be invoked without an instance; it needs only the remaining arguments:

```swift
let p = Point(x: 1, y: 2)
let q = Point(x: 4, y: 6)

let distanceFromP = p.distance(to:)  // method value
print(distanceFromP(q))  // "5.0"
let origin = Point(x: 0, y: 0)
print(distanceFromP(origin))  // "2.23606797749979", √5
```

Method values are useful when an API calls for a function value, and the desired behavior for that function is to call a method on a specific instance:

```swift
let distances = [q, origin].map(p.distance(to:))
```

Related to the method value is the *unapplied method reference*. Whereas a method value is formed by selecting a method from an instance, an unapplied method reference names the method through its type: `Point.distance(to:)`. It produces a *curried* function, one that takes the instance and returns a method value:

```swift
let distance = Point.distance(to:)
print(distance(p)(q))  // "5.0"
print(type(of: distance))  // "(Point) -> (Point) -> Double"
```

This is occasionally useful when you need to choose a method dynamically and apply it to many instances. Operators are static functions, so they can be chosen the same way, which is often clearer. In the following example, the variable `op` represents either the addition or the subtraction operator for `Point`, and `Path.translate(by:add:)` calls it for each point in the path:

```swift
extension Point {
    static func + (p: Point, q: Point) -> Point { Point(x: p.x + q.x, y: p.y + q.y) }
    static func - (p: Point, q: Point) -> Point { Point(x: p.x - q.x, y: p.y - q.y) }
}

extension Array where Element == Point {
    mutating func translate(by offset: Point, add: Bool) {
        let op: (Point, Point) -> Point = add ? (+) : (-)
        for i in indices {
            // Call either self[i] + offset or self[i] - offset.
            self[i] = op(self[i], offset)
        }
    }
}
```

The operators must be parenthesized, `(+)`, to refer to them as function values.

### 6.4.1. Key Paths

A *key path* is a reference to a property, rather than a method, that isn't yet bound to an instance. It's written with a backslash, the type name, and a path of properties: `\Point.x`, `\ColoredPoint.point.y`, `\Path.count`. When the type can be inferred, it may be omitted: `\.x`.

A key path can be applied to any instance of its root type using the `[keyPath:]` subscript, and if it refers to a mutable property, it can be used to modify it:

```swift
var pt = Point(x: 1, y: 2)
let kp = \Point.y
print(pt[keyPath: kp])  // "2.0"
pt[keyPath: kp] = 10
print(pt)  // "Point(x: 1.0, y: 10.0)"
```

Key paths are values: they can be stored, passed to functions, compared with `==`, and used as dictionary keys. And a key path literal can be used anywhere a function from the root type to the property type is expected, which makes for concise code with the standard library's higher-order functions:

```swift
let xs = perim.map(\.x)  // [1.0, 5.0, 5.0, 1.0]
let maxY = perim.max(by: { $0.y < $1.y })
let sortedByX = perim.sorted(using: KeyPathComparator(\.x))  // Foundation
```

Key paths are statically typed (`\Point.x` has type `WritableKeyPath<Point, Double>`), so they're checked by the compiler. We saw them used with `@dynamicMemberLookup` in Section 4.4, and we'll see them again in Chapter 12, where they provide a type-safe alternative to setting properties by name through reflection.

## 6.5. Example: Bit Vector Type

Sets in Swift are usually implemented as `Set<Element>`, a hash table. But there are many cases where a set of small non-negative integers with many elements is better represented by a *bit vector*: a data-flow analysis in a compiler, say, or the set of open file descriptors. A bit vector uses an array of unsigned integer values, or "words," each bit of which represents a possible element of the set. The set contains *i* if the *i*th bit is set. The following program demonstrates a simple bit vector type with three methods:

```swift
// swiftpl/ch6/intset
/// An IntSet is a set of small non-negative integers.
struct IntSet {
    private var words: [UInt64] = []

    /// Reports whether the set contains the non-negative value x.
    func contains(_ x: Int) -> Bool {
        let (word, bit) = x.quotientAndRemainder(dividingBy: 64)
        return word < words.count && words[word] & (1 << bit) != 0
    }

    /// Adds the non-negative value x to the set.
    mutating func insert(_ x: Int) {
        precondition(x >= 0, "IntSet elements must be non-negative")
        let (word, bit) = x.quotientAndRemainder(dividingBy: 64)
        while word >= words.count {
            words.append(0)
        }
        words[word] |= 1 << bit
    }

    /// Sets the set to the union of itself and other.
    mutating func formUnion(_ other: IntSet) {
        for (i, otherWord) in other.words.enumerated() {
            if i < words.count {
                words[i] |= otherWord
            } else {
                words.append(otherWord)
            }
        }
    }
}
```

Since each word has 64 bits, to locate the bit for `x`, we use the quotient `x / 64` as the word index and the remainder `x % 64` as the bit index within that word, conveniently computed together by `quotientAndRemainder(dividingBy:)`. The `formUnion` operation uses the bitwise OR operator `|` to compute the union 64 elements at a time. (We'll revisit the choice of 64-bit words in Exercise 6.5.) The expression `1 << bit` relies on Swift's heterogeneous shifts: the literal `1` takes its type, `UInt64`, from the other operand of `&` or `|=`, and the shift amount can be an `Int`. The method names follow the conventions of the standard library's `SetAlgebra` protocol, which `Set` and `OptionSet` conform to.

This implementation lacks many desirable features, some of which are posed as exercises below, but one is hard to live without: a way to print an `IntSet` as a string. Let's give it a `description` property by conforming to `CustomStringConvertible`:

```swift
extension IntSet: CustomStringConvertible {
    /// Returns the set as a string of the form "{1 2 3}".
    var description: String {
        var elements: [String] = []
        for (i, word) in words.enumerated() where word != 0 {
            for j in 0..<64 where word & (1 << j) != 0 {
                elements.append(String(64 * i + j))
            }
        }
        return "{" + elements.joined(separator: " ") + "}"
    }
}
```

Notice that the extension can access the `private` property `words`. In Swift, `private` means visible within the enclosing declaration *and its extensions in the same file*.

We can now demonstrate `IntSet` in action:

```swift
var x = IntSet()
var y = IntSet()
x.insert(1)
x.insert(144)
x.insert(9)
print(x)  // "{1 9 144}"

y.insert(9)
y.insert(42)
print(y)  // "{9 42}"

x.formUnion(y)
print(x)  // "{1 9 42 144}"

print(x.contains(9), x.contains(123))  // "true false"
```

A word of caution, for readers coming from Go: in the Go version of this example, a value and a pointer to it print differently, depending on whether the `String` method has a pointer receiver. In Swift, there's no such distinction. A struct's `description` works the same way on a `let`, a `var`, a temporary, or an element of an array.

And notice something we got for free. In Go, copying an `IntSet` copies the slice header, which aliases the words of the original; modifying the copy can corrupt the original, which is why Go's version needs an explicit `Copy` method. In Swift, `var z = x` gives `z` its own logical copy, thanks to copy-on-write arrays, and modifying `z` never affects `x`.

**Exercise 6.1:** Implement these additional members:

```swift
var count: Int  // return the number of elements
mutating func remove(_ x: Int)  // remove x from the set
mutating func removeAll()  // remove all elements from the set
```

**Exercise 6.2:** Define a variadic `mutating func insert(_ values: Int...)` method that allows a list of values to be added, such as `s.insert(1, 2, 3)`.

**Exercise 6.3:** Implement `formIntersection(_:)`, `subtract(_:)`, and `formSymmetricDifference(_:)`, and their non-mutating counterparts `intersection`, `subtracting`, and `symmetricDifference`. (The symmetric difference of two sets contains the elements present in one set or the other but not both.)

**Exercise 6.4:** Make `IntSet` conform to `Sequence`, so that its elements can be iterated with `for`-`in`. (See Section 7.7.) Which methods do you get for free?

**Exercise 6.5:** Make `IntSet` conform to the standard library's `SetAlgebra` protocol and `ExpressibleByArrayLiteral`. Which requirements have default implementations? Write a test that compares the behavior of `IntSet` with `Set<Int>` on random inputs.

## 6.6. Encapsulation

A variable or method of an object is said to be *encapsulated* if it is inaccessible to clients of the object. Encapsulation, sometimes called *information hiding*, is a key aspect of object-oriented programming.

Swift has five access levels, from most to least restrictive:

```
private       visible within the enclosing declaration, and its extensions in the same file
fileprivate   visible within the same source file
internal      visible within the same module (the default)
package       visible within modules of the same package (Swift 5.9)
public        visible to any module that imports this one
open          like public, but also allows subclassing and overriding outside the module
```

Go has only two levels, controlled by capitalization: exported and unexported. The unit of encapsulation in Go is the package. In Swift, you can encapsulate at the level of a single type, using `private`.

```swift
struct IntSet {
    private var words: [UInt64] = []
}
```

Clients of `IntSet` can't see `words`, so they can manipulate the set only through its methods, and the representation can change in the future without breaking them.

It's common to want a property that clients can read but not write. Swift supports that directly by giving the setter a more restrictive access level than the getter:

```swift
struct Counter {
    private(set) var count = 0

    mutating func increment() { count += 1 }
    mutating func reset() { count = 0 }
}
```

Clients can read `count` but can change it only with `increment` and `reset`. Go code achieves the same thing with an unexported field and an exported getter method; in Swift, getters and setters are almost never written as methods, since a property can always be changed later from stored to computed without breaking its clients.

That's one of the benefits of encapsulation. Here are three in all.

First, because clients cannot directly modify the object's variables, one need inspect fewer statements to understand the possible values of those variables.

Second, hiding implementation details prevents clients from depending on things that might change, which gives the designer greater freedom to evolve the implementation without breaking API compatibility. As an example, consider a `Buffer` type that holds bytes. It might be implemented with an `[UInt8]` today, and later grow an inline buffer for small contents, to avoid heap allocation. If its storage is `private`, the change is invisible to clients.

Third, and in many cases most important, encapsulation prevents clients from setting an object's variables arbitrarily. Because the object's variables can be set only by functions in the same type, the author of that type can ensure that all those functions maintain the object's internal invariants. For example, the `Counter` type above permits clients to increment the counter or to reset it to zero, but not to set it to some arbitrary value.

Swift offers one more tool for maintaining invariants: *property observers*. A stored property can have `willSet` and `didSet` blocks that run before and after each assignment:

```swift
struct Logger {
    var prefix: String
    var level: Int = 0 {
        didSet {
            level = min(max(level, 0), 5)  // clamp to valid range
        }
    }
}
```

When a property observer assigns to its own property, as here, the observer isn't triggered again. Observers let a type enforce constraints and maintain derived state while keeping the convenient property syntax.

Encapsulation is not always desirable. A type like `Point` reveals its properties, and that's fine: a point *is* its two coordinates, and there's no invariant to protect. Similarly, Foundation's `Date` hides its representation (a `Double` count of seconds) but publicly exposes conversions like `timeIntervalSince1970`, while `Duration` exposes its components. Like many design decisions, it's a judgment call.

When writing a library, remember that `public` means *forever*. Once a type or method is public and others depend on it, changing it is a breaking change. Swift's default of `internal` means that nothing is public by accident; make each declaration public deliberately, as part of the module's interface.

**Exercise 6.6:** Write a `Temperature` struct that stores a value in kelvins and exposes `celsius` and `fahrenheit` as computed properties with both getters and setters. Use a `private(set)` or a property observer to ensure that the temperature can never be set below absolute zero.
