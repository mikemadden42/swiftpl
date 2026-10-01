# Preface

> "Swift is a general-purpose programming language that's approachable for newcomers and powerful for experts. It is fast, modern, safe, and a joy to write." (From the Swift web site at `swift.org`)

Swift began as a personal project of Chris Lattner at Apple in July 2010. Others joined in 2011, and Apple introduced the language publicly at its Worldwide Developers Conference in June 2014. The aim was ambitious: a language fast enough for systems programming, pleasant enough for writing applications and scripts, and *safe by default*, so that the most common causes of crashes and security holes (null pointers, uninitialized memory, out-of-bounds access, arithmetic overflow, and, since Swift 6, data races) are either rejected by the compiler or stopped the moment they happen, rather than silently corrupting a running program.

Swift reads much like a scripting language, but it's compiled ahead of time, through LLVM, to native machine code. It's statically typed, with type inference that keeps most code free of type annotations. Its heart is *values*: structs, enums, strings, arrays, and dictionaries behave like numbers, so that each variable holds its own independent copy, while classes and actors provide shared, referenced objects where identity matters. Memory is managed by *automatic reference counting* (ARC), not a tracing garbage collector, so cleanup happens predictably, at the moment the last reference disappears.

Swift is best known as the language of apps for iPhone, iPad, and Mac, but it isn't confined to them. It runs web services on Linux, command-line tools on every desktop platform, firmware on microcontrollers, and code in the browser through WebAssembly; it builds Windows and Android programs; and it calls C libraries directly, and much of C++ as well. What attracts teams to it outside Apple's platforms is the same combination that makes it work well on them: performance in the same class as C and C++, with whole categories of their bugs ruled out.

Swift is open source. The compiler, standard library, core libraries, package manager, and the process by which the language evolves are all developed in public at `github.com/swiftlang`. Programs that rely only on the standard library and Foundation generally run unchanged on macOS, Linux, and Windows.

This book is a guide to writing Swift well: clear, idiomatic, and efficient code that takes full advantage of the language and its libraries. It's about the language itself, not about any user-interface framework, and every program in it runs from a terminal.

## The Origins of Swift

No language is invented from nothing, and Swift is open about its debts. Chris Lattner has described it as drawing ideas "from Objective-C, Rust, Haskell, Ruby, Python, C#, CLU, and far too many others to list." Tracing a few of those threads explains a good deal about how Swift looks and behaves.

**C and Objective-C** gave Swift its familiar control flow, its basic numeric types, and its purpose: to compile to efficient machine code that cooperates with operating systems and existing libraries. From its first day, Swift had to call Objective-C and C code and be called by it, and that requirement shaped many early decisions. Objective-C's interleaved method names live on in Swift's *argument labels*, which let a call like `list.insert(item, at: 0)` read like a phrase.

