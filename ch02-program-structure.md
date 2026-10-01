# 2. Program Structure

Chapter 1 moved quickly. This chapter slows down and looks at the parts that every Swift program is assembled from: names, declarations, constants and variables, assignment, new types, modules, and the rules of scope that decide which declaration a name refers to. Most of these ideas exist in some form in every language, so the emphasis here is on where Swift's choices differ, and why. The examples are deliberately small so that the language, not the algorithms, stays in focus.

## 2.1. Names

Names in Swift, whether of variables, functions, types, enum cases, or modules, start with a letter or underscore, followed by any number of letters, digits, and underscores. "Letter" covers most of Unicode, so `größe`, `π`, and even `🚀count` are legal names, though code meant for a wide audience usually sticks to what's easy to type. Names are case-sensitive: `userCount` and `UserCount` are different names.

Some words are reserved by the language:

```
actor       class       enum        func        import      init
let         protocol    struct      subscript   typealias   var
extension   deinit      associatedtype          operator    precedencegroup

break       case        catch       continue    default     defer
do          else        fallthrough for         guard       if
in          repeat      return      switch      throw       where
while       try         await

as   Any  false  is  nil  self  Self  super  true
```

Many other words, such as `mutating`, `override`, `lazy`, `weak`, `some`, `any`, `open`, and `async`, are *contextual*: they're keywords only in particular positions and ordinary names everywhere else. To use a reserved word as a name, perhaps to match a field in a data format, surround it with backticks, as in ``let `class` = "economy"``. Swift 6 also lets backticks enclose names with spaces and punctuation, which is mostly used for descriptive test names (Chapter 11).

The standard library declares many names of its own, such as `Int`, `String`, `print`, `min`, and `max`. These aren't reserved; your own declarations can reuse them, hiding the library's versions within their scope. That's legal but rarely wise.

A name declared inside a function is *local* to it. A name declared at the top level of a file is visible throughout the module. Whether it's visible *outside* the module isn't determined by the spelling of the name, as it is in some languages, but by an explicit *access level*. Declarations are `internal` by default, visible throughout their own module. They must be marked `public` to be seen by other modules, and can be narrowed to a single file with `fileprivate` or to a single declaration and its extensions with `private`. Sections 6.6 and 10.5 cover access control.

Swift naming follows strong community conventions, codified in the official *API Design Guidelines*. Types and protocols are `UpperCamelCase`; functions, methods, properties, variables, constants, enum cases, and argument labels are `lowerCamelCase`. Acronyms are uniformly upper- or lowercase depending on position, so it's `URLSession` and `htmlBody`, `userID` and `idToken`, but never `UrlSession` or `userId`.

The guidelines' central principle is *clarity at the point of use*. Names should read well where they're used, which is far more often than where they're declared. Swift's argument labels make that possible, letting a call read almost like an English phrase:

```swift
queue.insert(job, at: 0)
let i = names.firstIndex(of: "Grace")
text.replacing("colour", with: "color")
```

Name length should follow scope. A loop index used for three lines can be `i`; a closure's argument can be `$0`; a public method used across a codebase deserves a full, descriptive name.

## 2.2. Declarations

A *declaration* introduces a name and says what it refers to. Swift's main kinds of declarations are `let` and `var` for constants and variables, `func` for functions, `struct`, `enum`, `class`, `actor`, and `protocol` for types, `typealias` for alternative names of types, and `extension` for adding to existing types. This chapter covers constants, variables, and simple types; functions get Chapter 5, methods and extensions Chapter 6, and protocols Chapter 7.

A program's declarations can appear in any order. A function can be called above the point where it's declared, and a type can be used before its definition. The one exception is `main.swift`, whose top-level code runs in order, so a variable declared there must be declared before the code that uses it.

Here's a small program that computes a restaurant tip:

```swift
// swiftpl/ch2/tip
// Tip prints the tip and total for a restaurant bill.
import Foundation

let bill = 48.50
let rate = 0.18
let tip = bill * rate
print(String(format: "tip: $%.2f, total: $%.2f", tip, bill + tip))
// Output:
// tip: $8.73, total: $57.23
```

