# 2. Program Structure

In Swift, as in any other programming language, one builds large programs from a small set of basic constructs. Variables store values. Simple expressions are combined into larger ones with operations like addition and subtraction. Basic types are collected into aggregates like arrays and structs. Expressions are used in statements whose execution order is determined by control-flow statements like `if` and `for`. Statements are grouped into functions for isolation and reuse. Functions are gathered into source files and modules.

We saw examples of most of these in the previous chapter. In this chapter, we'll go into more detail about the basic structural elements of a Swift program. The example programs are intentionally simple, so we can focus on the language without getting sidetracked by complicated algorithms or data structures.

## 2.1. Names

The names of Swift functions, variables, constants, types, enum cases, and modules follow a simple rule: a name begins with a letter or an underscore and may have any number of additional letters, digits, and underscores. "Letter" is interpreted generously. Most of Unicode is allowed, including accented letters, Greek and CJK characters, and even emoji, so `π`, `naïve`, and `🐶🐮` are all legal names. The convention, however, is to stick to letters a reader can type. Case matters: `heapSort` and `Heapsort` are different names.

Swift has a few dozen *keywords* that are reserved in most contexts. Among those used in declarations and statements are:

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

Others, like `open`, `mutating`, `override`, `lazy`, `weak`, `some`, `any`, and `async`, are *contextual keywords*: they have special meaning only in particular positions and can otherwise be used as ordinary names. If you really need to use a reserved word as a name, perhaps because it's the name of a field in some external data format, enclose it in backticks: ``let `default` = 3``. Swift 6 extended backticks to allow nearly arbitrary text, including spaces, in a name, which is mostly useful for test function names (Chapter 11).

In addition, there are many predeclared names in the standard library, such as `Int`, `String`, `print`, `min`, and `max`. These are not reserved, so you may use them in declarations, shadowing the standard library's versions within your scope. Beware the confusion this can cause.

If an entity is declared within a function, it is *local* to that function. If declared outside of a function, it is visible in all files of the module to which it belongs. Unlike in Go, the case of the first letter has no effect on visibility. Instead, Swift has explicit *access control* keywords. By default every declaration has `internal` access, which means it is visible throughout its own module but not to other modules. To make a declaration visible to modules that import yours, mark it `public`; to hide it within a single file, mark it `private` or `fileprivate`. Section 6.6 and Section 10.5 discuss access control in detail.

Swift has strong conventions for names, which the community follows closely and which are set out in the official *API Design Guidelines*. Types and protocols use `UpperCamelCase`; everything else (functions, variables, constants, enum cases, argument labels) uses `lowerCamelCase`. Acronyms and initialisms are written in uniform case according to their position: `utf8Bytes`, `userID`, `URLSession`, `htmlEscape`, never `Utf8Bytes` or `userId`.

The guidelines emphasize *clarity at the point of use*. A name should make sense where it is called, not where it is declared, and should include enough words to avoid ambiguity but no needless ones. Swift's *argument labels* let the call read like a phrase:

```swift
names.insert("Ada", at: 0)
let i = names.firstIndex(of: "Ada")
words.remove(at: i)
```

Scope still matters for name length. A local loop variable might just be called `i`, and a closure parameter `$0`. A widely used function in a library deserves a longer, more descriptive name.

## 2.2. Declarations

A *declaration* names a program entity and specifies some or all of its properties. The main kinds of declarations are `let` and `var` for constants and variables, `func` for functions, `struct`, `enum`, `class`, `actor`, and `protocol` for new types, `typealias` for alternative names for types, and `extension` for adding to existing types. In this chapter we'll discuss constants, variables, and types; functions are covered in Chapter 5, methods and extensions in Chapter 6, and protocols in Chapter 7.

A Swift program is stored in one or more files whose names end in `.swift`. There is no fixed order of declarations within a file. A function may be called before its declaration appears, and a type may be used before its definition. The exception is `main.swift`, whose top-level code runs from top to bottom, so a top-level variable there must be declared before the statements that use it.

For example, this program declares a constant, a variable, and a function:

```swift
// swiftpl/ch2/boiling
// Boiling prints the boiling point of water.

let boilingF = 212.0

let f = boilingF
let c = (f - 32) * 5 / 9
print("boiling point = \(f)°F or \(c)°C")
// Output:
// boiling point = 212.0°F or 100.0°C
```

The constant `boilingF` is a global declaration, as are `f` and `c`. A global is visible throughout the module, which is fine for small programs. As programs grow, you'll want to organize code into functions and types, which keep names local.

