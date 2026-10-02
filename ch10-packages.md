# 10. Packages and the Swift Package Manager

Programs aren't written from scratch. Even a small command-line tool leans on the standard library for strings and collections, on Foundation for files and dates, and perhaps on a few open-source packages for argument parsing or networking. A large application may depend on dozens of packages, each with dependencies of its own. Keeping all that organized (deciding what code goes where, what each piece exposes to the others, which versions of outside code to use, and how to build it all reproducibly) is the job of a language's module and package system.

In Swift, that system is the Swift Package Manager, *SwiftPM* for short, which ships with every Swift toolchain. We've used it since Chapter 1 to build and run examples. This chapter explains how it works: how code is divided into modules, targets, and packages; how a manifest describes them; how dependencies are declared, resolved, and pinned; how access levels and imports control what each module sees of the others; and how the `swift` command builds, tests, documents, and formats a package.

## 10.1. Modules, Targets, Products, and Packages

SwiftPM organizes code at four levels, and keeping them distinct makes everything else easier to follow.

A **module** is the unit of compilation and of naming. It's a group of Swift source files compiled together; within a module, every file sees every other file's declarations without imports. Each module is also a namespace: two modules can each declare a type called `Parser` without conflict, and clients can tell them apart as `Expressions.Parser` and `Markdown.Parser`. `Foundation`, `NIOCore`, and the `Distance` library of Section 2.6 are modules.

A **target** is SwiftPM's recipe for building one module, or a test suite, or a plug-in. It records where the target's source files are, which other targets and products it depends on, and any special build settings.

A **product** is something a package makes available to the outside world: a *library*, which other packages can depend on, or an *executable*, which users can run. A product is built from one or more targets. Targets that belong to no product are private to their package.

A **package** is the unit of versioning and distribution: a directory, usually a Git repository, whose `Package.swift` manifest declares the package's targets, products, and dependencies. Packages are what get released, tagged with version numbers, and depended on.

So a package such as `swift-argument-parser` contains several targets, among them the library module `ArgumentParser`, and exposes that module as a product of the same name. Your package depends on the *package*, and your target depends on the *product*.

How large should a module be? Swift modules tend to be larger and fewer than the packages or namespaces of some other languages. A module is a natural unit of encapsulation, since its `internal` declarations are invisible outside it, and also a unit of compilation: the compiler type-checks a module as a whole and records its public interface so that dependent modules can be compiled without reparsing its source. A package with three or four modules is typical. Splitting a big module along natural seams can speed up incremental builds, because a change to one module recompiles only the modules that depend on its interface, and independent modules build in parallel.

## 10.2. Creating and Laying Out a Package

`swift package init` creates a new package in the current directory. Its `--type` option chooses a template: `library` (the default), `executable`, `tool` (an executable that uses ArgumentParser), `macro` (Section 12.8), `build-tool-plugin`, and others. A library package starts out like this:

```
mylib/
    Package.swift           the manifest
    Sources/
        MyLib/              one directory per target
            MyLib.swift
    Tests/
        MyLibTests/
            MyLibTests.swift
```

SwiftPM relies on *conventions* to keep manifests short. A target named `X` finds its sources in `Sources/X`, and a test target in `Tests/X`; everything with a `.swift` extension in that directory, including subdirectories, belongs to the target. A target can override the location with `path:`, but sticking to the convention makes packages instantly familiar to other Swift programmers.

Building creates two more things: a `.build` directory for build products and downloaded dependencies, which shouldn't be committed to source control, and, if the package has dependencies, a `Package.resolved` file recording their exact versions, which often should be (Section 10.4).

```
$ swift build
Building for debugging...
Build complete!
$ swift test
...
```

Each package is self-contained. Its dependencies are checked out under its own `.build` directory, and a shared cache in your home directory avoids downloading the same repository twice. There's no global workspace to configure.

## 10.3. The Package Manifest

The manifest, `Package.swift`, is a Swift program. It imports the `PackageDescription` module and creates a value of type `Package` describing the package. Here's a manifest for a package with a library, a command-line tool that uses it, and a test suite:

```swift
// swift-tools-version: 6.4
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

The comment on the first line isn't optional decoration. It declares the *tools version*: the oldest SwiftPM that can read this manifest, which also determines which manifest APIs are available and which defaults apply. With tools version 6.0 or later, every target compiles in the Swift 6 language mode unless it says otherwise.

`platforms` sets the minimum operating-system versions on Apple platforms (it's ignored on Linux and Windows). It matters because many APIs, `Mutex` and recent Foundation features among them, exist only on recent OS releases, and the compiler checks every use against these minimums.

`products` lists what other packages may use. `dependencies` lists the packages this one uses; Section 10.4 covers them in detail. `targets` lists the modules to build and how they depend on each other: a target depends on another target in the same package by name, as `calc` depends on `"Eval"`, and on a product from another package with `.product(name:package:)`.

Since the manifest is Swift, it can use variables, functions, and conditions. Resist the temptation to get clever, though. Tools such as editors and the Swift Package Index evaluate manifests constantly, and a manifest that reads like a plain declaration is easier for people and tools alike.

### 10.3.1. Kinds of Targets

Besides ordinary library targets, SwiftPM supports several specialized kinds:

```
.target                 a library module
.executableTarget       a module that builds a program (has main.swift or an @main type)
.testTarget             a test suite, run by swift test (Chapter 11)
.macro                  a compiler plug-in that implements macros (Section 12.8)
.plugin                 a SwiftPM plug-in: a build tool or a custom command (Section 10.7)
.systemLibrary          a module map for a C library installed on the system (Section 13.4)
.binaryTarget           a prebuilt framework or artifact bundle
```

A library target may also contain C, C++, or Objective-C instead of Swift. Its public headers go in an `include` directory, and Swift targets that depend on it can import it like any other module.

A target can carry *resources*, such as images, data files, or localized strings, declared with `resources: [.process("Resources")]` and found at run time through the generated `Bundle.module`.

### 10.3.2. Build Settings

Targets can adjust how they're compiled:

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

`.swiftLanguageMode` lets a target stay in an older language mode while it's migrated, as Section 9.6 described. `.enableUpcomingFeature` opts in early to behavior planned for a future language mode. `.define` sets a *compilation condition* that code can test with `#if ENABLE_TRACING`, here only in debug builds. `.unsafeFlags` passes arbitrary compiler flags; because such flags could do anything, SwiftPM refuses to build a package that uses them when it's someone else's dependency.

### 10.3.3. Platform-Specific Code

Code that must differ from one platform to another is written with *conditional compilation*, choosing among alternatives at compile time:

```swift
#if os(Linux)
import Glibc
#elseif os(Windows)
import WinSDK
#elseif canImport(Darwin)
import Darwin
#endif

#if arch(arm64)
// ...a path optimized for 64-bit ARM...
#endif

#if DEBUG
print("debug build")
#endif
```

The available conditions include `os()`, `arch()`, `canImport()`, `compiler(>=6.0)`, `swift(>=6.0)`, `targetEnvironment(simulator)`, and `hasFeature()`, plus any names defined with `.define`. Code in an inactive branch is parsed but not type-checked, so it may refer to APIs that don't exist on the current platform. Prefer `canImport` to `os` where possible: it asks about the capability you actually need rather than assuming which platforms have it.

Dependencies can be conditional too, so that a target links a library only where it's needed:

```swift
.target(
    name: "Net",
    dependencies: [
        .product(name: "NIOTransportServices", package: "swift-nio-transport-services",
                 condition: .when(platforms: [.macOS, .iOS])),
    ]
)
```

## 10.4. Depending on Other Packages

A dependency names a package by its location, most often a Git repository URL, and states which versions are acceptable:

```swift
.package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.0"),
```

The URL is the package's *identity*. Within one build, each identity appears once, however many packages depend on it. The last component of the URL, minus `.git`, is the name a target uses to refer to the package, as in `.product(name: "ArgumentParser", package: "swift-argument-parser")`. Organizations can also run a *package registry*, which identifies packages by scope and name (such as `apple.swift-argument-parser`) and serves release archives instead of Git repositories; public open-source packages are almost always referenced by URL.

### 10.4.1. Version Requirements

Packages are versioned with *semantic versioning*, `MAJOR.MINOR.PATCH`, marked by Git tags such as `1.5.0`. The convention is that a new major version may break clients, a new minor version adds features compatibly, and a new patch version fixes bugs compatibly. A dependency's *requirement* is the range of versions you accept:

```swift
.package(url: "...", from: "1.5.0")  // 1.5.0 ..< 2.0.0 (the usual choice)
.package(url: "...", .upToNextMinor(from: "1.5.0"))  // 1.5.0 ..< 1.6.0
.package(url: "...", exact: "1.5.2")  // exactly 1.5.2 (avoid: it blocks bug fixes)
.package(url: "...", "1.5.0"..<"1.8.0")  // an explicit range
.package(url: "...", branch: "main")  // the latest commit on a branch (development only)
.package(url: "...", revision: "a1b2c3d")  // one particular commit
.package(path: "../distance")  // a package in a local directory
```

`from:` is the right default for most dependencies: it trusts the package to follow semantic versioning, accepting fixes and features while ruling out the next major version. Branch and revision dependencies are useful while two packages are developed together, but a package that depends on one can't itself be released as a versioned dependency of others.

### 10.4.2. Resolution and `Package.resolved`

Before building, SwiftPM *resolves* the full graph of dependencies, choosing one version of each package that satisfies the requirements of every package that depends on it, preferring the newest such version. If no single version works (say, your two dependencies require `2.x` and `3.x` of a third package), resolution fails, and SwiftPM explains which requirements conflict. A graph can contain only one version of each package, so the authors of widely used packages have a strong incentive to avoid major-version churn.

The outcome is recorded in `Package.resolved`, which lists the exact version and commit chosen for every package in the graph. Later builds use exactly those versions, so a build is reproducible, until you explicitly ask for newer ones:

```
$ swift package resolve          # fetch what Package.resolved specifies
$ swift package update           # choose the newest allowed versions and rewrite Package.resolved
$ swift package update swift-nio # update only one package
$ swift package show-dependencies
.
├── swift-nio<https://github.com/apple/swift-nio.git@2.86.0>
│   ├── swift-collections<https://github.com/apple/swift-collections.git@1.2.1>
│   ├── swift-atomics<https://github.com/apple/swift-atomics.git@1.3.0>
│   └── swift-system<https://github.com/apple/swift-system.git@1.6.1>
└── swift-argument-parser<https://github.com/apple/swift-argument-parser.git@1.6.1>
```

(Your versions will differ.) Applications and tools should commit `Package.resolved`, so that every developer and every CI run builds against the same code. Libraries often don't, because their clients resolve the whole graph themselves and ignore a dependency's `Package.resolved` anyway.

### 10.4.3. Working on a Dependency

Sometimes you need to change a dependency, to debug it or try a fix before it's released. `swift package edit swift-nio` checks out an editable copy in a `Packages` directory inside your package, and builds use that copy until `swift package unedit swift-nio` restores the released version. For longer-term arrangements, a `.package(path:)` dependency, or a *mirror* configured with `swift package config set-mirror`, points SwiftPM at a different copy of a package without changing its identity.

## 10.5. Access Levels Across Modules

Section 6.6 introduced Swift's access levels. Several of them exist specifically to manage the boundaries between modules.

`internal`, the default, keeps a declaration within its module. Most declarations should stay `internal`; a module's public surface should be a deliberate choice, not an accident.

`public` makes a declaration available to other modules. Making a type public doesn't make its members public: each property, method, and initializer that clients should see must be marked too. In particular, the memberwise initializer that the compiler writes for a struct is always `internal`, so a public struct that clients should be able to create needs a public initializer written out, as `Kilometers` had in Section 2.6.

`package` (Swift 5.9) makes a declaration visible to every module in the same package, but not to other packages. It's how a package's modules share helpers without making them part of the package's public interface. For example, a package with modules `HTTPCore`, `HTTPClient`, and `HTTPServer` can keep its shared parsing code `package`-visible in `HTTPCore`.

`open` applies to classes and their members. A `public` class can be used, but not subclassed, outside its module, and its `public` methods can't be overridden there. `open` permits both. Designing a class to be safely subclassed by strangers is hard, so Swift makes it an explicit decision.

### 10.5.1. Inlining and Library Evolution

Ordinarily, the optimizer can't see inside other modules. A call to a public function in another module is a real call, and a generic function from another module can't be specialized for the caller's types. For performance-critical code, `@inlinable` publishes a function's *body* as part of its module's interface, so that it can be inlined and specialized in client modules. The standard library marks most of its generic algorithms `@inlinable`, which is how `map` and `sort` can be as fast as hand-written loops.

The cost is commitment: an inlinable body becomes part of the module's public contract, since clients compiled against one version keep using its old body even after the module changes. An `@inlinable` function may refer only to `public` declarations and to `internal` ones marked `@usableFromInline`.