The three constants are declared at the top level of `main.swift`. That's fine in a script-sized program, but as programs grow, computations belong in functions, where their names are local and their inputs explicit. A function declaration gives a name, a list of *parameters*, an optional result type after `->`, and a body:

```swift
// swiftpl/ch2/tip2
// Tip2 prints tips for a bill at several rates.
import Foundation

let bill = 48.50
for percent in [10, 18, 20] {
    let t = tip(on: bill, rate: Double(percent) / 100)
    print(String(format: "%d%%: $%.2f", percent, t))
}

/// Returns the tip on an amount, rounded to the nearest cent.
func tip(on amount: Double, rate: Double = 0.18) -> Double {
    (amount * rate * 100).rounded() / 100
}
```

```
10%: $4.85
18%: $8.73
20%: $9.70
```

Each parameter has an *argument label*, used by callers, and a *parameter name*, used in the body. Here `on` is the label and `amount` the name, so calls read `tip(on: bill)`. A parameter may also have a default value, like `rate`, which callers can omit. When a function's body is a single expression, its value is the result, and `return` may be left out. Section 5.1 covers all of this in detail.

## 2.3. Variables

A variable declaration has the general form

```swift
var name: Type = expression
```

and either the type or the initial value, but not both, may be left out. Without a type, the type is inferred from the initial value. Without an initial value, the variable must be assigned before it's first read.

That last rule is enforced strictly. Many languages give an unassigned variable a default value (0, `nil`, or an empty string) or leave whatever bytes happen to be in memory. Swift does neither. Instead, the compiler performs *definite initialization* analysis, tracking every path through the code, and rejects any read that might happen before a write:

```swift
var label: String
if count > 0 {
    label = "\(count) items"
}
print(label)  // compile error: variable 'label' used before being initialized
```

Every path that reaches the `print` must assign `label`. The same rule governs initializers, which must give every stored property a value before the new instance can be used. The rule eliminates a whole category of bugs: reading garbage memory, or silently using a zero nobody meant.

Optional variables are the exception. A `var` of optional type with no initial value starts out as `nil`:

```swift
var lastError: String?  // nil until something is assigned
```

Several variables can be declared together, and a *tuple pattern* can unpack several values at once:

```swift
var width, height: Int  // two uninitialized Ints
var name = "Ada", age = 36, member = true
var (x, y) = (0.0, 1.0)
```

The tuple form is especially handy with functions that return several values:

```swift
let (quotient, remainder) = 47.quotientAndRemainder(dividingBy: 5)
```

### 2.3.1. Constants

A `let` declaration has the same forms as `var`, but once its value is set it can never be reassigned:

```swift
let maxAttempts = 3
maxAttempts = 4  // compile error: cannot assign to value: 'maxAttempts' is a 'let' constant
```

A `let` needn't be initialized where it's declared, as long as every path assigns it exactly once before use. That makes it possible to choose a constant's value with ordinary control flow:

```swift
let greeting: String
if hour < 12 {
    greeting = "Good morning"
} else {
    greeting = "Good afternoon"
}
```

Prefer `let`, and use `var` only for things that really change. The compiler helps: it warns when a `var` is never modified. For value types, which include structs, enums, and all the standard collections, a `let` is completely immutable. A `let` array can't be appended to, and a `let` struct can't have any property changed. Section 2.3.2 explains the different meaning of `let` for reference types.

Note that a `let` is not a compile-time constant in the sense of C's `#define` or Go's `const`: its value may be computed at run time, by any expression. Section 3.6 describes how Swift's literals provide many of the conveniences of compile-time constants.

### 2.3.2. Values, References, and `inout`

Many languages rely on pointers for three jobs: letting a function update its caller's variables, avoiding expensive copies of large data, and building linked structures. Swift handles all three without everyday pointers, by dividing types into two families with different behavior, and by providing `inout` parameters.

A *value type* gives each variable its own independent value. Assignment, initialization, and argument passing copy the value, at least logically. Structs, enums, and tuples are value types, and so are all of the standard library's basic types and collections: `Int`, `Double`, `Bool`, `String`, `Array`, `Dictionary`, and `Set`.

