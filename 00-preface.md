# Preface

> "Swift is a general-purpose programming language that's approachable for newcomers and powerful for experts. It is fast, modern, safe, and a joy to write." (From the Swift web site at `swift.org`)

Swift was started in July 2010 by Chris Lattner at Apple. A small team joined the work in 2011, and Apple announced the language at its Worldwide Developers Conference in June 2014. The goals were a language as fast as C for systems work, as expressive as a scripting language for application work, and safe by default, so that whole categories of bugs (null pointer dereferences, uninitialized variables, buffer overruns, integer overflow, and, since Swift 6, data races) are caught by the compiler or stopped at run time instead of quietly corrupting memory.

Swift looks a little like a modern scripting language, but it compiles ahead of time to efficient native code through LLVM. It is statically typed, and its type inference keeps most programs free of type annotations. Values are the center of the language: structs, enums, strings, arrays, and dictionaries all have *value semantics*, so assigning one copies it, logically if not physically. Classes and actors provide reference semantics where shared identity is what you want. Memory is managed by *automatic reference counting* (ARC) rather than a tracing garbage collector, which gives predictable performance and deterministic cleanup.

Swift is well known as the language of iPhone, iPad, and Mac applications, but it is a general-purpose language. It runs servers on Linux, command-line tools, embedded firmware on microcontrollers, WebAssembly modules in the browser, and Windows applications. It can call C directly, and with C++ interoperability enabled, much of C++ too. Teams adopting it often value the same balance that made Go popular: Swift programs run about as fast as C++ programs, yet a large class of crashes and security holes can't happen.

Swift is an open-source project. Its compiler, standard library, core libraries, package manager, and language evolution process are all developed in public at `github.com/swiftlang`, with contributions from a worldwide community. Swift runs on macOS and the other Apple platforms, on Linux, on Windows, on Android, and on bare-metal embedded targets. Programs written for one of these environments that use only the standard library and Foundation generally work without change on the others.

This book is meant to help you start using Swift effectively right away and to use it well, taking full advantage of the language and its libraries to write clear, idiomatic, and efficient programs. It is a book about Swift the language, not about any user-interface framework. Every program here runs from a terminal.

## The Origins of Swift

Successful languages borrow from their ancestors, and you can learn a lot about why a language is the way it is by tracing those influences. Swift's own documentation describes it as drawing ideas "from Objective-C, Rust, Haskell, Ruby, Python, C#, CLU, and far too many others to list."

From **C** and **Objective-C**, Swift inherited its control-flow statements, its basic numeric types, and above all its role as a language that compiles to efficient machine code and works naturally with existing operating systems and libraries. Swift was designed from the first day to call C and Objective-C code and to be called by it, and this constraint shaped many early decisions. Objective-C's named arguments survive as Swift's *argument labels*: `insert(_:at:)` reads like Smalltalk at the call site.

From the **ML** family (Standard ML, OCaml, and Haskell), Swift took algebraic data types, which it calls enumerations with associated values; pattern matching over them; type inference; and the idea that the absence of a value should be a type, `Optional`, rather than a special pointer that every variable might secretly hold. Haskell's type classes are the ancestor of Swift's *protocols* with associated types, and **CLU**, one of the first languages with abstract data types and parameterized modules, influenced the shape of its generics.

From **Rust**, which grew up at the same time, Swift took a growing interest in ownership. Swift 5.9 added noncopyable types (`~Copyable`) and the `borrowing` and `consuming` parameter modifiers, and Swift 6 added a compile-time model of data-race safety that is close in spirit to Rust's `Send` and `Sync`.

From **C#** and **JavaScript**, Swift adopted the `async`/`await` syntax for asynchronous code, and from the **Erlang** and **Actor model** tradition it adopted *actors*, objects that protect their own state by handling one request at a time.

From **Python** and **Ruby**, Swift learned that a language for professionals can still be pleasant: a playground or a one-line script should not need boilerplate, string interpolation should just work, and the `for`-`in` loop should be the usual way to iterate.