This matters most for libraries shipped as binaries and updated separately from the programs that use them, such as Apple's system frameworks. Those are built with *library evolution* enabled, which lets them add stored properties to structs and cases to enums without breaking already-compiled clients, at some cost in performance; types whose layout will never change can be marked `@frozen` to recover it. Ordinary source packages, which are always compiled together with their clients, leave library evolution off, and none of this applies.

## 10.6. Imports and Module Interfaces

A source file uses another module by importing it. Imports come before other declarations, and each names a module:

```swift
import Foundation
import NIOCore
import Distance
```

An import makes the module's public declarations available throughout the file, without qualification. It's an error to import a module that the file's target doesn't depend on in the manifest. And imports are per file: importing `Foundation` in one file doesn't make it available in the module's other files.

If two imported modules declare the same name, uses of that name must be qualified with the module, as in `Foundation.Data` or `SystemPackage.FilePath`. Swift can't rename a module on import, but collisions are uncommon in practice, because Swift names are written to stand on their own (Section 10.6.2).

### 10.6.1. Kinds of Import

Several variations on `import` are worth knowing.

*Access-level imports* (Swift 6) say whether a dependency is part of the module's public interface. A plain `import` is effectively `public`: the module's public declarations may mention the imported module's types. Writing `internal import` (or `private import`) declares the dependency an implementation detail, and the compiler checks that none of its types leak into public declarations:

```swift
internal import SwiftSoup  // used inside the module, but never in its public API
public import Foundation  // our public API mentions URL and Date
```

That keeps an implementation dependency replaceable without breaking clients, and lets the build system skip recompiling clients when it changes. The upcoming feature `InternalImportsByDefault` makes `internal` the default for every import that doesn't say otherwise.

*Scoped imports* bring in a single declaration, as in `import struct Foundation.URL`. They document a narrow dependency, but they still make the module's extensions and operators visible, so they're less precise than they look.

`@testable import` gives a test target access to a module's `internal` declarations (Section 11.1). It works only for modules built with testing enabled, as debug builds are.

`@preconcurrency import` suppresses concurrency diagnostics about types from a module that hasn't yet adopted `Sendable` annotations, a tool for migrating to Swift 6.

Importing a module never runs code. Swift has no module initializers: as Section 2.6.2 explained, global variables are initialized lazily on first use, so nothing happens merely because a module is imported or linked. A library that needs callers to register plug-ins, decoders, or drivers asks for that explicitly, typically by taking a list of types or values that conform to a protocol:

```swift
let image = try decodeImage(data, using: [PNGDecoder.self, JPEGDecoder.self])
```

That keeps a program's capabilities visible in its source rather than depending on which modules happen to be linked.

### 10.6.2. Naming within Modules

Module names use `UpperCamelCase`, like types: `Foundation`, `ArgumentParser`, `NIOCore`. Package names, which name repositories, are conventionally lowercase with hyphens, such as `swift-argument-parser`, and a package's main module usually takes its name without the `swift-` prefix.

Because imports are unqualified, a Swift name appears in client code *without* its module as context: after `import Foundation`, code writes `URL`, not `Foundation.URL`. So names must make sense on their own. A JSON library's encoder should be called `JSONEncoder`, not just `Encoder`, since `Encoder` alone is what clients would see. Within a type, on the other hand, the type provides the context, so members shouldn't repeat it: `array.count`, not `array.arrayCount`; `url.host`, not `url.urlHost`.

Choose module names that say what the module is for, like `Links` or `Distance`. Names such as `Utilities`, `Common`, or `Helpers` say nothing, and modules with such names tend to collect unrelated code until nobody can say what they're for.

For a module's public API, the API Design Guidelines (Section 2.1) apply with extra force, since its users won't have its source in front of them. Choose argument labels that make calls read naturally, name methods by their effects, document every public declaration (Section 10.7.3), and aim above all for clarity at the point of use.

## 10.7. The `swift` Tool

A single command, `swift`, runs most of the toolchain:

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

### 10.7.1. Building and Running

`swift build` builds every target in the package and its dependencies. By default it uses the *debug* configuration, which compiles quickly, keeps all run-time checks and assertions, and includes full debugging information. `-c release` builds with optimization, which can make compute-heavy programs many times faster:

```
$ swift build
Building for debugging...
[42/42] Linking calc
Build complete! (4.21s)
$ swift build -c release
Building for production...
Build complete! (38.66s)
```