```swift
var original = ["a", "b"]
var copy = original
copy.append("c")
print(original)  // "["a", "b"]"
print(copy)  // "["a", "b", "c"]"
```

"Logically" is important here. The collections use *copy-on-write*: assignment shares the underlying storage, and a real copy is made only if one of the values is later modified while sharing. Passing a large array to a function is therefore as cheap as passing a pointer. Section 4.2 shows how this works.

A *reference type* gives variables references to a single shared instance. Classes and actors are reference types, as are closures:

```swift
final class Session {
    var requests = 0
}

let s1 = Session()
let s2 = s1  // the same session
s2.requests += 1
print(s1.requests)  // "1"
```

`s1` and `s2` are `let` constants, yet `requests` changed. For a reference type, `let` fixes which object the name refers to, not the object's contents. The operator `===` tests whether two references refer to the same instance: `s1 === s2` is `true`.

Classes suit things with *identity*, where everyone holding a reference should see the same changes: a network session, an open file, a cache shared across a program. Most other data is safer as a value, since nothing else can change it while you hold it. Swift encourages reaching for `struct` first.

That leaves the job of letting a function modify a variable in its caller, which is what `inout` parameters do:

```swift
func applyInterest(to balance: inout Double, rate: Double) {
    balance += balance * rate
}

var savings = 1000.0
applyInterest(to: &savings, rate: 0.05)
print(savings)  // "1050.0"
```

The `&` at the call site marks the argument that may be modified, so a reader can tell at a glance. Conceptually, the function receives a copy of the value and writes the result back when it returns; in practice, the compiler usually passes the variable's address. Only variables can be passed `inout`, not constants, literals, or the results of expressions.

Swift also enforces *exclusive access* to `inout` arguments: while a function is modifying a variable through an `inout` parameter, nothing else may access that variable. So this doesn't compile:

```swift
func transfer(_ amount: Double, from source: inout Double, to target: inout Double) {
    source -= amount
    target += amount
}

transfer(10, from: &savings, to: &savings)  // error: overlapping accesses to 'savings'
```

When the compiler can't prove that accesses don't overlap, it inserts a check that runs while the program executes. This *Law of Exclusivity* rules out the confusing bugs that arise when two names secretly refer to the same memory, and lets the optimizer assume that an `inout` parameter isn't changed behind its back.

Swift does have pointers, such as `UnsafeMutablePointer<Int>`, for working with C and other low-level code; Chapter 13 explains them.

Value types also make a popular library possible. `ArgumentParser` turns a program's command-line arguments into the properties of a struct. Here's a greeting command that takes a name, an optional repetition count, and a flag:

```swift
// swiftpl/ch2/greet
// Greet greets someone, possibly more than once and possibly loudly.
import ArgumentParser

@main
struct Greet: ParsableCommand {
    @Option(name: .shortAndLong, help: "how many times to greet")
    var count = 1

    @Flag(help: "greet loudly")
    var shout = false

    @Argument(help: "who to greet")
    var name = "world"

    func run() {
        let greeting = "Hello, \(name)!"
        for _ in 0..<count {
            print(shout ? greeting.uppercased() : greeting)
        }
    }
}
```

This file uses `@main` rather than top-level code, so it must not be named `main.swift`; call it `Greet.swift`. The package depends on `https://github.com/apple/swift-argument-parser` and uses its `ArgumentParser` product, declared in the manifest as in Section 1.7.

The struct conforms to the `ParsableCommand` protocol, and each stored property carries a *property wrapper* describing how it's filled in from the command line: `@Option` for a named value, `@Flag` for a Boolean switch, and `@Argument` for a positional argument. The initial values are the defaults. When the program starts, `ArgumentParser` creates an instance of `Greet`, sets its properties from the arguments, and calls `run`:

