# 10. Packages and the Swift Package Manager

A modest-size program today might contain 10,000 functions. Yet its author need think about only a few of them and design even fewer, because the vast majority were written by others and made available for reuse through modules and packages.

Swift comes with a standard library and core libraries (Foundation, Dispatch, XCTest, Swift Testing) that cover the needs of most applications, and the Swift community has published thousands more packages, many of which can be found at `swiftpackageindex.com`. In this chapter, we'll show how to use existing packages and create new ones.

Swift also comes with the Swift Package Manager (SwiftPM), a sophisticated but simple-to-use tool for managing modules and packages. At the beginning of the book, we showed how to use `swift run` to build and run example programs. In this chapter, we'll look at the concepts underlying the package manager and how to use it.

## 10.1. Introduction

The purpose of any module system is to make the design and maintenance of large programs practical by grouping related features together into units that can be easily understood and changed, independent of the other parts of the program. This *modularity* allows modules to be shared and reused by different projects, distributed within an organization, or made available to the wider world.

Swift's terminology distinguishes three levels:

- A *module* is a unit of code distribution and of namespacing: a set of Swift source files that are compiled together, whose declarations share a namespace, and which other modules can `import`. `Foundation`, `NIOCore`, and `TempConv` are modules.
- A *target* is SwiftPM's description of how to build one module (or a test bundle, or a plugin): its source directory, its dependencies, and its build settings.
- A *package* is a unit of *versioning* and *distribution*: a directory, usually a Git repository, containing a `Package.swift` manifest that declares one or more targets, and the *products* (libraries and executables) that it makes available to other packages.

Each module provides a separate namespace for its declarations, so that a name declared in one module doesn't collide with the same name in another. Each module also provides an *encapsulation* boundary: by controlling which declarations are visible outside the module with access levels, a module's author can hide implementation details, so that clients depend only on the module's public interface.

Swift's module system has a different emphasis from Go's. Go's packages are small and numerous, and a Go program typically imports dozens of them, each identified by an import path. In Swift, modules tend to be larger, with a single module often corresponding to what Go would spread across several packages, and the code within a module is not further subdivided into namespaces except by types. A Swift package with three or four targets is typical; one with forty would be unusual.

Compilation speed is a concern for every large project, and Swift is not as fast to compile as Go. Swift's rich type inference and generics give the compiler more to do. The module system helps. Each module is compiled into a *module interface* that records the types and signatures of its public declarations, so modules that depend on it can be compiled without reparsing its source. And the build system compiles independent modules in parallel, and recompiles only the files affected by a change. Breaking a large module into smaller ones along natural boundaries often speeds up incremental builds.

## 10.2. Package Identity and Dependencies

Each package that you depend on is identified by its *location*, usually a Git URL:

```swift
.package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.0"),
```

The URL is the package's *identity*, and SwiftPM ensures that only one copy of each package is used in a build, even if it's depended on by several other packages. The last path component of the URL, minus `.git`, is used as the package's name in references from targets, as in `.product(name: "ArgumentParser", package: "swift-argument-parser")`.

Packages can also be fetched from a *package registry*, which identifies packages by a scope and name, like `apple.swift-argument-parser`, and serves source archives directly rather than Git repositories. Registries are mostly used inside organizations; public packages are almost always referenced by URL.

### 10.2.1. Versions

Packages are versioned using *semantic versioning*: a version has the form `MAJOR.MINOR.PATCH`, and by convention, the major version is incremented for changes that break the package's API, the minor version for backward-compatible additions, and the patch version for backward-compatible bug fixes. A Git tag such as `1.5.0` marks each release.

A dependency declaration specifies a *requirement*, a range of versions you're willing to accept:

```swift
.package(url: "...", from: "1.5.0")  // 1.5.0 ..< 2.0.0 (the usual choice)
.package(url: "...", .upToNextMinor(from: "1.5.0"))  // 1.5.0 ..< 1.6.0
.package(url: "...", exact: "1.5.2")  // exactly 1.5.2 (avoid)
.package(url: "...", "1.5.0"..<"1.8.0")  // an explicit range
.package(url: "...", branch: "main")  // the tip of a branch (for development only)
.package(url: "...", revision: "a1b2c3d")  // a specific commit
.package(path: "../TempConv")  // a local package on disk
```

When you build, SwiftPM *resolves* the dependency graph: it finds, for every package in the graph, the newest version that satisfies the requirements of every package that depends on it. If no such version exists, because two of your dependencies require incompatible versions of a third, resolution fails and SwiftPM explains the conflict.

