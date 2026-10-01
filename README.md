# Programming in Swift

*A book about Swift 6, modeled on the structure of Donovan & Kernighan's* The Go Programming Language.

The chapters follow the same overall plan as the Go book, a tutorial, then the language from the ground up, then concurrency, packages, testing, reflection, and low-level programming, but the text, examples, and exercises are original and written for Swift. Where the two languages differ, chapters cover the Swift way of solving the same problem: structured concurrency and asynchronous sequences rather than channels, copy-on-write collections rather than slices, protocol extensions rather than struct embedding.

Example programs carry a label such as `swiftpl/ch1/hello`, which names the package directory where you would keep the code. Unless the text says otherwise, each is the `main.swift` of an executable package. A label ending in `(continued)` adds to the package of the previous example with the same label, and one ending in `(builds on NAME)` revises the earlier program `NAME`: start from a copy of that package.

## Contents

**[Preface](00-preface.md)**
- The Origins of Swift
- The Swift Project
- Organization of the Book
- Where to Find More Information
- Acknowledgments

**[1. Tutorial](ch01-tutorial.md)**
- 1.1 Hello, World
- 1.2 Command-Line Arguments
- 1.3 Counting Things
- 1.4 Drawing a Spirograph
- 1.5 Downloading a URL
- 1.6 Timing Requests Concurrently
- 1.7 A Web Server
- 1.8 Loose Ends

**[2. Program Structure](ch02-program-structure.md)**
- 2.1 Names
- 2.2 Declarations
- 2.3 Variables
- 2.4 Assignments
- 2.5 Type Declarations
- 2.6 Modules and Files
- 2.7 Scope

**[3. Basic Data Types](ch03-basic-data-types.md)**
- 3.1 Integers
- 3.2 Floating-Point Numbers
- 3.3 Complex Numbers
- 3.4 Booleans
- 3.5 Strings
- 3.6 Literals and Constants

**[4. Composite Types](ch04-composite-types.md)**
- 4.1 Arrays
- 4.2 Slices and Copy-on-Write
- 4.3 Dictionaries and Sets
- 4.4 Structs and Tuples
- 4.5 JSON
- 4.6 Text Templates with String Interpolation

**[5. Functions](ch05-functions.md)**
- 5.1 Function Declarations
- 5.2 Recursion
- 5.3 Multiple Return Values
- 5.4 Errors
- 5.5 Function Values
- 5.6 Closures
- 5.7 Variadic Functions
- 5.8 Deferred Statements
- 5.9 Traps
- 5.10 Recovering from Failure

**[6. Methods](ch06-methods.md)**
- 6.1 Method Declarations
- 6.2 Mutating Methods and Reference Types
- 6.3 Composing Types with Extensions and Protocols
- 6.4 Method Values and Key Paths
- 6.5 Example: A Ring Buffer
- 6.6 Encapsulation

**[7. Protocols](ch07-protocols.md)**
- 7.1 Protocols as Contracts
- 7.2 Protocol Types
- 7.3 Protocol Conformance
- 7.4 Parsing Flags with a Protocol
- 7.5 Existential Values: `any` and `some`
- 7.6 Sorting with `Comparable`
- 7.7 Generics and Associated Types
- 7.8 The `Error` Protocol
- 7.9 Example: Expression Evaluator
- 7.10 Type Casting
- 7.11 Discriminating Errors with Casts
- 7.12 Querying Behaviors with Conditional Casts
- 7.13 Type Switches
- 7.14 Example: Reading a News Feed with XMLParser
- 7.15 Choosing Between Protocols, Generics, and Enums

**[8. Tasks and Asynchronous Sequences](ch08-tasks.md)**
- 8.1 Tasks and `async`/`await`
- 8.2 Example: Concurrent Clock Server
- 8.3 Example: Concurrent Reminder Server
- 8.4 Async Streams
- 8.5 Looping in Parallel
- 8.6 Example: Concurrent Web Crawler
- 8.7 Racing Tasks
- 8.8 Example: Concurrent Directory Traversal
- 8.9 Cancellation
- 8.10 Example: Chat Server