Products land in `.build/debug` or `.build/release`. `swift run` builds an executable product and runs it, passing along any arguments after the product name; `swift build --product calc` or `--target Eval` builds just part of the package:

```
$ swift run calc
sqrt(2)
1.414213562
```

### 10.7.2. Building for Distribution

An executable built normally depends on the Swift runtime libraries being installed on the machine that runs it. `--static-swift-stdlib` links them into the executable instead. On Linux, the *Static Linux SDK* goes further and produces fully static executables, depending on no shared libraries at all, that run on any distribution:

```
$ swift sdk install <URL of the static Linux SDK bundle>
$ swift build -c release --swift-sdk x86_64-swift-linux-musl
```

The same *Swift SDK* mechanism supports cross-compilation, such as building Linux or WebAssembly binaries on a Mac.

### 10.7.3. Documenting Packages

A package's public API should be documented, and Swift's conventions make that easy to do well. A *documentation comment*, written with `///` (or `/** ... */`), goes immediately before the declaration it describes. It's written in Markdown. Its first paragraph is a one-sentence summary, which editors show in completion lists; later paragraphs give details; and special list items describe parameters, results, and errors:

```swift
/// Parses the input string as an arithmetic expression.
///
/// The grammar supports the four arithmetic operators, unary `+` and `-`,
/// parentheses, numeric literals, variables, and calls to `min`, `max`,
/// `abs`, and `sqrt`.
///
/// - Parameter input: The text of the expression.
/// - Returns: The syntax tree of the expression.
/// - Throws: ``SyntaxError`` if the input is not a valid expression.
public func parse(_ input: String) throws(SyntaxError) -> Expr
```

A name in double backticks, like ``` ``SyntaxError`` ```, becomes a link to that symbol's documentation. Editors display these comments when you hover over or complete a name, and *DocC*, the documentation compiler, turns them into a browsable site, which can also include articles and step-by-step tutorials written in Markdown alongside the code. With the `swift-docc-plugin` package added as a dependency:

```
$ swift package generate-documentation --target Eval
$ swift package --disable-sandbox preview-documentation --target Eval
```

The Swift Package Index builds and hosts DocC documentation for the packages it lists.

### 10.7.4. Formatting and Plug-ins

`swift format` reformats source files into a consistent style, and `swift format lint` reports deviations without changing anything, which suits CI. A `.swift-format` file in the package configures the style.

SwiftPM can be extended with *plug-ins*, which are themselves packages. A *command plug-in* adds a subcommand to `swift package`, as the DocC plug-in adds `generate-documentation`. A *build-tool plug-in* runs during builds to generate code, for instance compiling Protocol Buffers definitions to Swift, or embedding a version number. Plug-ins run in a sandbox, without network access and with write access only to designated directories, unless the user grants more.

### 10.7.5. Inspecting Packages

Several commands report on a package's structure, which is useful for tools and scripts as well as people. `swift package describe` summarizes targets, products, and dependencies, and `--type json` produces the same information as JSON:

```
$ swift package describe
Name: eval
Manifest display name: eval
Path: /home/user/eval
Tools version: 6.4
Dependencies:
    Type:
        sourceControl
    Identity:
        swift-argument-parser
    Url: https://github.com/apple/swift-argument-parser.git
    Requirement:
        Range:
            Lower bound:
                1.5.0
            Upper bound:
                2.0.0

Platforms:
    Name: macos
    Version: 14.0

    Name: ios
    Version: 17.0

Products:
    Name: Eval
    Type:
        Library:
            automatic
    Targets:
        Eval
...
```

`swift package dump-package` prints the fully evaluated manifest as JSON, and `swift package show-dependencies --format dot` writes the dependency graph in a form that Graphviz can draw.

The next chapter turns to one more subcommand, `swift test`.

**Exercise 10.1:** Create a package with three targets, `Core`, `CLI`, and `Server`, where `CLI` and `Server` depend on `Core`. Give `Core` a helper function with `package` access, and confirm that both other targets can use it but that a separate package depending on yours can't.

**Exercise 10.2:** Write a tool that reads the output of `swift package show-dependencies --format json` and lists every package in the graph along with how many other packages depend on it.

**Exercise 10.3:** Using `swift package describe --type json`, write a tool that reports how many Swift source files and lines of code each target in a package contains.

**Exercise 10.4:** Change one of the book's library modules to use `internal import` for every dependency that doesn't appear in its public API. Which imports have to stay public, and why?