Some of Swift's ideas have no clear precedent. Its *protocol-oriented programming* (protocols that carry default implementations through extensions, combined with value types instead of class hierarchies) became a style of its own. Its strings treat a user-perceived character, a Unicode extended grapheme cluster, as the unit of text, so that `"🇨🇦".count` is 1, which few other languages attempt. And its copy-on-write collections give the safety of value semantics with the performance of shared storage.

## The Swift Project

Every language reflects the problems its creators wanted to solve. Swift was born from a specific frustration: Objective-C was productive and well loved, but its C foundation made it unsafe in ways that could not be fixed without breaking it. Every pointer could be `nil`, every array access could run off the end, and every integer could silently overflow. Apple needed a language that kept Objective-C's dynamism where it mattered while removing undefined behavior from ordinary code.

The project's guiding phrase is that Swift should be *safe, fast, and expressive*, in that order of priority when they conflict. *Safe* means that undefined behavior is never the default. Variables must be initialized before use, arrays check their bounds, integer arithmetic traps on overflow, and a value that might be absent says so in its type. Unsafe operations exist, but they are spelled with the word `unsafe` and are easy to find. *Fast* means that the safe default must also be efficient enough for systems work, so the compiler optimizes aggressively, specializes generic code, and avoids hidden allocations wherever it can. *Expressive* means that the language should let you say what you mean, with a syntax that scales from a playground experiment to a large codebase.

Unlike Go, Swift is not a language of radical minimalism. It has generics, operator overloading, default parameter values, error handling with `throws`, initializers and deinitializers, class inheritance, macros, attributes, and property wrappers. The design philosophy is *progressive disclosure*: a beginner can write useful programs knowing only a small subset, and the more advanced features stay out of the way until they are needed. This book follows the same path. The early chapters use a small, plain subset of the language, and later chapters add generics, concurrency, reflection, and unsafe programming one at a time.

Swift's evolution happens in public. Changes to the language and standard library are proposed, debated, and accepted or rejected through the Swift Evolution process, and every proposal (`SE-0001` through the many hundreds since) is archived with its rationale. Since Swift 5.0 the language has maintained *ABI stability* on Apple platforms, and since Swift 6.0 it has kept *source compatibility* through language modes: a module written in Swift 5 mode continues to compile with a Swift 6 compiler, and can be migrated to the stricter Swift 6 mode one module at a time.

Swift encourages awareness of how programs use memory. Value types are stored inline, without a heap allocation or pointer indirection, wherever the compiler can manage it. The standard collections use copy-on-write so that passing an array to a function costs a reference-count increment, not a copy. And since modern computers are parallel machines, Swift 6 makes concurrency a first-class part of the language: tasks are cheap to create, the compiler checks that values shared between them are safe to share, and actors serialize access to mutable state without explicit locks.

The Swift toolchain includes the compiler, a REPL, the debugger LLDB, the Swift Package Manager, a code formatter, a language server (SourceKit-LSP) for editors, a documentation compiler (DocC), and a testing library (Swift Testing). As with Go's `go` tool, a single command, `swift`, drives most of these tasks, and a project's structure is described by convention plus one small manifest file, `Package.swift`, which is itself written in Swift.

## Organization of the Book

We assume that you have programmed in one or more other languages, whether compiled like C, C++, Java, and Go, or interpreted like Python, Ruby, and JavaScript, so we won't spell everything out as if for a total beginner. Surface syntax will be familiar, as will variables and constants, expressions, control flow, and functions.

Chapter 1 is a tutorial on the basic constructs of Swift, introduced through about a dozen programs for everyday tasks like reading and writing files, formatting text, creating images, and talking to Internet clients and servers.

Chapter 2 describes the structural elements of a Swift program: declarations, variables, new types, modules and files, and scope. Chapter 3 discusses numbers, booleans, strings, and literals, and explains how Swift processes Unicode. Chapter 4 describes composite types, that is, types built up from simpler ones using arrays, dictionaries, sets, structs, and tuples, and explains copy-on-write, Swift's way of combining value semantics with efficient sharing. Chapter 5 covers functions and closures and discusses error handling with `throws`, the `defer` statement, and what happens when a program traps.