A function declaration has a name, a list of parameters (the variables whose values are provided by the function's callers), an optional result type, and a body that contains the statements that define what the function does. The result type is omitted if the function does not return anything. Execution of the function begins with the first statement and continues until it encounters a `return` statement or reaches the end of a function that has no result type. Control and any result are then returned to the caller.

We've seen a fair number of functions already and there are lots more to come, including an extensive discussion in Chapter 5, so this is only a sketch. The function `fToC` below encapsulates the temperature conversion logic so that it is defined only once but may be used from multiple places. Here `main.swift` calls it twice, using the values of two different local constants:

```swift
// swiftpl/ch2/ftoc
// Ftoc prints two Fahrenheit-to-Celsius conversions.

let freezingF = 32.0
let boilingF = 212.0
print("\(freezingF)°F = \(fToC(freezingF))°C")  // "32.0°F = 0.0°C"
print("\(boilingF)°F = \(fToC(boilingF))°C")  // "212.0°F = 100.0°C"

func fToC(_ f: Double) -> Double {
    (f - 32) * 5 / 9
}
```

The underscore before the parameter name says that callers don't write an argument label, so the call is `fToC(212)` rather than `fToC(f: 212)`. Section 5.1 explains labels in full. Since the body of `fToC` consists of a single expression, the `return` keyword may be omitted; the value of the expression is the function's result.

## 2.3. Variables

A `var` declaration creates a variable of a particular type, attaches a name to it, and sets its initial value. Each declaration has the general form

```swift
var name: Type = expression
```

Either the `: Type` or the `= expression` part may be omitted, but not both. If the type is omitted, it is inferred from the initializer expression. If the initializer is omitted, the variable must be assigned a value before it is first read.

This last point is one of the key differences between Swift and Go. Go gives every variable a *zero value* at declaration, so a variable is never uninitialized. Swift has no zero values. Instead, the compiler performs *definite initialization* analysis, and refuses to compile any program that might read a variable before it has been assigned:

```swift
var s: String
if useGreeting {
    s = "hello"
}
print(s)  // compile error: variable 's' used before being initialized
```

Every path to the `print` must assign `s` first. The rule applies equally to properties of structs and classes: an initializer must assign every stored property before the new value can be used. This catches a class of bugs that are subtle in languages where uninitialized memory holds garbage or where a forgotten initialization silently leaves a zero.

Optional variables are the one exception: a `var` of optional type with no initializer is implicitly initialized to `nil`.

```swift
var name: String?  // implicitly nil
```

It is possible to declare and optionally initialize a set of variables in a single declaration, with a list of names, or with a *tuple pattern*:

```swift
var i, j, k: Int  // three Ints, not yet initialized
var b = true, f = 2.3, s = "four"  // Bool, Double, String
var (x, y) = (1, 2)  // destructure a tuple
```

The tuple form is especially useful with functions that return multiple values:

```swift
let (data, response) = try await URLSession.shared.data(from: url)
```

### 2.3.1. Constants

A `let` declaration has the same forms as `var`, but the name it introduces can't be reassigned after its initialization:

```swift
let maxRetries = 3
maxRetries = 4  // compile error: cannot assign to value: 'maxRetries' is a 'let' constant
```

Like a variable, a `let` constant need not be initialized at its declaration, so long as it is assigned exactly once on every path before it is used:

```swift
let message: String
if count == 0 {
    message = "none"
} else {
    message = "\(count) items"
}
```

This is a common idiom: the compiler guarantees that `message` is initialized exactly once, and the reader knows it never changes afterwards.

Use `let` unless you need `var`. The compiler will warn about any `var` that is never mutated. In Swift, unlike in many languages, `let` constants are genuinely immutable when they hold value types: a `let` array can't have elements appended, and a `let` struct can't have its properties changed. We'll return to this distinction between values and references in Section 2.3.2.

Note that Swift's `let` is not the same thing as Go's `const`. A Go constant is evaluated at compile time and can only hold basic types, while a Swift `let` may hold any value, computed at run time. We'll discuss Swift's compile-time literals in Section 3.6.

### 2.3.2. Values, References, and `inout`

Go programs use pointers routinely: to share a variable with a function that will update it, to avoid copying large structs, and to build linked data structures. Swift programs rarely use pointers. Instead, Swift divides types into two families with different semantics, and provides `inout` parameters for the remaining case.

A *value type* is one in which each variable holds its own independent copy of the data. Assignment, initialization, and argument passing all copy the value, at least logically. Structs, enums, and tuples are value types, as are all of the standard library's basic types and collections: `Int`, `Double`, `Bool`, `String`, `Array`, `Dictionary`, and `Set`.

```swift
var a = [1, 2, 3]
var b = a  // b is an independent copy of a
b.append(4)
print(a)  // "[1, 2, 3]"
print(b)  // "[1, 2, 3, 4]"
```

The copy is logical, not necessarily physical. Arrays, strings, and the other collections use *copy-on-write*: `b = a` merely shares the underlying storage, and the actual copy happens only if and when one of them is modified while the storage is shared. So passing a big array to a function costs about as much as passing a pointer. We'll look at the mechanism in Section 4.2.

A *reference type* is one in which variables hold references to a shared instance. Classes, actors, and closures are reference types.

```swift
final class Counter {
    var n = 0
}

let c1 = Counter()
let c2 = c1  // c2 refers to the same object as c1
c2.n += 1
print(c1.n)  // "1"
```

Notice that `c1` and `c2` are `let` constants, yet we changed `n`. For a reference type, `let` fixes the *reference* (`c1` will always refer to this particular `Counter`) but not the object's contents. Two references are *identical* if they refer to the same instance, which the `===` operator tests: `c1 === c2` is `true`.

Classes are the right tool when identity matters, when something represents a single shared resource like a file, a network connection, or a cache. Most other data is better modeled with value types, which can't be modified behind your back by some other part of the program. Swift encourages you to reach for `struct` first.

What about the remaining use of pointers: letting a function update a variable in its caller? That's what `inout` parameters do. An `inout` parameter is passed with an ampersand, and the function may modify it:

```swift
func incr(_ p: inout Int) -> Int {
    p += 1  // increments the caller's variable
    return p
}

var v = 1
_ = incr(&v)  // side effect: v is now 2
print(incr(&v))  // "3" (and v is 3)
```

Semantically, `inout` is *copy-in, copy-out*: the function receives the value, modifies it, and the modified value is written back when the function returns. In practice the compiler passes the variable's address, so there's no copying. Either way, the effect is that the caller's variable changes.

Swift is strict about `inout`. The argument must be a variable (not a `let`, a literal, or the result of an expression), and the `&` makes it obvious at the call site that the argument may change. Swift also enforces *exclusive access*: while a function holds an `inout` reference to a variable, no other code may access that same variable. This program, for example, does not compile:

```swift
var total = 10
func addTwice(_ x: inout Int, _ y: inout Int) { ... }
addTwice(&total, &total)  // error: overlapping accesses to 'total'
```

Where the compiler can't prove the absence of overlapping access, it inserts a run-time check. The *Law of Exclusivity* rules out the aliasing bugs that make pointer-heavy code hard to reason about, and it gives the optimizer freedom to assume that an `inout` parameter is not modified by anything else during the call.

Unsafe pointers exist too, with names like `UnsafeMutablePointer<Int>`, for interoperating with C and for low-level code. We'll study them in Chapter 13.

The `ArgumentParser` package, one of the most widely used Swift packages, is a good illustration of what value semantics and a few other features let you do. It parses command-line flags and arguments into the properties of a struct. Here's a version of `echo` that takes two optional flags: `-n` causes `echo` to omit the trailing newline that would normally be printed, and `-s sep` causes it to separate the output arguments by the contents of the string `sep` instead of the default single space.

```swift
// swiftpl/ch2/echo4
// Echo4 prints its command-line arguments.
import ArgumentParser

@main
struct Echo4: ParsableCommand {
    @Flag(name: .customShort("n"), help: "omit trailing newline")
    var omitNewline = false

    @Option(name: .customShort("s"), help: "separator")
    var sep = " "

    @Argument var words: [String] = []

    func run() {
        print(words.joined(separator: sep), terminator: omitNewline ? "" : "\n")
    }
}
```

Because it uses `@main` rather than top-level code, this file must *not* be called `main.swift`; call it `Echo4.swift`. The package depends on `https://github.com/apple/swift-argument-parser` and its `ArgumentParser` product, declared in the manifest as in Section 1.7.

The struct conforms to the `ParsableCommand` protocol. Each of its stored properties is decorated with a *property wrapper*: `@Flag` for a Boolean flag, `@Option` for a flag that takes a value, and `@Argument` for positional arguments. A property wrapper is a type that attaches extra behavior to a property, in this case information about how to parse it. The initial values are the defaults. When the program starts, `ArgumentParser` creates an instance of `Echo4`, fills in its properties from the command line, and calls `run`.

Let's run some test cases on `echo4`:

```
$ swift build
$ .build/debug/echo4 a bc def
a bc def
$ .build/debug/echo4 -s / a bc def
a/bc/def
$ .build/debug/echo4 -n a bc def
a bc def$
$ .build/debug/echo4 --help
USAGE: echo4 [-n] [-s <s>] [<words> ...]

ARGUMENTS:
  <words>

OPTIONS:
  -n                      omit trailing newline
  -s <s>                  separator (default:  )
  -h, --help              Show help information.
```

If the user supplies an invalid argument or flag, the program prints an error message and the usage, and exits with a non-zero status.

The `?:` *conditional operator* in `omitNewline ? "" : "\n"` evaluates to `""` if `omitNewline` is true and to `"\n"` otherwise. In Swift 5.9 and later, `if` and `switch` can also be used as expressions, so you could equally write:

```swift
let terminator = if omitNewline { "" } else { "\n" }
```

### 2.3.3. Initializers and `deinit`

Every struct, class, and enum is created by an *initializer*, a special function named `init` that's called with the type's name: `Counter()`, `Point(x: 1, y: 2)`, `String(contentsOfFile: path, encoding: .utf8)`. Structs get a *memberwise initializer* for free, with one parameter per stored property. Classes don't, but a class whose stored properties all have default values gets an `init()` that takes no arguments, as `Counter` did above. You can write your own initializers too, as we'll see in Chapter 4 and Chapter 6.

Classes (but not structs) may also have a *deinitializer*, a `deinit` block that runs just before an instance is destroyed:

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

The *lifetime* of a variable is the interval of time during which it exists as the program executes. The lifetime of a global variable is the entire execution of the program. Local variables, by contrast, have dynamic lifetimes: a new instance is created each time the declaration is executed, and the variable lives on until it is *unreachable*, at which point its storage may be recycled. Function parameters and results are local variables too; they are created each time their enclosing function is called.

For value types, the compiler decides where variables live. Most locals are kept in registers or on the stack, but a local that's captured by an escaping closure, for example, must be moved to the heap, because it has to outlive the call that created it. As in Go, this is not something you need to think about to write correct programs, though it's worth keeping in mind during performance optimization.

For reference types, Swift uses *automatic reference counting* (ARC) rather than a tracing garbage collector. Every class instance keeps a count of the strong references to it. The compiler inserts code to increment the count when a reference is copied and to decrement it when a reference goes away, and when the count drops to zero, the instance's `deinit` runs and its memory is freed, immediately and deterministically.

```swift
do {
    let f = TempFile(path: "/tmp/scratch")
    ...
}  // f goes out of scope here; deinit runs and the file is removed
```

The price of reference counting is that it can't reclaim *cycles*. If object A holds a strong reference to object B and B holds one back to A, neither count ever reaches zero, and both leak. You break such cycles by making one of the references `weak` (which doesn't contribute to the count and becomes `nil` when the object is freed) or `unowned` (which doesn't contribute to the count and traps if used after the object is freed):

```swift
final class Node {
    var children: [Node] = []
    weak var parent: Node?  // weak to avoid a cycle with children
}
```

A related trap involves closures. A closure stored in a property of an object, that refers to `self`, creates a cycle. The usual fix is a *capture list*, `{ [weak self] in ... }`, which we'll see in Chapter 5.

## 2.4. Assignments

The value held by a variable is updated by an assignment statement, which in its simplest form has a variable on the left of the `=` sign and an expression on the right.

```swift
x = 1  // named variable
p.x = 1  // property of a struct or class
a[i] = 1  // element of an array, dictionary, or other collection
counts[k, default: 0] = 1  // subscript with extra arguments
```

Unlike in C, an assignment is a statement, not an expression, and produces no value. That rules out the classic bug `if x = y` for `if x == y`, which is a compile error in Swift.

Each of the arithmetic and bitwise binary operators has a corresponding *assignment operator* allowing, for example, the last statement to be rewritten as

```swift
count[x] *= scale
```

which saves us from having to repeat (and re-evaluate) the expression for the variable. Swift has no increment or decrement operators; `v += 1` and `v -= 1` take their place.

Assignment to a property or element of a value type works through as many levels as needed: `points[i].x += 1` modifies the `x` of the `i`th element of the array in place. Swift handles this by treating each level as an `inout` access, so `points` must be a `var`.

### 2.4.1. Tuple Assignment

Another form of assignment, *tuple assignment*, allows several variables to be assigned at once. All of the right-hand side expressions are evaluated before any of the variables are updated, making this form most useful when some of the variables appear on both sides of the assignment, as happens, for example, when swapping the values of two variables:

```swift
(x, y) = (y, x)
```

Since this is so common, the standard library also has a `swap` function, `swap(&x, &y)`, and arrays have a `swapAt` method that swaps two elements: `a.swapAt(i, j)`.

Tuple assignment also helps to compute the greatest common divisor (GCD) of two integers:

```swift
func gcd(_ x: Int, _ y: Int) -> Int {
    var (x, y) = (x, y)
    while y != 0 {
        (x, y) = (y, x % y)
    }
    return x
}
```

Function parameters are constants, so we copy them into local variables with the same names before changing them. This kind of shadowing is common and harmless.

Or to compute the *n*th Fibonacci number iteratively:

```swift
func fib(_ n: Int) -> Int {
    var (x, y) = (0, 1)
    for _ in 0..<n {
        (x, y) = (y, x + y)
    }
    return x
}
```

Certain expressions produce several values. When such a call is used in an assignment, the left-hand side must have as many variables as the function has results:

```swift
let (quotient, remainder) = 17.quotientAndRemainder(dividingBy: 5)
```

Tuples may also have labeled elements, so the same result can be used without destructuring:

```swift
let qr = 17.quotientAndRemainder(dividingBy: 5)
print(qr.quotient, qr.remainder)  // "3 2"
```

If a function returns a value that you don't need, the compiler will usually warn that the result is unused. Assign it to the blank identifier `_` to make it clear that the discard is deliberate:

```swift
_ = incr(&v)
```

A function whose result is often ignored can be marked `@discardableResult` to suppress the warning; `Array.removeFirst()` and `Dictionary.updateValue(_:forKey:)` are two examples.

### 2.4.2. Assignability

Assignment statements are an explicit form of assignment, but there are many places in a program where an assignment occurs *implicitly*: a function call implicitly assigns the argument values to the corresponding parameters; a `return` statement implicitly assigns the operands to the result; and a literal expression for a collection implicitly assigns each of its elements:

```swift
let medals = ["gold", "silver", "bronze"]
```

The elements of an array literal must all have the same type, which is inferred from the elements, here `String`.

For an assignment, explicit or implicit, to be legal, the value must be *assignable* to the type of the variable. The rule is simple: the types must match exactly, with a few deliberate exceptions. A value of type `T` may be assigned to a variable of type `T?` (it's wrapped automatically); a class instance may be assigned to a variable of one of its superclasses; and a value may be assigned to a variable of a protocol type that it conforms to (Chapter 7). There are no implicit numeric conversions at all: an `Int` cannot be assigned to an `Int64` or a `Double` variable without an explicit conversion.

Whether two values may be compared with `==` and `!=` is related to assignability: they must be of the same type, and the type must conform to the `Equatable` protocol. As with assignability, we'll explain the cases as we encounter each new type.

## 2.5. Type Declarations

The type of a variable or expression defines the characteristics of the values it may take on, such as their size, how they are represented internally, the intrinsic operations that can be performed on them, and the methods associated with them.

In any program there are variables that share the same representation but signify very different concepts. For instance, an `Int` could be used to represent a loop index, a timestamp, a file descriptor, or a month; a `Double` could represent a velocity or a temperature on one of several scales; and a `String` could represent a password or the name of a color.

Swift offers two ways to give a name to a type, and they have quite different effects. A `typealias` declaration merely introduces an alternative name for an existing type:

```swift
typealias Celsius = Double
typealias Fahrenheit = Double
```

`Celsius` and `Fahrenheit` are now just other spellings of `Double`. They can make declarations easier to read, but they provide no protection: you can freely mix `Celsius` and `Fahrenheit` values in an expression, which is exactly the mistake we'd like to prevent. Type aliases are useful mainly for abbreviating long generic types, such as `typealias Handler = @Sendable (Request) async throws -> Response`.

To define a *new* type that has the same representation as an existing type but is distinct from it, wrap the existing type in a struct. A struct with a single stored property costs nothing at run time: it has exactly the same size and layout as the property it wraps.

```swift
// swiftpl/ch2/tempconv0
// Package tempconv performs Celsius and Fahrenheit temperature computations.

struct Celsius {
    var value: Double
}

struct Fahrenheit {
    var value: Double
}

let absoluteZeroC = Celsius(value: -273.15)
let freezingC = Celsius(value: 0)
let boilingC = Celsius(value: 100)

func cToF(_ c: Celsius) -> Fahrenheit {
    Fahrenheit(value: c.value * 9 / 5 + 32)
}

func fToC(_ f: Fahrenheit) -> Celsius {
    Celsius(value: (f.value - 32) * 5 / 9)
}
```

This module defines two types, `Celsius` and `Fahrenheit`, for the two units of temperature. Even though both have the same underlying representation, they are not the same type, so they cannot be compared or combined in arithmetic expressions. Distinguishing the types makes it possible to avoid errors like inadvertently combining temperatures in the two different scales; an explicit conversion like `cToF` is required to convert from one to the other.

This is more verbose than we'd like, and a temperature that can't be added to another temperature isn't very useful. Swift lets us fix both problems. First, a type can describe how it should be created from a literal, by conforming to a protocol such as `ExpressibleByFloatLiteral`. Second, a type can define *operators*, like `+` and `<`, as static functions. Third, it can describe how it should be printed by conforming to `CustomStringConvertible` and supplying a `description` property. Here's a fuller version, using *extensions* to add each capability:

```swift
extension Celsius: ExpressibleByFloatLiteral, ExpressibleByIntegerLiteral {
    init(floatLiteral value: Double) { self.value = value }
    init(integerLiteral value: Int) { self.value = Double(value) }
}

extension Celsius: Comparable {
    static func + (a: Celsius, b: Celsius) -> Celsius { Celsius(value: a.value + b.value) }
    static func - (a: Celsius, b: Celsius) -> Celsius { Celsius(value: a.value - b.value) }
    static func < (a: Celsius, b: Celsius) -> Bool { a.value < b.value }
}

extension Celsius: CustomStringConvertible {
    var description: String { "\(value)°C" }
}
```

Now `Celsius` values can be written as literals and combined:

```swift
let c: Celsius = 100
print(c - absoluteZeroC)  // "373.15°C"
let f = cToF(c)
print(c == f)  // compile error: binary operator '==' cannot be applied to 'Celsius' and 'Fahrenheit'
```

The `Comparable` protocol requires `<` and, through its parent protocol `Equatable`, `==`. We didn't write `==`; for a struct whose stored properties are all `Equatable`, the compiler synthesizes it automatically. Conversion functions like `cToF` don't change the value or its representation, but they make the change of meaning explicit.

Many types declare a `description` property of this form because it controls how values of the type appear when printed by `print` or interpolated into a string:

```swift
let c = fToC(Fahrenheit(value: 212))
print(c)  // "100.0°C"
print("Water boils at \(c)")  // "Water boils at 100.0°C"
```

Extensions, operators, and protocol conformances are the subjects of Chapters 6 and 7, so don't worry if their details are not yet clear.

## 2.6. Modules and Files

Modules in Swift serve the same purposes as libraries or packages in other languages, supporting modularity, encapsulation, separate compilation, and reuse. The source code for a module resides in one or more `.swift` files, usually in a directory whose name is the name of the module. Within a package, each module is a *target*. A package may contain several targets, which may depend on each other.

Each module serves as a separate *namespace* for its declarations. Within the `Foundation` module, for example, the type `URL` is just `URL`, and so it is in any file that imports `Foundation`. If two imported modules both declare a `URL`, you can disambiguate by qualifying the name with the module: `Foundation.URL`.

Access control lets us hide information. An `internal` declaration (the default) is visible in every file of its module but not outside it. Only `public` declarations are visible to clients.

To illustrate the basics, suppose that our temperature conversion software has become popular and we want to make it available to the Swift community as a new module. How do we do that?

Let's create a package called `tempconv` with a library target, also called `TempConv` (module names conventionally use `UpperCamelCase`), and an executable that uses it. The manifest lists both targets:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "tempconv",
    products: [
        .library(name: "TempConv", targets: ["TempConv"]),
    ],
    targets: [
        .target(name: "TempConv"),
        .executableTarget(name: "cf", dependencies: ["TempConv"]),
    ]
)
```

The library's source code is in two files, to show how declarations in separate files of a module are accessed. We put the declarations of the types and their literal conformances in one file:

```swift
// swiftpl/ch2/tempconv/Sources/TempConv/TempConv.swift
// Package TempConv performs Celsius and Fahrenheit conversions.

public struct Celsius: Hashable, Sendable {
    public var value: Double
    public init(_ value: Double) { self.value = value }
}

public struct Fahrenheit: Hashable, Sendable {
    public var value: Double
    public init(_ value: Double) { self.value = value }
}

extension Celsius {
    public static let absoluteZero = Celsius(-273.15)
    public static let freezing = Celsius(0)
    public static let boiling = Celsius(100)
}

extension Celsius: CustomStringConvertible {
    public var description: String { "\(value)°C" }
}

extension Fahrenheit: CustomStringConvertible {
    public var description: String { "\(value)°F" }
}
```

and the conversion functions in another:

```swift
// swiftpl/ch2/tempconv/Sources/TempConv/Conv.swift

/// Converts a Celsius temperature to Fahrenheit.
public func cToF(_ c: Celsius) -> Fahrenheit {
    Fahrenheit(c.value * 9 / 5 + 32)
}

/// Converts a Fahrenheit temperature to Celsius.
public func fToC(_ f: Fahrenheit) -> Celsius {
    Celsius((f.value - 32) * 5 / 9)
}
```

Every type, initializer, property, and function that a client needs is marked `public`. Notice in particular the explicit `public init`: the memberwise initializer that the compiler synthesizes for a struct is only `internal`, so a public struct that clients should be able to create needs a public initializer written by hand. This is a deliberate design choice: the struct's author must opt in to making its representation part of its public interface.

The constants are written as *static properties* of `Celsius` rather than as globals. Clients write `Celsius.boiling`, or just `.boiling` wherever the type is known, which avoids cluttering the global namespace. The conformances to `Hashable` and `Sendable` are synthesized by the compiler; we'll explain them in Chapters 4 and 9.

### 2.6.1. Imports

Within a Swift program, every module is identified by its name, like `Foundation` or `TempConv`. To use the public declarations of another module from a file, you must `import` that module in that file:

```swift
// swiftpl/ch2/tempconv/Sources/cf/main.swift
// Cf converts its numeric argument to Celsius and Fahrenheit.
import Foundation
import TempConv

for arg in CommandLine.arguments.dropFirst() {
    guard let t = Double(arg) else {
        FileHandle.standardError.write(Data("cf: invalid number: \(arg)\n".utf8))
        exit(1)
    }
    let f = Fahrenheit(t)
    let c = Celsius(t)
    print("\(f) = \(fToC(f)), \(c) = \(cToF(c))")
}
```

The import declaration makes the module's public names available throughout the file, without qualification. Imports are per file: importing `Foundation` in one file of a module doesn't make it available in the others. It is an error to import a module that the target doesn't depend on in its manifest.

```
$ swift run cf 32
32.0°F = 0.0°C, 32.0°C = 89.6°F
$ swift run cf 212
212.0°F = 100.0°C, 212.0°C = 413.6°F
$ swift run cf -40
-40.0°F = -40.0°C, -40.0°C = -40.0°F
```

An unused import is not an error in Swift, unlike in Go, but it does slow compilation slightly, and the formatter and linters can be configured to flag it.

### 2.6.2. Module Initialization

Swift has no equivalent of Go's `init` functions, which run when a package is loaded. Instead, global variables and static properties are *lazily* initialized: each is initialized the first time it is accessed, not when the program starts. The initialization is guaranteed to happen exactly once, even if several threads access the variable for the first time simultaneously. The exception is top-level code in `main.swift`, whose variables are initialized in order as the code executes.

Lazy initialization means that programs start quickly, since no work is done for globals that are never used, and that the order in which modules are loaded can never matter.

For some variables, like tables of data, an initializer expression may not be the simplest way to set its initial value. In that case, initialize it with a *closure expression* that is called immediately. For example, the following file defines a function `popCount` that returns the number of set bits (bits whose value is 1) in a `UInt64` value, which is called its *population count*. It uses a table built once, the first time it's needed, to precompute the result for each possible 8-bit value, so that `popCount` needn't take 64 steps but can just return the sum of eight table lookups. (This is definitely *not* the fastest algorithm for counting bits; it's just a convenient illustration.)

```swift
// swiftpl/ch2/popcount
// pc[i] is the population count of i.
let pc: [UInt8] = {
    var table = [UInt8](repeating: 0, count: 256)
    for i in 0..<256 {
        table[i] = table[i / 2] + UInt8(i & 1)
    }
    return table
}()

/// Returns the population count (number of set bits) of x.
func popCount(_ x: UInt64) -> Int {
    var n = 0
    for shift in stride(from: 0, to: 64, by: 8) {
        n += Int(pc[Int((x >> UInt64(shift)) & 0xFF)])
    }
    return n
}
```

The braces after `=` define a closure, an anonymous function, and the `()` after the closing brace calls it. The closure's result becomes the initial value of `pc`. Because `pc` is a global in a file other than `main.swift`, the closure runs only the first time `pc` is used.

Of course, in real code you wouldn't write this function at all: every integer type has a property `nonzeroBitCount` that the compiler turns into the machine's population-count instruction where one exists.

**Exercise 2.1:** Add a `Kelvin` type to `TempConv`, with conversion functions to and from the other two scales. Absolute zero is 0 K, and an increase of 1 K is the same as an increase of 1°C.

**Exercise 2.2:** Write a general-purpose unit-conversion program analogous to `cf` that reads numbers from its command-line arguments or from the standard input if there are no arguments, and converts each number into units like temperature in Celsius and Fahrenheit, length in feet and meters, weight in pounds and kilograms, and the like.

**Exercise 2.3:** Rewrite `popCount` to use a loop instead of the explicit sum of eight lookups. Compare the performance of the two versions. (Section 11.4 shows how to compare the performance of different implementations systematically.)

**Exercise 2.4:** Write a version of `popCount` that counts bits by shifting its argument through 64 bit positions, testing the rightmost bit each time. Compare its performance to the table-lookup version and to `nonzeroBitCount`.

**Exercise 2.5:** The expression `x & (x - 1)` clears the rightmost non-zero bit of `x`. Write a version of `popCount` that counts bits by using this fact, and assess its performance.

## 2.7. Scope

A declaration associates a name with a program entity, such as a function or a variable. The *scope* of a declaration is the part of the source code where a use of the declared name refers to that declaration.

Don't confuse scope with lifetime. The scope of a declaration is a region of the program text; it is a compile-time property. The lifetime of a variable is the range of time during execution when the variable can be referred to by other parts of the program; it is a run-time property.

A syntactic *block* is a sequence of statements enclosed in braces like those that surround the body of a function, a loop, or an `if` statement. A name declared inside a block is not visible outside that block. The block encloses its declarations and determines their scope.

At the top level, declarations of the standard library and imported modules are visible in the files that import them. Declarations of types, functions, and global variables in a module are visible throughout that module (subject to `private` and `fileprivate`). Declarations inside a function body are visible from the point of declaration to the end of the enclosing block. Unlike globals, local variables must be declared before they are used.

When the compiler encounters a reference to a name, it looks for a declaration, starting with the innermost enclosing block and working up to the module and finally to imported modules. If the compiler finds no declaration, it reports an "undeclared identifier" error. If a name is declared in both an outer block and an inner block, the inner declaration will be found first. In that case, the inner declaration is said to *shadow* or *hide* the outer one, making it inaccessible:

```swift
func f() {}

let g = "g"

func example() {
    let f = "f"
    print(f)  // "f"; local let f shadows module-level func f
    print(g)  // "g"; module-level let
    print(h)  // compile error: cannot find 'h' in scope
}
```

Within a single block, however, a name may be declared only once; redeclaring it is an error. This differs from Go's `:=`, which can quietly declare a new variable when you meant to assign to an existing one.

Shadowing is used deliberately and constantly with optional binding, where unwrapping a value under the same name is the norm:

```swift
func greet(_ name: String?) {
    if let name = name {
        print("Hello, \(name)")  // name is a String here, not a String?
    }
}
```

Since Swift 5.7, the shorthand `if let name { ... }` means the same thing.

Within a function, blocks may be nested to any depth, so one local declaration can shadow another. Most blocks are created by control-flow constructs like `if` statements and `for` loops. The program below has three different variables called `x` because each declaration appears in a different block. (This example illustrates scope rules, not good style!)

```swift
let x = "hello!"
for x in x {
    let x = x.uppercased()
    print(x, terminator: "")  // "HELLO!" (one letter per iteration)
}
print()
```

The expressions `x` in the sequence position, `x` in the loop body, and `x.uppercased()` each refer to a different declaration. The sequence expression `for x in x` is evaluated in the outer scope, where `x` is the string; the loop variable `x` is a `Character`; and the inner `let x` is a new `String` that shadows the loop variable.

The conditions of `if` and `guard` statements create scopes as well. A name bound by `if let` is visible only within the `if` block. A name bound by `guard let`, by contrast, is visible from the `guard` statement to the end of the *enclosing* block, which is the whole point of `guard`:

```swift
func readConfig(path: String) throws -> Config {
    guard let data = FileManager.default.contents(atPath: path) else {
        throw ConfigError.missing(path)
    }
    // data is in scope here, unwrapped.
    return try JSONDecoder().decode(Config.self, from: data)
}
```

Shadowing can still cause surprises. Consider a program that keeps the current working directory in a module-level variable:

```swift
var cwd = ""

func setUpWorkingDirectory() {
    let cwd = FileManager.default.currentDirectoryPath  // NOTE: wrong!
    print("Working directory = \(cwd)")
}
```

Since `cwd` is declared with `let` in the function, it is a new local constant that shadows the module-level variable, and the global `cwd` remains empty. Swift doesn't warn about this. The fix is to assign rather than declare:

```swift
func setUpWorkingDirectory() {
    cwd = FileManager.default.currentDirectoryPath
    print("Working directory = \(cwd)")
}
```

(In Swift 6, you'll find that mutable global variables are harder to use than this example suggests, because they're shared mutable state that any task could access concurrently. The compiler will insist that `cwd` either be isolated to an actor, typically `@MainActor`, or be replaced by something safer. We'll see why in Chapter 9. It's one more reason to avoid global variables.)