**The ML family**, including Standard ML, OCaml, and Haskell, supplied algebraic data types (Swift's enums with associated values), pattern matching over them, type inference, and the idea that a missing value should be represented by a type, `Optional`, rather than by a null that any variable might be hiding. Haskell's type classes are the ancestors of Swift's protocols with associated types, and **CLU**, an early language built around abstract data types, influenced its generics.

**Rust**, developed during the same years, shaped Swift's later thinking about ownership. Swift 5.9 introduced noncopyable types and the `borrowing` and `consuming` parameter conventions, and Swift 6's compile-time data-race checking resembles Rust's `Send` and `Sync` in spirit.

**C# and JavaScript** popularized the `async`/`await` style that Swift adopted for asynchronous code, and the *actor model*, best known from Erlang, inspired Swift's actors: objects that protect their state by serving one request at a time.

**Python and Ruby** showed that a serious language can also be a pleasant one. Swift tries to keep that lesson: a one-line script needs no ceremony, string interpolation simply works, and `for`-`in` is the ordinary way to loop.

A few of Swift's ideas are distinctively its own. *Protocol-oriented programming*, the combination of protocols carrying default implementations with value types in place of class hierarchies, has become a recognizable style. Swift's strings take the user-perceived character, the Unicode *grapheme cluster*, as their unit, so that a flag emoji built from two code points counts as one character. And its collections use *copy-on-write* to give value semantics without the cost of constant copying.

## The Swift Project

Every language is a response to problems its designers lived with. Swift's was specific: Objective-C was productive and well liked, but it sat on top of C, and C's hazards couldn't be removed without breaking it. Any object pointer could be `nil`, any array index could stray past the end, any integer could wrap around silently. The way forward was a new language that kept what worked about Objective-C while making undefined behavior something you have to ask for, not something that happens by default.

The project describes its priorities as *safe, fast, and expressive*, in that order when they conflict. *Safe* means that the ordinary path can't corrupt memory. Variables must be initialized before use, arrays check their bounds, integers check for overflow, and a value that may be absent says so in its type. Operations that bypass these checks exist, but they're spelled with the word *unsafe* and easy to find. *Fast* means that the safe default has to be efficient enough for systems programming, so the compiler optimizes aggressively, specializes generic code for the types it's used with, and avoids hidden allocation. *Expressive* means the language should let people say what they mean, in code that reads well both in a quick script and in a large codebase.

Swift isn't a minimal language. It has generics, operator overloading, default arguments, error handling with `throws`, initializers and deinitializers, classes with inheritance, macros, attributes, and property wrappers. Its guiding principle for managing that breadth is *progressive disclosure*: a newcomer can write useful programs with a small subset, and the more advanced features stay out of the way until they're needed. This book follows the same path, beginning with plain code and introducing generics, concurrency, reflection, and unsafe programming one at a time.

Changes to the language happen in public. Each proposal is written up, discussed on the Swift Forums, and accepted, revised, or rejected through the *Swift Evolution* process, and the archive of proposals, numbered from `SE-0001` into the hundreds, records the reasoning behind nearly every feature. Swift has kept *ABI stability* on Apple platforms since Swift 5.0, and since Swift 6.0, *language modes* let code written for Swift 5 keep compiling with new compilers while it's migrated to Swift 6's stricter checking one module at a time.

Swift also takes the machine seriously. Value types are stored inline, without heap allocation, wherever possible; collections share storage until they're modified; and concurrency is part of the language, with cheap tasks, compiler-checked sharing between them, and actors that serialize access to state without explicit locks.

The toolchain comes complete: the compiler; an interactive REPL; the LLDB debugger; the Swift Package Manager; a formatter; a language server, SourceKit-LSP, for editors; the DocC documentation compiler; and the Swift Testing library. One command, `swift`, drives most of them, and a package describes itself with conventions plus a single manifest file, `Package.swift`, written in Swift.

## Organization of the Book

This book is written for programmers who already know at least one other language, perhaps Python, JavaScript, Java, C#, C++, or Go. We don't explain what a variable or a loop is; we explain how Swift's variables and loops work, and where they differ from what you may expect.

The first five chapters cover the material every programmer needs, beginning with a quick tour. **Chapter 1** presents about a dozen short, complete programs: adding up numbers from the command line, counting log entries, drawing a picture, downloading web pages, timing network requests concurrently, and running a small web server. **Chapter 2** covers the structure of programs: names, declarations, constants and variables, value and reference types, assignment, new types, modules, and scope. **Chapter 3** covers numbers, Booleans, strings and Unicode, and literals. **Chapter 4** covers arrays, slices, dictionaries, sets, structs, and tuples, explains copy-on-write, and ends by reading JSON from a web service and producing reports from it. **Chapter 5** covers functions and closures, error handling, `defer`, and what happens when a program traps.

The rest of the book focuses on the areas where Swift is most distinctive. **Chapter 6** covers methods, extensions, mutation, composition, and encapsulation; **Chapter 7** covers protocols and generics, including when to prefer each over the other and over enums. **Chapter 8** introduces Swift's concurrency model: `async` and `await`, tasks and task groups, cancellation, and asynchronous sequences. **Chapter 9** deals with state shared between tasks, explaining data races and how Swift prevents them with `Sendable`, mutexes, and actors.

**Chapter 10** explains the Swift Package Manager: modules, targets, and packages; manifests; dependencies and versions; access control across modules; and the `swift` command. **Chapter 11** covers testing with Swift Testing, along with coverage, benchmarking, and profiling. **Chapter 12** covers reflection with `Mirror`, and the compile-time alternatives that Swift generally prefers: `Codable`, key paths, and macros. **Chapter 13** covers memory layout, unsafe pointers, binary data, and calling C, and discusses when stepping outside Swift's safety guarantees is justified.

Most sections end with exercises, which range from small variations on the examples to substantial programs of your own.

Example programs are labeled with the name of the package they belong in, such as `swiftpl/ch1/hello`. To try one, create an executable package with that name and replace its `main.swift` with the example's code (examples that use `@main` go in a file named after their type instead, as Section 2.3.2 explains):

```
$ mkdir hello && cd hello
$ swift package init --type executable
$ # replace Sources/hello/main.swift with the example
$ swift run
¡Hola, mundo! 👋
```

The examples require Swift 6.0 or later; the few that use features from Swift 6.1 or 6.2 say so. To see which version you have:

```
$ swift --version
Swift version 6.2 (swift-6.2-RELEASE)
Target: x86_64-unknown-linux-gnu
```

If you don't have Swift, or need a newer version, follow the instructions at `https://www.swift.org/install`. The recommended installer, `swiftly`, can install several toolchains side by side and switch between them, on macOS and Linux.

## Where to Find More Information

The official site, `https://www.swift.org`, is the place to start. It links to the language's official guide and reference, *The Swift Programming Language* (a different work from this one, despite the shared title), to documentation for the standard library, and to guides for server, command-line, and embedded development.

The Swift Evolution repository, `https://github.com/swiftlang/swift-evolution`, holds every proposal for changing the language, each with its motivation, design, and the alternatives considered. When you want to know why Swift works the way it does, the relevant proposal is usually the best explanation available. The Swift Forums, `https://forums.swift.org`, are where proposals are discussed and where questions are answered, often by the people who built the feature.

The Swift Package Index, `https://swiftpackageindex.com`, catalogs open-source Swift packages, with hosted documentation and information about which platforms and compiler versions each supports.

For quick experiments, `swift repl` (or just `swift`) starts an interactive session, and `swift file.swift` compiles and runs a single file. The REPL is built on the LLDB debugger, so you can set breakpoints and inspect values in it too.

## Acknowledgments

The structure and pacing of this book are modeled on *The Go Programming Language* by Alan A. A. Donovan and Brian W. Kernighan (Addison-Wesley, 2015), one of the finest introductions ever written to a programming language. Its chapter plan, and its conviction that a language is best taught through small, complete, useful programs, explained thoroughly, guided this book. The text, examples, and exercises here are our own, written for Swift, and so are any errors.

We're grateful as well to the Swift community, whose proposals, forum discussions, documentation, and open-source packages describe the language and its ecosystem more completely than any single book could.