Chapters 1 through 5 are thus the basics, things that are part of any mainstream imperative language. Swift's syntax and style sometimes differ from other languages, but most programmers will pick them up quickly. The remaining chapters focus on topics where Swift's approach is less conventional: methods, protocols and generics, concurrency, packages, testing, and reflection.

Swift supports classes and inheritance, but idiomatic Swift prefers value types and *protocols*. Methods may be attached to any named type, including structs and enums, and existing types (even `Int` and `String`) can be extended with new methods and new protocol conformances after the fact. Methods are covered in Chapter 6 and protocols, together with the generics they enable, in Chapter 7.

Chapter 8 presents Swift's approach to concurrency: structured tasks, `async`/`await`, task groups, and asynchronous sequences, which play the role that goroutines and channels play in Go. Chapter 9 explains the problems of shared mutable state and Swift's answers to them: the `Sendable` protocol, mutexes, and actors, all checked by the compiler.

Chapter 10 describes the Swift Package Manager, the mechanism for organizing code into modules and packages and for depending on other people's packages. It also shows how to make effective use of the `swift` command for building, running, testing, and formatting.

Chapter 11 deals with testing using the Swift Testing library, whose `@Test` and `#expect` macros keep tests short and their failure messages informative, along with code coverage, benchmarking, and profiling.

Chapter 12 discusses reflection, the ability of a program to examine its own values at run time, through `Mirror`, and the *compile-time* alternatives that Swift prefers: the `Codable` protocols and macros. Chapter 13 explains the details of low-level programming with unsafe pointers and memory layout, how to call C, and when stepping outside Swift's safety guarantees is appropriate.

Each chapter has a number of exercises that you can use to test your understanding of Swift, and to explore extensions and alternatives to the examples from the book.

Each example of more than a few lines is identified by a label such as `swiftpl/ch1/helloworld`. To try one, create a package with that name and put the code in its `main.swift` file:

```
$ mkdir helloworld && cd helloworld
$ swift package init --type executable
$ # replace Sources/helloworld/main.swift with the example
$ swift run
Hello, 世界
```

To run the examples, you will need Swift 6.0 or later. Some examples use features from Swift 6.1 and 6.2, which the text points out.

```
$ swift --version
Swift version 6.2 (swift-6.2-RELEASE)
Target: x86_64-unknown-linux-gnu
```

If the `swift` command on your computer is older or missing, follow the instructions at `https://www.swift.org/install`. The recommended installer, `swiftly`, can install and switch between toolchains on macOS and Linux.

## Where to Find More Information

The best source for more information about Swift is the official web site, `https://www.swift.org`. It links to the official language guide and reference, *The Swift Programming Language* (sometimes called TSPL, and not to be confused with this book), to the standard library documentation, and to getting-started guides for servers, command-line tools, and embedded systems.

The Swift Evolution repository at `https://github.com/swiftlang/swift-evolution` holds every proposal for changing the language, each with a detailed motivation and design discussion. When you wonder *why* Swift does something the way it does, the relevant proposal is usually the best answer. The Swift Forums at `https://forums.swift.org` are where those proposals are discussed, and where you can ask questions of the people who build the language.

The Swift Package Index at `https://swiftpackageindex.com` is a searchable catalog of open-source Swift packages, with hosted documentation and compatibility information for each platform and compiler version.

For quick experiments, run `swift repl` (or just `swift`) to get an interactive prompt, or write a single file and run it with `swift file.swift`. The REPL is a full LLDB session, so you can also set breakpoints and inspect values in it.

## Acknowledgments

This book borrows its shape, its pacing, and many of its example ideas from *The Go Programming Language* by Alan A. A. Donovan and Brian W. Kernighan (Addison-Wesley, 2015), which remains one of the best introductions to any programming language. We have tried to follow its approach of teaching a language through small, complete, useful programs, and of explaining not only what the language does but why. The prose and code here are written for Swift; any errors in them are ours.

We are also grateful to the Swift community, whose evolution proposals, forum posts, and open-source packages document the language more thoroughly than any one book can.