The result of resolution is recorded in a file called `Package.resolved`, which lists the exact version and commit of every package in the graph. Subsequent builds use exactly those versions, so builds are reproducible, until you ask for newer ones:

```
$ swift package resolve          # resolve according to Package.resolved, if present
$ swift package update           # pick the newest allowed versions and update Package.resolved
$ swift package update NIOCore   # update just one package
$ swift package show-dependencies
.
├── swift-nio<https://github.com/apple/swift-nio.git@2.86.0>
│   ├── swift-collections<https://github.com/apple/swift-collections.git@1.2.1>
│   ├── swift-atomics<https://github.com/apple/swift-atomics.git@1.3.0>
│   └── swift-system<https://github.com/apple/swift-system.git@1.6.1>
└── swift-argument-parser<https://github.com/apple/swift-argument-parser.git@1.6.1>
```

(The version numbers you see will differ.) For an application or executable, commit `Package.resolved` to source control, so that everyone builds with the same versions. For a library, it's common not to, since the library's clients will do their own resolution anyway.

SwiftPM requires that a dependency graph contain at most one version of each package. That's simpler than Go's approach, which allows multiple major versions of a module to coexist under different import paths, but it means that package authors must be careful about introducing new major versions, since a client can't use version 1 and version 2 of the same package at once.

## 10.3. The Package Manifest

The manifest, `Package.swift`, is a Swift program that constructs a value of type `Package` from the `PackageDescription` module. Because it's ordinary Swift, it can use variables and conditionals, but it should be kept simple and declarative, since tools need to evaluate it quickly. Here is a manifest for a package with a library, an executable that uses it, and tests:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "eval",
    platforms: [.macOS(.v14), .iOS(.v17)],
    products: [
        .library(name: "Eval", targets: ["Eval"]),
        .executable(name: "calc", targets: ["calc"]),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.0"),
    ],
    targets: [
        .target(name: "Eval"),
        .executableTarget(
            name: "calc",
            dependencies: [
                "Eval",
                .product(name: "ArgumentParser", package: "swift-argument-parser"),
            ]
        ),
        .testTarget(name: "EvalTests", dependencies: ["Eval"]),
    ]
)
```

The first line is a special comment that specifies the *tools version*, the minimum version of SwiftPM that can build this package, and also selects the manifest API and default settings. With tools version 6.0 or later, targets are compiled in the Swift 6 language mode by default.

`platforms` sets the minimum deployment target for Apple platforms; it has no effect on Linux or Windows. It matters because many APIs, such as `Mutex` and newer Foundation features, are available only on recent versions of Apple's operating systems.

`products` lists what the package makes available to *other* packages. Targets that aren't part of any product are internal to the package. An executable product can be run with `swift run calc`, and a library product can be depended on by name.

`targets` lists the modules to build. By convention, the sources of a target named `X` are in `Sources/X` and those of a test target in `Tests/X`; you can override the location with a `path:` argument, but it's better not to. The kinds of target are:

```
.target                 a library module
.executableTarget       a module that builds an executable (has main.swift or @main)
.testTarget             a module of tests, run by swift test
.macro                  a compiler plug-in implementing macros (Chapter 12)
.plugin                 a SwiftPM plug-in: a build tool or a command
.systemLibrary          a module map exposing a C library installed on the system (Chapter 13)
.binaryTarget           a prebuilt XCFramework or artifact bundle
```

A target may contain C, C++, or Objective-C sources instead of Swift, in which case its public headers go in an `include` directory and are exposed to Swift as a module automatically. A target may also include *resources* (images, data files, localized strings) declared with `resources: [.process("Resources")]`, which are accessed at run time through the generated `Bundle.module`.

Each target can specify build settings:

```swift
.target(
    name: "Eval",
    swiftSettings: [
        .swiftLanguageMode(.v6),
        .enableUpcomingFeature("ExistentialAny"),
        .define("ENABLE_TRACING", .when(configuration: .debug)),
        .unsafeFlags(["-Ounchecked"]),  // not allowed in dependencies
    ]
)
```

`.define` sets a compilation condition that can be tested with `#if ENABLE_TRACING`. `.unsafeFlags` passes arbitrary flags to the compiler, and for safety, SwiftPM refuses to build a package that uses them when it's a dependency of another package.

### 10.3.1. Platform-Specific Code

Swift has no equivalent of Go's file-name conventions (`_linux.go`) or build tags; instead, code that differs by platform is written with *conditional compilation* blocks:

```swift
#if os(Linux)
import Glibc
#elseif os(Windows)
import WinSDK
#elseif canImport(Darwin)
import Darwin
#endif

#if arch(arm64)
// ...
#endif

#if DEBUG
print("debug build")
#endif
```