```
$ swift build
$ .build/debug/greet
Hello, world!
$ .build/debug/greet --count 2 Grace
Hello, Grace!
Hello, Grace!
$ .build/debug/greet -c 1 --shout Ada
HELLO, ADA!
$ .build/debug/greet --help
USAGE: greet [--count <count>] [--shout] [<name>]

ARGUMENTS:
  <name>                  who to greet (default: world)

OPTIONS:
  -c, --count <count>     how many times to greet (default: 1)
  --shout                 greet loudly
  -h, --help              Show help information.
```

Invalid arguments, such as a non-numeric count, produce an error message and the usage text, and the program exits with a failure status, all without any code of ours.

The expression `shout ? greeting.uppercased() : greeting` uses the *conditional operator*: it evaluates to the second operand if the condition is true and the third otherwise. Since Swift 5.9, `if` and `switch` can also be used as expressions, so the same thing could be written `let line = if shout { greeting.uppercased() } else { greeting }`.

### 2.3.3. Initializers and `deinit`

Every instance of a struct, class, or enum is created by an *initializer*, a special function named `init` and invoked through the type's name: `Session()`, `Point(x: 1, y: 2)`, `String(contentsOfFile: path, encoding: .utf8)`. A struct automatically gets a *memberwise initializer* with a parameter for each stored property. A class doesn't, but a class whose stored properties all have default values, like `Session`, gets an initializer with no parameters. Chapters 4 and 6 show how to write initializers of your own.

A class may also define a *deinitializer*, `deinit`, which runs just before an instance is destroyed. It's the place to release resources the object owns:

```swift
final class TempFile {
    let path: String
    init(path: String) {
        self.path = path
    }
    deinit {
        try? FileManager.default.removeItem(atPath: path)
    }
}
```

### 2.3.4. Lifetime of Variables

A variable's *lifetime* is the stretch of the program's execution during which it exists. A global variable lives for the whole run. A local variable comes into existence each time its declaration is executed and lives as long as anything can still use it; function parameters are local variables too, created afresh on each call.

Where a value is stored is the compiler's decision. Most local values live in registers or on the stack, but a local variable captured by a closure that outlives the function, for example, must be moved to the heap. None of this affects whether a program is correct, though it can matter for performance.

Instances of classes live on the heap and are managed by *automatic reference counting* (ARC). Each instance counts the strong references to it. Copying a reference increments the count, dropping one decrements it, and when the count reaches zero, the instance's `deinit` runs and its memory is freed, immediately and predictably, not at some later collection:

```swift
do {
    let scratch = TempFile(path: "/tmp/scratch")
    // ...use scratch...
}  // the last reference disappears here: deinit runs and the file is removed
```

Reference counting has one blind spot: *cycles*. If object A holds a strong reference to B and B holds one back to A, neither count can reach zero, and both objects leak. Cycles are broken by making one of the references `weak`, which doesn't count toward keeping the object alive and becomes `nil` when the object is freed, or `unowned`, which doesn't count either and is assumed never to outlive its object:

```swift
final class TreeNode {
    var children: [TreeNode] = []
    weak var parent: TreeNode?  // weak, so parent and children don't keep each other alive
}
```

Closures can form cycles too: an object that stores a closure that refers back to the object. The usual cure is a *capture list*, `{ [weak self] in ... }`, described in Section 5.6.

## 2.4. Assignments

An assignment statement updates the value of a variable, or of some part of one:

```swift
total = 0  // a variable
session.requests = 0  // a property of an instance
scores[2] = 97  // an element of an array or dictionary
stock["figs", default: 0] = 12  // a subscript with extra arguments
```

In Swift, assignment is a statement, not an expression; it produces no value. So the classic C typo `if x = 0` is a compile-time error instead of a silent bug.

Each arithmetic and bitwise operator has an *assignment operator* form, which updates a variable in place without repeating its name or re-evaluating the expression that locates it:

```swift
scores[index(of: player)] += 5
```

`+= 1` and `-= 1` take the place of the `++` and `--` operators found in other languages; Swift removed them because they encouraged hard-to-read expressions.

Assignments can reach as deep into a value as needed. `orders[i].customer.address.city = "Leeds"` changes one property of a struct nested inside a struct inside an array element, in place, without copying anything else, provided `orders` is a `var`.

### 2.4.1. Tuple Assignment

