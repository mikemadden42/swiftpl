# Appendix: Swift for Go Programmers

This book's structure follows that of a well-known book about Go, and many readers come to Swift with Go experience. This appendix is for them: a quick map from Go's idioms to Swift's, with pointers to the chapters that explain each Swift feature in full. The two languages share a lot, including compilation to native code, a strong standard library, first-class functions and closures, lexical scope, and an emphasis on simple concurrency, but they make different choices in nearly every area, and knowing the differences up front will save you from writing Go in Swift.

## A.1. At a Glance

| Go | Swift | See |
|---|---|---|
| package, import path | module, target, package | Ch. 10 |
| exported by capitalization | `public`, `internal`, `private`, ... | 6.6, 10.5 |
| zero values | definite initialization | 2.3 |
| `var x = ...`, `x := ...` | `var x = ...`, `let x = ...` | 2.3 |
| pointer receivers, `*T` | `mutating` methods, `inout`, classes | 2.3.2, 6.2 |
| struct embedding | composition, protocol extensions | 4.4.3, 6.3 |
| implicit interfaces | explicit protocol conformance | Ch. 7 |
| `iota` constants | enums, option sets | 3.6 |
| tagged structs, interface unions | enums with associated values | 4.7 |
| `nil` pointers, maps, slices | optionals (`T?`) | 4.8 |
| `switch`, type switch | `switch` with pattern matching | 4.9, 7.13 |
| slices | arrays, `ArraySlice`, copy-on-write | 4.1, 4.2 |
| maps | dictionaries and sets | 4.3 |
| `(value, error)` returns | `throws`, `try`, `do`-`catch` | 5.4 |
| `defer` (function scope) | `defer` (block scope) | 5.8 |
| `panic`, `recover` | traps (no recovery) | 5.9, 5.10 |
| generics with constraints | generics, associated types, packs | 7.7 |
| goroutines | tasks | 8.1 |
| `sync.WaitGroup` | task groups | 8.5 |
| channels | `AsyncStream`, `AsyncChannel` | 8.4 |
| `select` | racing tasks, merged streams | 8.7 |
| `context.Context` cancellation | task cancellation | 8.9 |
| `sync.Mutex`, `sync/atomic` | `Mutex`, `Atomic`, actors | 9.2, 9.3 |
| race detector | compile-time checking, TSan | 9.1, 9.6 |
| `go build`, `go test`, `gofmt` | `swift build`, `swift test`, `swift format` | 10.7, Ch. 11 |
| `reflect` | `Mirror`, `Codable`, macros | Ch. 12 |
| `unsafe`, cgo | unsafe pointers, direct C import | Ch. 13 |
| garbage collection | automatic reference counting | 2.3.4 |

## A.2. Declarations and Initialization

Go gives every variable a zero value. Swift has no zero values; instead, the compiler proves that every variable is assigned before it's read, and refuses to compile code where it can't (Section 2.3). Where Go code relies on a zero value meaning "nothing yet," Swift code uses an optional or an explicit default.

Swift distinguishes `let` (a constant) from `var` (a variable), and idiomatic Swift uses `let` wherever possible. There's no `:=`; type inference works with both `let` and `var`.

```go
var count int          // 0
name := "gopher"
```

```swift
var count = 0          // must be initialized, or assigned before use
let name = "gopher"
```

Visibility isn't determined by capitalization. Declarations are `internal` (visible throughout their module) by default, and must be marked `public` to be used from other modules (Sections 6.6 and 10.5). Naming conventions are also different: Swift uses `lowerCamelCase` for functions and properties and `UpperCamelCase` for types, and its API guidelines favor descriptive names with argument labels (`insert(_:at:)`) over short ones (Section 2.1).

## A.3. Values, Pointers, and Methods

In Go, you choose between value and pointer receivers, and pass `*T` when a function must modify its argument. Swift has no everyday pointers (Section 2.3.2). Instead:

- **Value types** (structs, enums, tuples, and all the standard collections) are copied on assignment. A method that modifies a struct is marked `mutating`, and a function that modifies a caller's variable takes an `inout` parameter, called with `&`.
- **Reference types** (classes and actors) are shared on assignment, like a Go pointer to a struct.

```go
type Counter struct{ n int }
func (c *Counter) Incr() { c.n++ }
```

```swift
struct Counter {
    var n = 0
    mutating func incr() { n += 1 }
}
```