The conditions `os()`, `arch()`, `canImport()`, `compiler(>=6.0)`, `swift(>=6.0)`, `targetEnvironment(simulator)`, and `hasFeature()` cover most needs. Code in an inactive `#if` block is parsed but not type-checked, so it can refer to APIs that don't exist on the current platform.

Dependencies can be made conditional too, so that a target uses a library only on the platforms that need it:

```swift
.target(
    name: "Net",
    dependencies: [
        .product(name: "NIOTransportServices", package: "swift-nio-transport-services",
                 condition: .when(platforms: [.macOS, .iOS])),
    ]
)
```

## 10.4. Import Declarations

A Swift source file may contain any number of `import` declarations, which must appear before any other declarations. Each one names a module:

```swift
import Foundation
import NIOCore
import TempConv
```

An `import` makes all the public declarations of the module visible in the file, unqualified. If two imported modules declare the same name, the ambiguity must be resolved where the name is used, by qualifying it with the module name:

```swift
import Foundation
import SystemPackage

let p1 = FilePath("/tmp")  // SystemPackage.FilePath
let d = Foundation.Data()  // explicitly qualified
```

Unlike Go, Swift has no way to rename a module on import. In practice, name conflicts between modules are rare, because Swift names are more descriptive than Go's (types are not usually named after their package), and module qualification resolves those that occur.

### 10.4.1. Import Variants

A few variations on `import` are worth knowing.

*Scoped imports* import a single declaration: `import struct Foundation.URL` or `import func Darwin.sqrt`. They're occasionally used to make a dependency explicit, but in practice they still make the module's extensions and operators visible, so they're less precise than they appear.

*Access-level imports* (Swift 6) control whether a dependency is part of your module's public interface. By default, every import is effectively `public`, meaning that clients of your module may see types from it in your API. Marking an import `internal` or `private` declares that the dependency is an implementation detail, and the compiler checks that none of its types appear in your public declarations:

```swift
internal import SwiftSoup  // our public API doesn't mention SwiftSoup's types
public import Foundation  // our public API uses Date and URL
```

This makes it possible to replace an implementation dependency without breaking clients, and it lets the build system avoid recompiling clients when the dependency changes. The upcoming-feature flag `InternalImportsByDefault` reverses the default, making every import `internal` unless marked `public`, and is likely to become the default in a future language mode.

`@testable import` makes a module's `internal` declarations visible, as if they were `public`. It's allowed only in test targets, and only when the module was built with testing enabled (as debug builds are). We'll use it in Chapter 11.

`@preconcurrency import` suppresses `Sendable`-related diagnostics for types from a module that hasn't yet adopted Swift's concurrency annotations. It's a migration aid.

### 10.4.2. No Side-Effect Imports

Go has *blank imports*, `import _ "image/png"`, which import a package only for the side effects of its initialization, typically to register a decoder or driver with some central registry. Swift has nothing equivalent, because, as we saw in Section 2.6, Swift modules have no initialization code: global variables are initialized lazily, on first use, and nothing runs merely because a module is imported or linked.

Swift libraries therefore use explicit registration, or avoid registries altogether. Rather than registering image decoders by import, a Swift API would take a list of decoder types, or define a protocol and let the caller pass a value that conforms to it:

```swift
let decoders: [any ImageDecoder.Type] = [PNGDecoder.self, JPEGDecoder.self, GIFDecoder.self]
let image = try decodeImage(data, using: decoders)
```

This is more verbose than a blank import, but it makes the program's dependencies visible in its code, and it means that whether a format is supported doesn't depend on what some other file happened to import.

## 10.5. Access Levels Across Modules

Section 6.6 introduced Swift's access levels. Several of them matter specifically at module boundaries.

`internal` (the default) hides a declaration from other modules. Most declarations in a module should stay `internal`.

`public` exposes a declaration to other modules. A public type's members are still `internal` unless they're marked `public` individually, and a public struct's memberwise initializer is always `internal`, so you must write a public one by hand. This deliberate friction makes you choose your public interface.

`package` (Swift 5.9) exposes a declaration to the other modules *of the same package*, but not to clients outside it. It serves the role of Go's `internal` directories: shared implementation details among a package's own targets. For example, a package with modules `HTTPCore`, `HTTPClient`, and `HTTPServer` can share helpers declared `package` in `HTTPCore` without making them part of the package's API.

`open` applies to classes and their members. A `public` class can be used by other modules but not subclassed by them, and a `public` method can't be overridden. `open` allows both. Since designing a class for subclassing by strangers is hard, Swift makes it opt-in.

### 10.5.1. Inlining and Library Evolution