A *tuple assignment* updates several variables at once. Every expression on the right is evaluated before any variable on the left changes, which makes it the natural way to express values that depend on each other's old values. Swapping two variables is the simplest case:

```swift
(left, right) = (right, left)
```

The same works for any number of values. Here three variables rotate, each taking the value of the next:

```swift
var (first, second, third) = ("red", "green", "blue")
(first, second, third) = (second, third, first)
print(first, second, third)  // "green blue red"
```

and here a single pass over an array tracks its smallest and largest values together:

```swift
var (low, high) = (Int.max, Int.min)
for reading in [17, 4, 22, 9] {
    (low, high) = (min(low, reading), max(high, reading))
}
print(low, high)  // "4 22"
```

For swapping two variables there's also the library function `swap(&a, &b)`, and for swapping two array elements in place, `a.swapAt(i, j)`.

Tuples returned from functions can be unpacked by assignment, or kept whole and accessed by label:

```swift
let result = 47.quotientAndRemainder(dividingBy: 5)
print(result.quotient, result.remainder)  // "9 2"
```

When a function returns a value you don't need, the compiler warns that it's unused. Assigning it to `_` says that ignoring it is deliberate:

```swift
_ = cache.removeValue(forKey: staleKey)
```

Functions whose results are routinely ignored can be declared `@discardableResult`, which suppresses the warning; `removeLast()` on arrays is one.

### 2.4.2. Assignability

Assignments also happen implicitly: when an argument is passed to a parameter, when a function returns a value, and when the elements of a collection literal are stored:

```swift
let primaries = ["red", "green", "blue"]  // each element is assigned into the array
```

In every case, explicit or implicit, the value must be *assignable* to the type of its destination. Swift's rule is that the types must match exactly, with a few specific exceptions: a value of type `T` can be assigned to `T?`, wrapping it automatically; an instance of a class can be assigned to a variable of one of its superclasses; and a value can be assigned to a variable of a protocol type it conforms to (Chapter 7). There are *no* implicit numeric conversions, not even widening ones. An `Int32` can't be assigned to an `Int64` without writing `Int64(x)`.

Comparison follows similar rules. `==` and `!=` require two operands of the same type, and that type must conform to the `Equatable` protocol. Each new type in later chapters will say whether and how its values can be compared.

## 2.5. Type Declarations

A type tells you how a value is represented and what you can do with it. Often, though, several different concepts share a representation. A `Double` might be a price, a distance, or an angle; a `String` might be a username or a password. Mixing up such values is a classic source of bugs, and the type system can help prevent it, if the concepts have types of their own.

Swift has two ways to name a type. A `typealias` merely gives an existing type another name:

```swift
typealias Kilometers = Double
typealias Miles = Double
```

This documents intent, but protects nothing: since `Kilometers` and `Miles` are both just `Double`, the compiler will happily add one to the other. Type aliases are most useful for shortening long types, such as `typealias Handler = @Sendable (Request) async throws -> Response`.

To make a genuinely new type with the same representation as an existing one, wrap it in a struct. A single-property struct costs nothing at run time, since it's laid out exactly like the property it contains:

```swift
// swiftpl/ch2/distance0
// Distance0 defines distinct types for kilometers and miles.

struct Kilometers {
    var value: Double
}

struct Miles {
    var value: Double
}

let marathon = Kilometers(value: 42.195)

func kmToMiles(_ k: Kilometers) -> Miles {
    Miles(value: k.value / 1.609344)
}

func milesToKm(_ m: Miles) -> Kilometers {
    Kilometers(value: m.value * 1.609344)
}
```

Now distances in different units can't be combined by accident; converting between them requires an explicit call to `kmToMiles` or `milesToKm`. (History has examples of what happens otherwise: a spacecraft has been lost to a mix-up between metric and imperial units in software.)

As written, though, the types are awkward: you can't write a distance as a plain literal, add two distances, or print one nicely. Swift lets the types opt in to each of those abilities by conforming to protocols, which we'll add in *extensions*:

```swift
// swiftpl/ch2/distance0 (continued)
extension Kilometers: ExpressibleByFloatLiteral, ExpressibleByIntegerLiteral {
    init(floatLiteral value: Double) { self.value = value }
    init(integerLiteral value: Int) { self.value = Double(value) }
}

extension Kilometers: Comparable {
    static func + (a: Kilometers, b: Kilometers) -> Kilometers { Kilometers(value: a.value + b.value) }
    static func - (a: Kilometers, b: Kilometers) -> Kilometers { Kilometers(value: a.value - b.value) }
    static func < (a: Kilometers, b: Kilometers) -> Bool { a.value < b.value }
}

extension Kilometers: CustomStringConvertible {
    var description: String { "\(value) km" }
}
```

With these, kilometers behave like numbers, but only with each other:

```swift
// swiftpl/ch2/distance0 (continued)
let warmup: Kilometers = 3
print(marathon + warmup)  // "45.195 km"
print(warmup < marathon)  // "true"
```

But they still can't be mixed with miles:

```swift
let m = kmToMiles(marathon)
print(marathon == m)  // compile error: '==' cannot be applied to 'Kilometers' and 'Miles'
```

`Comparable` requires `<`, and, through its parent protocol `Equatable`, `==`; the compiler writes `==` for us, since every stored property of the struct is itself `Equatable`. The `description` property, required by `CustomStringConvertible`, controls how a value looks when it's printed or interpolated into a string, which is why `print` shows `45.195 km`.

Extensions, operators, and protocols are covered in Chapters 6 and 7; for now, the point is that a few lines turn a raw number into a type with its own meaning.

## 2.6. Modules and Files

Code is organized into *modules*. A module is a set of Swift source files compiled together, usually kept in one directory; all the files of a module share a namespace, and each module's `public` declarations form its interface to other modules. In the Swift Package Manager, each module is a *target*, and a *package* may contain several targets that depend on each other.

Module names keep declarations from colliding. Two modules can each declare a type named `Distance`; a file importing both can tell them apart by qualifying the name with the module, as `Geometry.Distance`.

Let's turn our distance types into a reusable library. The package will contain a library target, `Distance`, and a command-line tool, `dist`, that uses it:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "distance",
    products: [
        .library(name: "Distance", targets: ["Distance"]),
    ],
    targets: [
        .target(name: "Distance"),
        .executableTarget(name: "dist", dependencies: ["Distance"]),
    ]
)
```

The library has two source files, to show that declarations in one file of a module are visible in the others without any imports. The types go in one file:

```swift
// swiftpl/ch2/distance/Sources/Distance/Distance.swift
// Package Distance provides distinct types for distances in kilometers and miles.
import Foundation

public struct Kilometers: Hashable, Sendable {
    public var value: Double
    public init(_ value: Double) { self.value = value }
}

public struct Miles: Hashable, Sendable {
    public var value: Double
    public init(_ value: Double) { self.value = value }
}

extension Kilometers {
    public static let marathon = Kilometers(42.195)
    public static let halfMarathon = Kilometers(21.0975)
}

extension Kilometers: CustomStringConvertible {
    public var description: String { String(format: "%.2f km", value) }
}

extension Miles: CustomStringConvertible {
    public var description: String { String(format: "%.2f mi", value) }
}
```

and the conversions in another:

```swift
// swiftpl/ch2/distance/Sources/Distance/Convert.swift

/// The number of kilometers in one international mile.
let kilometersPerMile = 1.609344

/// Converts a distance in kilometers to miles.
public func kmToMiles(_ k: Kilometers) -> Miles {
    Miles(k.value / kilometersPerMile)
}