**[9. Concurrency with Shared State](ch09-shared-state.md)**
- 9.1 Data Races and `Sendable`
- 9.2 Mutual Exclusion: `Mutex`
- 9.3 Actors
- 9.4 Actor Reentrancy
- 9.5 Lazy Initialization
- 9.6 Strict Concurrency Checking and the Thread Sanitizer
- 9.7 Example: Concurrent Non-Blocking Cache
- 9.8 Tasks and Threads

**[10. Packages and the Swift Package Manager](ch10-packages.md)**
- 10.1 Modules, Targets, Products, and Packages
- 10.2 Creating and Laying Out a Package
- 10.3 The Package Manifest
- 10.4 Depending on Other Packages
- 10.5 Access Levels Across Modules
- 10.6 Imports and Module Interfaces
- 10.7 The `swift` Tool

**[11. Testing](ch11-testing.md)**
- 11.1 The `swift test` Tool
- 11.2 `@Test` Functions
- 11.3 Coverage
- 11.4 Benchmarks
- 11.5 Profiling
- 11.6 Documentation Examples

**[12. Reflection](ch12-reflection.md)**
- 12.1 Why Reflection?
- 12.2 `Mirror`
- 12.3 `inspect`, a Recursive Value Printer
- 12.4 Example: Encoding S-Expressions
- 12.5 Setting Values with Key Paths
- 12.6 Example: Decoding S-Expressions
- 12.7 Customizing Keys with `CodingKeys`
- 12.8 Macros: Reflection at Compile Time
- 12.9 A Word of Caution

**[13. Low-Level Programming](ch13-low-level.md)**
- 13.1 `MemoryLayout`: Size, Alignment, Stride, and Offset
- 13.2 Unsafe Pointers
- 13.3 Example: Reading a Binary File Header
- 13.4 Calling C Code with Direct Interoperation
- 13.5 Another Word of Caution

## Requirements

The examples target Swift 6 in the Swift 6 language mode, with strict concurrency checking, on macOS or Linux. Some use features from Swift 6.1 or 6.2, which the text points out. Chapter 10 explains how to declare package dependencies. The packages used in the book are:

| Package | Used for | Chapters |
|---|---|---|
| [swift-argument-parser](https://github.com/apple/swift-argument-parser) | command-line options | 2, 7, 10 |
| [Hummingbird](https://github.com/hummingbird-project/hummingbird) | web servers | 1, 4 |
| [SwiftNIO](https://github.com/apple/swift-nio) and [swift-nio-extras](https://github.com/apple/swift-nio-extras) | TCP servers | 8 |
| [swift-numerics](https://github.com/apple/swift-numerics) | complex numbers | 3 |
| [swift-crypto](https://github.com/apple/swift-crypto) | SHA-256 digests | 4, 8 |
| [SwiftSoup](https://github.com/scinfu/SwiftSoup) | HTML parsing | 5, 8 |
| [swift-async-algorithms](https://github.com/apple/swift-async-algorithms) | channels and sequence operators | 8 |
| [swift-collections](https://github.com/apple/swift-collections) | `Deque` | 5, 8 |
| [swift-system](https://github.com/apple/swift-system) | `Errno`, `FilePath` | 7, 10 |
| [swift-log](https://github.com/apple/swift-log) | logging | 5 |
| [swift-syntax](https://github.com/swiftlang/swift-syntax) | macros | 12 |
| [package-benchmark](https://github.com/ordo-one/package-benchmark) | benchmarks | 11 |
| [swift-docc-plugin](https://github.com/swiftlang/swift-docc-plugin) | documentation | 10 |

Chapter 13's SQLite example also needs the SQLite development library (`libsqlite3-dev` on Debian and Ubuntu); on Apple platforms, SQLite is part of the SDK.

## License

The text and the example code of this book are released under the [MIT License](LICENSE). Copyright © 2026 Michael Madden.