Normally, the optimizer can't see into other modules: a call to a public function in another module is a real call, and a generic function in another module can't be specialized for the caller's types. For performance-critical code, the attribute `@inlinable` exposes a function's *body* as part of the module's interface, allowing it to be inlined and specialized across module boundaries. Most of the standard library is `@inlinable`, which is why generic algorithms like `map` and `sort` are as fast as hand-written loops.

`@inlinable` has a cost: the function body becomes part of the module's public contract, since clients compiled against one version will keep using its old body even after the library is updated. An `@inlinable` function may use only `public` declarations and `internal` ones marked `@usableFromInline`.

That concern matters most for libraries that are distributed in binary form and updated independently of their clients, like Apple's system frameworks. Such libraries are compiled with *library evolution* enabled, which makes it possible to add stored properties to structs, cases to enums, and so on, without recompiling clients, at some cost in performance. Types whose layout will never change can be marked `@frozen` to recover that performance. For source packages, which are always compiled together with their clients, library evolution is off, and none of this is a concern.

## 10.6. Modules and Naming

In this section, we'll offer some advice on how to follow Swift's distinctive conventions for naming modules and their members.

Module names are `UpperCamelCase`, like type names: `Foundation`, `ArgumentParser`, `NIOCore`. Package names, which identify repositories, are conventionally lowercase with hyphens: `swift-argument-parser`, `swift-nio`. A package's main module usually takes the package name minus the `swift-` prefix, in `UpperCamelCase`.

Because Swift imports are unqualified (after `import Foundation`, you write `URL`, not `Foundation.URL`), names in Swift must be meaningful on their own, without the module name as context. This is the opposite of Go's convention, in which a package's name is part of every reference to its members, so that `bytes.Buffer`, `http.Get`, and `json.Marshal` are short because the package name provides context. In Swift, the corresponding names are longer and self-describing: `Data`, `URLSession.data(from:)`, `JSONEncoder.encode(_:)`. A Swift module named `JSON` should not have a type called `Encoder`; it should be `JSONEncoder`, since that's how it will appear in client code.

Within a type, the opposite applies: don't repeat the type's name in its members. It's `array.count`, not `array.arrayCount`; `URL.host`, not `URL.urlHost`. Context provided by the type is always visible at the call site.

Avoid generic module names like `Utils`, `Common`, or `Helpers`, which say nothing about what's inside and tend to accumulate unrelated code. Prefer names that describe a coherent purpose, like `Links` or `TempConv`.

The API Design Guidelines, which we've referred to throughout the book, apply with special force to public APIs, which will be read by many people who didn't write them. Use argument labels to make calls read naturally. Name methods according to their side effects. Document every public declaration with a `///` comment. And strive for clarity at the point of use, which is, in the end, what all of these conventions are for.

## 10.7. The `swift` Tool

The rest of this chapter concerns the `swift` command, which is used for building, running, testing, and managing packages. It's actually a driver for a set of subcommands:

```
$ swift --help
...
SUBCOMMANDS:
  build                   Build sources into binary products
  run                     Build and run an executable product
  test                    Build and run tests
  package                 Perform operations on Swift packages
  format                  Format Swift source code
  repl                    Start an interactive session
  sdk                     Manage Swift SDKs for cross-compilation
...
```

### 10.7.1. Package Layout

To create a new package, run `swift package init` in an empty directory. Its `--type` option selects a template: `library` (the default), `executable`, `tool` (an executable using ArgumentParser), `macro`, or `build-tool-plugin`. The resulting layout follows these conventions:

```
mypackage/
    Package.swift           the manifest
    Package.resolved        the resolved versions of dependencies
    Sources/
        MyLibrary/          one directory per target
            MyLibrary.swift
        mytool/
            main.swift
    Tests/
        MyLibraryTests/
            MyLibraryTests.swift
    .build/                 build products and checked-out dependencies (don't commit)
```

Unlike Go before modules, with its single `GOPATH` workspace, every Swift package is self-contained. Dependencies are checked out inside each package's `.build/checkouts` directory, and a cache in your home directory avoids downloading the same repository twice.

### 10.7.2. Building Packages

`swift build` compiles every target in the package and its dependencies. By default, it builds in the *debug* configuration, which compiles quickly, keeps run-time assertions, and includes full debugging information. `swift build -c release` builds with optimization:

```
$ swift build
Building for debugging...
[42/42] Linking calc
Build complete! (4.21s)
$ swift build -c release
Building for production...
Build complete! (38.66s)
$ ls .build/release/calc
.build/release/calc
```

Build products go in `.build/debug` or `.build/release`. `swift run` builds and then runs an executable product, passing any further arguments to it:

```
$ swift run calc -- "sqrt(2)"
1.4142135623730951
```

`swift build --product calc` builds just one product and its dependencies, and `--target` builds one target.

To produce an executable that runs on other machines, add `--static-swift-stdlib` to link the Swift runtime statically. On Linux, the *Static Linux SDK* goes further, producing fully static executables, with no dependencies on any shared libraries, that run on any Linux distribution:

```
$ swift sdk install <URL of the static Linux SDK bundle>
$ swift build -c release --swift-sdk x86_64-swift-linux-musl
```

The same mechanism, Swift SDKs, supports cross-compilation to other platforms, such as compiling on macOS for Linux, or for WebAssembly.

### 10.7.3. Documenting Packages

Swift style strongly encourages good documentation of package APIs. Each public declaration should be immediately preceded by a documentation comment. Documentation comments start with `///` (or `/** ... */`) and are written in Markdown. The first paragraph is a summary, and subsequent paragraphs give the details. Special list items document parameters, return values, and errors:

```swift
/// Parses the input string as an arithmetic expression.
///
/// The grammar supports the four arithmetic operators, unary `+` and `-`,
/// parentheses, numeric literals, variables, and calls to `pow`, `sin`, and `sqrt`.
///
/// - Parameter input: The text of the expression.
/// - Returns: The syntax tree of the expression.
/// - Throws: ``SyntaxError`` if the input is not a valid expression.
public func parse(_ input: String) throws(SyntaxError) -> Expr
```

Double backticks, as in ``` ``SyntaxError`` ```, create a link to another symbol. Your editor shows these comments when you hover over a symbol, and *DocC*, the Swift documentation compiler, turns them into a browsable web site, with articles and tutorials written in separate Markdown files alongside the code. With the `swift-docc-plugin` package added as a dependency:

```
$ swift package generate-documentation --target Eval
$ swift package --disable-sandbox preview-documentation --target Eval
```

The Swift Package Index builds and hosts DocC documentation for every package it lists.

### 10.7.4. Formatting and Plug-ins

`swift format` formats Swift source code according to a configurable style, and `swift format lint` reports style violations without changing anything. A `.swift-format` file in the package directory configures the style; without one, the defaults are used.

SwiftPM can be extended with *plug-ins*, which are themselves Swift packages. A *command plug-in* adds a subcommand to `swift package`, like the DocC plug-in's `generate-documentation`. A *build tool plug-in* runs during the build to generate source code, for example to compile Protocol Buffers definitions into Swift or to embed version information. Plug-ins run in a sandbox, with no network access and write access only to designated directories, unless the user explicitly grants more.

### 10.7.5. Querying Packages

`swift package describe` prints a summary of the package's targets and products, and with `--type json`, produces machine-readable output suitable for tools:

```
$ swift package describe
Name: eval
Manifest display name: eval
Path: /home/user/eval
Tools version: 6.0
Dependencies:
    Url: https://github.com/apple/swift-argument-parser.git
    Version: 1.5.0..<2.0.0
Platforms:
    Name: macos
    Version: 14.0
Products:
    Name: Eval
    Type:
        Library:
            automatic
    Targets:
        Eval
...
```

`swift package dump-package` prints the evaluated manifest as JSON, and `swift package show-dependencies --format dot` produces a dependency graph that can be rendered with Graphviz.

### 10.7.6. Editing Dependencies

Sometimes you need to make a change to a dependency, to debug it or to try a fix before it's released. `swift package edit` checks out a dependency into a `Packages/` directory in your package, where you can modify it freely; the build uses your edited copy until you run `swift package unedit`. For a dependency that you're developing at the same time as your own package, a `.package(path:)` dependency, or a dependency *mirror* configured with `swift package config set-mirror`, achieves the same thing more permanently.

In this chapter, we've explained how to use the `swift` tool's most important features for packages. In the next chapter, we'll see how it's used for testing.

**Exercise 10.1:** Create a package with three targets, `Core`, `CLI`, and `Server`, in which `CLI` and `Server` both depend on `Core`. Declare a helper function in `Core` with `package` access, and confirm that it can be used from the other targets but not from a separate package that depends on yours.

**Exercise 10.2:** Write a program that reads the JSON output of `swift package show-dependencies --format json` and prints the transitive dependencies of the package, with the number of other packages that depend on each.

**Exercise 10.3:** Using `swift package describe --type json`, write a tool that reports, for each target in a package, how many source files it has and how many lines of Swift they contain.

**Exercise 10.4:** Construct a tool that reports the set of all modules that transitively depend on the modules named by its arguments, within a package's dependency graph.