/// Converts a distance in miles to kilometers.
public func milesToKm(_ m: Miles) -> Kilometers {
    Kilometers(m.value * kilometersPerMile)
}
```

Everything a client needs is marked `public`. That includes the initializers: the memberwise initializer the compiler writes for a struct is only `internal`, so a public struct whose clients should be able to create values needs a public initializer written out by hand. The extra step is intentional, since a struct's stored properties become part of its public interface only when its author says so. The constant `kilometersPerMile`, on the other hand, is left `internal`: clients don't need it, and keeping it hidden leaves us free to change it.

Well-known distances are *static properties* of `Kilometers` rather than global constants, so clients write `Kilometers.marathon`, or just `.marathon` where the type is clear from context. The conformances to `Hashable` and `Sendable`, which let distances serve as dictionary keys (Chapter 4) and be shared between concurrent tasks (Chapter 9), are written by the compiler.

### 2.6.1. Imports

To use another module's public declarations, a file `import`s it:

```swift
// swiftpl/ch2/distance/Sources/dist/main.swift
// Dist converts each numeric argument between kilometers and miles.
import Distance
import Foundation

for arg in CommandLine.arguments.dropFirst() {
    guard let x = Double(arg) else {
        FileHandle.standardError.write(Data("dist: not a number: \(arg)\n".utf8))
        exit(1)
    }
    let k = Kilometers(x)
    let m = Miles(x)
    print("\(k) = \(kmToMiles(k)), \(m) = \(milesToKm(m))")
}
```

```
$ swift run dist 5 26.2
5.00 km = 3.11 mi, 5.00 mi = 8.05 km
26.20 km = 16.28 mi, 26.20 mi = 42.16 km
```

An `import` makes the module's public names available throughout one file, with no qualification needed. Imports are per file: importing `Foundation` in one file of a module doesn't make it available in the others. A target can import only modules it depends on in the manifest.

### 2.6.2. Module Initialization

Some languages run initialization code when a module or package is loaded. Swift doesn't. Instead, every global variable and static property is initialized *lazily*, on its first use, and the language guarantees that this happens exactly once, even if several threads reach the variable at the same moment. (Top-level variables in `main.swift` are the exception; they're initialized in order as the top-level code runs.)

Lazy initialization means that a program never pays for globals it doesn't use, and that the order in which modules happen to be loaded can never affect the result.

When a global's initial value takes more than one expression to compute, use a closure that's called immediately, or any other expression that produces the value. Here's an example: a function that computes the CRC-32 checksum, used in ZIP files, PNG images, and Ethernet frames to detect corrupted data. The fast way to compute it uses a table of 256 precomputed values, one for each possible byte, and the table is a perfect candidate for a lazily initialized global:

```swift
// swiftpl/ch2/crc32
/// crcTable[n] is the CRC-32 of the single byte n, used to process input a byte at a time.
let crcTable: [UInt32] = (0..<256).map { n in
    var c = UInt32(n)
    for _ in 0..<8 {
        c = c & 1 != 0 ? 0xEDB8_8320 ^ (c >> 1) : c >> 1
    }
    return c
}

/// Returns the CRC-32 checksum of the bytes.
func crc32(_ bytes: some Sequence<UInt8>) -> UInt32 {
    var crc: UInt32 = 0xFFFF_FFFF
    for b in bytes {
        crc = crcTable[Int((crc ^ UInt32(b)) & 0xFF)] ^ (crc >> 8)
    }
    return crc ^ 0xFFFF_FFFF
}