Swift's collections are values. Assigning a Go slice shares its backing array, so a change through one slice is visible through another; assigning a Swift array behaves as a copy, made cheap by copy-on-write (Section 4.2). There's no Swift equivalent of the `s = append(s, x)` idiom: `append` modifies the array in place, and aliasing can't be observed.

Methods can be added to any type, including standard library types and types from other modules, with an *extension* (Section 6.1.1). Go's restriction to types defined in the same package doesn't apply.

## A.4. Interfaces and Protocols

Go interfaces are satisfied implicitly: any type with the right methods conforms. Swift protocols require an explicit declaration, but because conformance can be added in an extension, after the fact and even to types you don't own, they're nearly as flexible:

```go
type Shape interface{ Area() float64 }
// Square satisfies Shape simply by having an Area method.
```

```swift
protocol Shape { var area: Double { get } }
extension Square: Shape {}  // explicit, but can be added anywhere
```

Swift protocols can do things Go interfaces can't: require properties, initializers, static members, and operators; provide *default implementations* in extensions (Section 6.3.2); and have *associated types* (Section 7.7.3). Go's struct embedding, which promotes an inner type's methods, has no direct equivalent; Swift shares behavior through protocol extensions, and data through plain composition (Section 6.3).

Swift distinguishes two ways of using a protocol as a type. `any Shape` is an *existential*, like a Go interface value, which boxes a value of some conforming type. `some Shape` is a generic parameter, resolved at compile time and specialized for each concrete type, with no boxing. Prefer `some` (Section 7.5).

## A.5. Errors

Go returns errors as ordinary values and checks them with `if err != nil`. Swift functions that can fail are marked `throws`, every call to them is marked `try`, and errors propagate to the caller automatically unless caught with `do`-`catch` (Section 5.4):

```go
data, err := os.ReadFile(path)
if err != nil {
    return nil, fmt.Errorf("reading config: %w", err)
}
```

```swift
let data: Data
do {
    data = try Data(contentsOf: url)
} catch {
    throw ConfigError.unreadable(path: path, underlying: error)
}
```

The control flow is still explicit, since every point where an error can escape is marked `try`, but the repetitive checking disappears. An error is any type conforming to `Error`, usually an enum whose cases name the ways an operation can fail; `catch` clauses can match specific cases (Section 5.4.2), much as Go code uses `errors.Is` and `errors.As`. Swift 6 also supports *typed throws*, `throws(ParseError)`, for functions with a closed set of failures.

When there's only one way to fail and no explanation is needed, Swift functions return an optional instead of throwing, where Go would use a `(value, ok)` pair: dictionary lookups, string-to-number conversion, and searches all return `nil` for "not found."

## A.6. `nil`, Optionals, and Enums

In Go, pointers, maps, slices, channels, functions, and interfaces can all be `nil`, and dereferencing a `nil` pointer panics. In Swift, *no* type can be `nil` except an optional, written `T?`, and the compiler won't let you use an optional as if it held a value until you've unwrapped it with `if let`, `guard let`, `??`, optional chaining, or pattern matching (Section 4.8). Null-pointer crashes don't happen in ordinary Swift code.

Go's `iota` constants become Swift enums (Section 3.6), which are distinct types with exhaustiveness checking. Swift enums can also carry associated values, which makes them *sum types* (Section 4.7): a value is exactly one of several cases, each with its own data. Where Go code would use an interface with a type switch, or a struct with a "kind" field and several sometimes-meaningful fields, Swift code uses an enum, and the compiler ensures that every `switch` handles every case:

```swift
enum Shape {
    case circle(radius: Double)
    case rectangle(width: Double, height: Double)
}
```

Swift's `switch` is more capable than Go's, matching on tuples, enum cases, ranges, optionals, and types, and binding parts of the value as it goes (Section 4.9). Cases don't fall through; there's no `break` needed, and `fallthrough` exists but is rare.

## A.7. Generics

Go added generics in version 1.18 with deliberately limited features. Swift has had generics from the start, and they're central to the standard library (Section 7.7). Constraints are protocols, as in `func largest<T: Comparable>(_ values: [T]) -> T?`, and `where` clauses express relationships between types. Protocols can have associated types, so that `Sequence`'s element type, for example, is part of the protocol. Extensions can add methods only for certain type arguments, and *conditional conformance* makes `[T]` `Equatable` exactly when `T` is. Parameter packs (Section 7.7.5) let a function or type take any number of type parameters. The compiler specializes generic code for the types it's used with, so generics carry no run-time cost in optimized builds.

## A.8. Concurrency

Go's model is goroutines communicating over channels. Swift's is *structured concurrency* (Chapter 8):

- A `go f()` statement starts work that runs independently. Swift's closest equivalent is `Task { await f() }`, but idiomatic Swift prefers *child tasks* created with `async let` or a task group, which can't outlive the scope that creates them, so they can't leak.
- `sync.WaitGroup` becomes a task group: `withTaskGroup` doesn't return until every child has finished, and if a child throws, the remaining children are cancelled automatically (Section 8.5).
- Channels become asynchronous sequences. `AsyncStream` is a buffered channel with separate sending and receiving ends; `AsyncChannel`, from the `swift-async-algorithms` package, is closer to an unbuffered Go channel (Section 8.4).
- `select` becomes either a race between child tasks in a task group, where the first result wins and the rest are cancelled, or a single stream fed by several producers (Section 8.7).
- `context.Context` cancellation is built in: cancelling a task cancels all its children, and asynchronous APIs check for cancellation themselves (Section 8.9).

The biggest difference is the `async` keyword. In Go, any function can block. In Swift, a function that may suspend must be marked `async`, and calls to it marked `await`, so every point where a function can pause is visible in the source (Section 8.1.1).

For shared state, Swift has a `Mutex` that *contains* the data it protects, so the data can't be reached without holding the lock (Section 9.2), and *actors*, objects that serialize access to their state, playing the role of a goroutine that owns some data and serves requests for it (Section 9.3). Most importantly, Swift 6 checks for data races *at compile time*: code that would let two tasks access the same mutable state unsafely doesn't compile (Section 9.1). Go's race detector finds races that occur during a run; Swift's compiler rejects programs that could have one.

## A.9. `defer`, `panic`, and `recover`

Swift's `defer` runs when control leaves the enclosing *block*, not the whole function (Section 5.8), so a `defer` inside a loop body runs at the end of each iteration, which is usually what you wanted in Go but didn't get.

Swift has no `recover`. Programming errors, such as an out-of-range index, a forced unwrap of `nil`, or an integer overflow, *trap*, ending the program immediately (Section 5.9). Expected failures are thrown errors, which can always be caught. Integer overflow, which silently wraps in Go, traps in Swift unless you use the explicit wrapping operators `&+`, `&-`, and `&*` (Section 3.1.1).

## A.10. Memory Management

Go uses a tracing garbage collector. Swift uses *automatic reference counting* (Section 2.3.4): each class instance counts the references to it and is freed the moment the count reaches zero, so cleanup in `deinit` is deterministic and there are no collection pauses. The cost is that reference cycles aren't collected automatically; they must be broken with `weak` or `unowned` references, and the most common source of them, closures that capture `self`, is handled with a capture list such as `[weak self]` (Section 5.6.1).

## A.11. Tooling

The `swift` command plays the role of the `go` command (Section 10.7):

| Go | Swift |
|---|---|
| `go mod init` | `swift package init` |
| `go build` | `swift build` (`-c release` for optimized builds) |
| `go run .` | `swift run` |
| `go test ./...` | `swift test` |
| `gofmt` | `swift format` |
| `go.mod`, `go.sum` | `Package.swift`, `Package.resolved` |
| `go doc` | DocC (`swift package generate-documentation`) |
| `go vet` | compiler warnings and strict concurrency checking |

A Swift package's manifest, `Package.swift`, is itself written in Swift, and declares targets, products, and dependencies with version requirements (Chapter 10). Swift Testing, with its `@Test` functions and `#expect` macro, plays the role of Go's `testing` package and its table-driven tests (Chapter 11); benchmarks come from separate packages rather than the test framework (Section 11.4).

## A.12. What You'll Miss, and What You'll Gain

Go programmers often miss Go's fast compilation, its small language that fits in one's head, and the single obvious way to do most things. Swift is a larger language and compiles more slowly, and it offers more than one way to solve many problems, which this book tries to navigate by recommending one.

In exchange, Swift offers optionals instead of `nil` crashes, enums with associated values and exhaustive pattern matching, value semantics for collections, `throws` instead of repeated error checks, a richer generics system, and compile-time data-race safety. Each of these removes a category of bugs that Go programmers guard against by discipline and testing. When you find yourself reaching for a Go idiom, look up its counterpart in the table at the start of this appendix; the chapter it points to will show the Swift way.