print(String(crc32("hello".utf8), radix: 16))  // "3610a686"
```

The table is built by `map` over the range `0..<256`, with a closure that computes each entry from the polynomial `0xEDB88320`; the details of the arithmetic, a bitwise form of polynomial division, don't matter here. What does matter is when it runs: because `crcTable` is a global in a file other than `main.swift`, the table is built the first time `crc32` uses it, and never again, however many checksums are computed or tasks compute them.

The parameter type `some Sequence<UInt8>` means that `crc32` accepts any sequence of bytes, such as an array, a string's UTF-8 view, or a `Data` value. Section 7.5 explains this notation.

**Exercise 2.1:** Add `Meters` and `Feet` types to the `Distance` module, with conversions between all four units. How many conversion functions does that take? Can you find a design that needs fewer?

**Exercise 2.2:** Make `dist` read numbers from the standard input, one per line, when it's given no arguments, and accept an optional unit suffix (`5km`, `3mi`) that limits the output to the conversion out of that unit.

**Exercise 2.3:** Write a command, `cksum32`, that prints the CRC-32 of each file named on its command line, in hexadecimal. Compare its results with another tool, such as `crc32` or Python's `zlib.crc32`.

**Exercise 2.4:** Write a version of `crc32` that uses no table, processing each byte bit by bit as the table-building closure does. Measure how much slower it is. (Section 11.4 shows how to compare the performance of implementations systematically.)

## 2.7. Scope

The *scope* of a declaration is the region of source code in which its name refers to it. Scope is a property of the program text, determined at compile time. Don't confuse it with *lifetime*, the stretch of execution during which a variable exists, which is a run-time property.

A *block* is a sequence of statements in braces, such as the body of a function, a loop, or an `if`. A name declared in a block is visible only from its declaration to the end of the block.

Across a module, the scopes nest like this: declarations of the standard library and imported modules are visible in each file that imports them; types, functions, and global variables are visible throughout their module (unless they're `private` or `fileprivate`); and local declarations are visible from their declaration to the end of their block. Unlike global names, local variables must be declared before they're used.

To resolve a name, the compiler searches outward from the innermost block, through enclosing blocks, the module, and finally imported modules, and uses the first declaration it finds. If an inner block declares the same name as an outer one, the inner declaration *shadows* the outer one within its scope:

```swift
func render() {}

let theme = "light"

func example() {
    let render = "draft"
    print(render)  // "draft": the local constant shadows the module-level function
    print(theme)  // "light": the module-level constant
    print(undefinedName)  // compile error: cannot find 'undefinedName' in scope
}
```

A name can be declared only once in any one block; a second declaration in the same block is an error, which prevents many accidental redeclarations.

Shadowing is used on purpose all the time with optional binding. Unwrapping an optional under its own name is the normal idiom:

```swift
func welcome(_ nickname: String?) {
    if let nickname = nickname {
        print("Welcome back, \(nickname)")  // here nickname is a String, not a String?
    }
}
```

Since Swift 5.7, `if let nickname { ... }` means the same thing.

Blocks nest to any depth, and each can shadow names from those around it. In the function below, two variables named `x` coexist (a style to avoid in real code, but useful to illustrate the rules):

```swift
func shout(_ x: String) {
    for c in x {  // x is the parameter, a String
        if let x = c.uppercased().first {  // a new x, a Character, shadows the parameter
            print(x, terminator: "")
        }
    }
    print(" (\(x.count) letters)")  // x is the parameter again
}

shout("hello!")  // "HELLO! (6 letters)"
```

Inside the `if let` block, `x` is the `Character` it binds; everywhere else in the function, including the loop's sequence and the final `print`, it's the `String` parameter.

Conditions in `if`, `while`, and `guard` statements create scopes as well, with one asymmetry that makes `guard` so useful: a name bound by `if let` is visible only inside the `if` block, but a name bound by `guard let` is visible from the `guard` to the end of the *enclosing* block:

```swift
func loadSettings(from path: String) throws -> Settings {
    guard let data = FileManager.default.contents(atPath: path) else {
        throw SettingsError.missing(path)
    }
    // data is in scope, and unwrapped, from here to the end of the function
    return try JSONDecoder().decode(Settings.self, from: data)
}
```

Shadowing does have a classic pitfall. Suppose a program keeps a logging level in a global variable and sets it from the command line:

```swift
var logLevel = "info"

func configure(arguments: [String]) {
    let logLevel = arguments.contains("-v") ? "debug" : "info"  // NOTE: wrong!
    print("logging at \(logLevel)")
}
```

The `let` inside the function declares a *new* local constant that shadows the global. The function prints the level it computed, which looks right, but the global never changes, and the rest of the program keeps logging at `info`. Nothing warns about it. The fix is to assign, not declare:

```swift
func configure(arguments: [String]) {
    logLevel = arguments.contains("-v") ? "debug" : "info"
    print("logging at \(logLevel)")
}
```

(In practice, the Swift 6 compiler makes global variables like `logLevel` harder to use than this, because any task in the program could access one concurrently. It requires a mutable global to be protected, for example by isolating it to the main actor with `@MainActor`. Chapter 9 explains why. It's one more reason to keep state out of globals.)
