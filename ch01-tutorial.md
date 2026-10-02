# 1. Tutorial

The quickest way into a programming language is to read and run small programs that do something useful, and that's what this chapter offers: a quick tour of Swift through a dozen or so short programs. They read command-line arguments, count things in files, draw a picture, download web pages, time network requests concurrently, and run a small web server. Each program introduces a few features of the language, and each feature is explained only as far as the program needs. The chapters that follow fill in the details.

If you already know another language, you'll find yourself mapping Swift onto it as you read. That's natural, and mostly helpful, but watch for the places where Swift deliberately departs from what you know. Optionals, value semantics, argument labels, and the separation between `let` and `var` are the features most often written un-idiomatically by newcomers, and the examples in this book are meant to show the idiomatic way.

## 1.1. Hello, World

Tradition demands that the first program print a greeting:

```swift
// swiftpl/ch1/hello
print("¡Hola, mundo! 👋")
```

That's a complete program. Save it as `hello.swift` and run it:

```
$ swift hello.swift
¡Hola, mundo! 👋
```

Swift strings are Unicode throughout, so accented letters and emoji need no special treatment.

Swift is a compiled language: its toolchain translates source code into native machine code before the program runs. The command above compiles the file and runs the result in one step, which is convenient for experiments. To produce an executable you can keep and run later, use the compiler directly:

```
$ swiftc hello.swift
$ ./hello
¡Hola, mundo! 👋
```

Programs of more than one file, and programs that use libraries, are built with the Swift Package Manager, covered fully in Chapter 10. To create a new executable package:

```
$ mkdir hello && cd hello
$ swift package init --type executable
```

This creates a *manifest*, `Package.swift`, describing what to build, and a source file, `Sources/hello/main.swift`:

```
hello/
    Package.swift
    Sources/
        hello/
            main.swift
```

Put the program in `main.swift`, then build and run it in one command:

```
$ swift run
Building for debugging...
Build complete!
¡Hola, mundo! 👋
```

Each example in this book carries a label like `swiftpl/ch1/hello` that names the package it belongs in.

A few things about this tiny program are worth noticing. Swift code lives in *modules*. Every file in a module can see every declaration in the module's other files, with no headers and no imports between them. The standard library, module `Swift`, is imported into every file automatically; it supplies basic types like `Int`, `String`, and `Array`, and functions like `print`. Other modules must be imported, most often `Foundation`, which provides files, dates, URLs, networking, and formatting on every platform Swift supports.

A file named `main.swift` is special. The statements in it are *top-level code*, executed in order when the program starts, which is why the greeting needs no `main` function. In any other file, only declarations are allowed. Larger programs often prefer to mark a type with `@main` instead, giving it a static `main` method:

```swift
@main
struct Hello {
    static func main() {
        print("¡Hola, mundo! 👋")
    }
}
```

Statements end at the end of a line; semicolons are needed only to put two statements on one line, which is rare. Swift doesn't impose a layout on your code, but the toolchain includes `swift format`, which rewrites code into a consistent style, and the examples in this book follow its conventions, except that they indent by four spaces rather than its default of two (a `.swift-format` configuration file sets that). Editors that use Swift's language server, SourceKit-LSP, can format on every save.

## 1.2. Command-Line Arguments

A program needs input, and the simplest source of input is its command line. The standard library provides the arguments in `CommandLine.arguments`, an *array* of strings (written `[String]`), in which element 0 is the name the program was invoked by and the rest are the arguments that followed it.

Our first program adds up the numbers given on its command line:

```swift
// swiftpl/ch1/sum1
// Sum1 prints the total of its numeric command-line arguments.

let args = CommandLine.arguments
var total = 0.0
var i = 1
while i < args.count {
    if let x = Double(args[i]) {
        total += x
    } else {
        print("ignoring \(args[i]): not a number")
    }
    i += 1
}
print(total)
```

```
$ swift run sum1 12 30 0.5 lots
ignoring lots: not a number
42.5
```

Lines that begin with `//` are comments. By convention, each program begins with a comment saying what it does.

The program introduces Swift's two kinds of names for values. A `let` declares a *constant*, which keeps its first value forever. A `var` declares a *variable*, which can be assigned new values. Swift encourages `let` wherever possible: the compiler warns about any `var` that's never changed, and code is easier to follow when most names can't change underneath you.

No declaration states a type, because the compiler *infers* types from initial values. `args` is a `[String]`, `total` a `Double` because its initial value `0.0` is a floating-point literal, and `i` an `Int`. Swift is strictly typed even so: every name has exactly one type, fixed at compile time, and you can always write it out, as in `var total: Double = 0`.

`Double(args[i])` tries to convert a string to a number. Not every string is a number, so the conversion returns an *optional*, written `Double?`, which holds either a number or `nil`, meaning no value. Swift never lets you use an optional as if it were the value inside; you have to check first. The `if let` statement does both at once: if the conversion produced a number, it's bound to the new constant `x` and the first block runs; otherwise the `else` block runs. We'll see much more of optionals, which are how Swift eliminates null-pointer errors, in the next section.

The `while` loop repeats while its condition is true. Conditions need no parentheses, but the body always needs braces, and the condition must be a `Bool`: Swift never treats numbers or pointers as true or false. The statement `i += 1` increments `i`; Swift has no `++` operator, because the explicit form is clearer.

String literals can contain *interpolations*: `\(` *expression* `)` inside a string is replaced by the expression's value, as in `"ignoring \(args[i]): not a number"`. It works with values of any type.

Index arithmetic is clumsy and error-prone, and Swift usually avoids it with a `for`-`in` loop, which visits each element of a *sequence* in turn:

```swift
// swiftpl/ch1/sum2
// Sum2 prints the total of its numeric command-line arguments.

var total = 0.0
for arg in CommandLine.arguments.dropFirst() {
    if let x = Double(arg) {
        total += x
    }
}
print(total)
```

`dropFirst()` skips the program name. It returns an `ArraySlice`, a view of part of the array that copies nothing (Section 4.2). The loop variable `arg` is a new constant in each iteration.

`for`-`in` also iterates over *ranges* of integers: `0..<n` is the half-open range from 0 up to but not including `n`, and `1...n` the closed range including `n`. There's no C-style three-part `for` loop.

Finally, a version that leans on the standard library. `compactMap` converts every argument with `Double(_:)` and keeps only the successes, and `reduce` combines the results with `+`:

```swift
// swiftpl/ch1/sum3
let numbers = CommandLine.arguments.dropFirst().compactMap { Double($0) }
print(numbers.reduce(0, +))
```

The braces after `compactMap` are a *closure*, an anonymous function, and `$0` is its argument. Chapter 5 covers closures in detail. For quick debugging, `print` can show any value directly: `print(numbers)` displays an array in brackets, like `[12.0, 30.0, 0.5]`.

**Exercise 1.1:** Modify `sum2` to print the count and the average of the numbers as well as the total.

**Exercise 1.2:** Print each argument on its own line along with its position and whether it's a number. (Hint: look up `enumerated()`.)

**Exercise 1.3:** Use the `min()` and `max()` methods of arrays to make `sum3` also print the smallest and largest numbers. What do they return when there are no numbers?

## 1.3. Counting Things

A great many useful programs follow the same pattern: read some input, examine each piece, and accumulate counts or totals. Our next program applies it to log files. Suppose each line of a log looks like this:

```
2025-09-30T09:12:44Z INFO server started on port 8080
2025-09-30T09:13:02Z WARN slow request: 2.4s /reports
2025-09-30T09:13:05Z ERROR database connection lost
```

The program counts how many lines of each level (`INFO`, `WARN`, `ERROR`, and so on) appear in its standard input:

```swift
// swiftpl/ch1/levels1
// Levels1 counts the log levels in lines read from standard input.

var counts: [String: Int] = [:]
while let line = readLine() {
    let fields = line.split(separator: " ")
    if fields.count >= 2 {
        counts[String(fields[1]), default: 0] += 1
    }
}
for (level, n) in counts {
    print("\(level)\t\(n)")
}
```

```
$ swift run levels1 < server.log
INFO	1902
ERROR	14
WARN	87
```

`counts` is a *dictionary*, a table that maps keys to values, here strings to integers. Its type, `[String: Int]`, is shorthand for `Dictionary<String, Int>`, and `[:]` is an empty dictionary. Looking up a key in a dictionary is fast no matter how many entries it has.

Looking up a key that isn't there gives `nil`, so a dictionary subscript returns an optional: `counts["DEBUG"]` has type `Int?`. To count, though, we want a missing key to act like zero, and the `default:` form of the subscript says exactly that. `counts[key, default: 0] += 1` reads the current count, or 0 if there is none, adds one, and stores it back.

`readLine()` reads one line from the standard input, without its newline. At the end of the input there's no line to return, so its result is an optional too, `String?`, and it returns `nil`. The loop `while let line = readLine()` keeps reading and binding each line to `line` until that happens. This *optional binding* appears constantly in Swift; `if let` and `guard let` (Section 1.5) use the same idea.

`split(separator:)` breaks the line into fields at each space. The pieces are `Substring`s, views into the original line, so we convert the level to a `String` before using it as a key; otherwise each key would keep its entire line alive.

The second loop iterates over the dictionary, binding each entry's key and value to `level` and `n`. The order is unspecified, and in fact changes from run to run: Swift randomizes dictionary order on purpose, so that no program comes to depend on an accident. When order matters, sort first, as the next version does.

The next version reads the files named on its command line, or the standard input if there are none, and lists the levels from most to least frequent:

```swift
// swiftpl/ch1/levels2
// Levels2 counts the log levels in the named files, or standard input.
import Foundation

var counts: [String: Int] = [:]
let paths = CommandLine.arguments.dropFirst()
if paths.isEmpty {
    while let line = readLine() {
        countLevel(in: line, into: &counts)
    }
} else {
    for path in paths {
        do {
            let text = try String(contentsOfFile: path, encoding: .utf8)
            for line in text.split(separator: "\n") {
                countLevel(in: line, into: &counts)
            }
        } catch {
            printError("levels2: \(path): \(error.localizedDescription)")
        }
    }
}
for (level, n) in counts.sorted(by: { $0.value > $1.value }) {
    print("\(level)\t\(n)")
}

func countLevel(in line: some StringProtocol, into counts: inout [String: Int]) {
    let fields = line.split(separator: " ")
    if fields.count >= 2 {
        counts[String(fields[1]), default: 0] += 1
    }
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

Reading a file can fail in several ways, such as a missing file, no permission, or invalid UTF-8, and `String(contentsOfFile:encoding:)` reports failure by *throwing an error*. Swift requires a `try` before every call that can throw, so that the possible failure points are visible, and the call must be inside a `do` block with a `catch` clause or inside a function that itself throws. If the read fails, control jumps to `catch`, where the error is available as `error`. We report it on the standard error stream and go on to the next file. `print` writes only to the standard output, so the small `printError` function does the job with Foundation's `FileHandle`. Chapter 5 discusses error handling in depth.

`countLevel` must add to the caller's dictionary. Function parameters are constants, and dictionaries, like all Swift collections, are *values*: a function receiving one normally gets its own logical copy. Declaring the parameter `inout` lets the function modify the caller's variable instead, and the call marks this with `&counts`, so the reader can see that `counts` may change. The parameter type `some StringProtocol` accepts any string-like value, so the function works both with the `String`s from `readLine` and with the `Substring`s produced by splitting a file.

Functions declared in `main.swift` can be used anywhere in the file, even above their declarations; only top-level variables have to be declared before use.

The output is sorted with `sorted(by:)`, which takes a closure saying when one element should precede another. Here each element is a key-value pair, and `$0.value > $1.value` puts larger counts first.

Reading a whole file into memory and then splitting it is simple and fast for files of reasonable size. For very large logs, a streaming approach is better; Chapter 8 introduces asynchronous sequences, which make one easy.

**Exercise 1.4:** Modify `levels2` to also print, for each level, the timestamp of the first and last line that had it.

**Exercise 1.5:** Add an option that prints the names of the files containing any `ERROR` lines, and how many each has.

## 1.4. Drawing a Spirograph

Programs can produce pictures as well as text. This one draws a *hypotrochoid*, the looping curve made by a pen fixed to a small wheel rolling around inside a ring, familiar from the Spirograph toy. The output is SVG, a text-based image format that every web browser can display.

```swift
// swiftpl/ch1/spirograph
// Spirograph draws a random hypotrochoid as an SVG image.
import Foundation

let size = 300.0  // the image is size × size pixels
let steps = 10_000  // points along the curve
let turns = 40.0  // revolutions to draw; enough for most curves to close
let colors = ["crimson", "darkorange", "seagreen", "royalblue", "purple"]

func spirograph() -> String {
    // The ring has radius 1; choose the wheel's radius and the pen's offset at random.
    let r = Double.random(in: 0.2...0.8)
    let d = Double.random(in: 0.3...0.9)
    let k = (1 - r) / r
    let scale = size / 2 / (1 - r + d)  // fit the curve's maximum extent to the image

    var points: [String] = []
    for i in 0...steps {
        let t = Double(i) / Double(steps) * turns * 2 * .pi
        let x = (1 - r) * cos(t) + d * cos(k * t)
        let y = (1 - r) * sin(t) - d * sin(k * t)
        points.append(String(format: "%.1f,%.1f", size / 2 + x * scale, size / 2 + y * scale))
    }
    let color = colors.randomElement()!
    return """
        <svg xmlns="http://www.w3.org/2000/svg" width="\(Int(size))" height="\(Int(size))">
        <polyline fill="none" stroke="\(color)" stroke-width="0.6"
            points="\(points.joined(separator: " "))"/>
        </svg>
        """
}

print(spirograph())
```

```
$ swift run spirograph > curve.svg
```

Open `curve.svg` in a browser to see the result; run the program again for a different curve.

The constants at the top have types inferred from their literals: `size` and `turns` are `Double`s, and `steps` is an `Int` (the underscore in `10_000` just groups digits). Swift never converts between number types on its own, so wherever an `Int` meets a `Double`, as in `Double(i) / Double(steps)`, the conversion is written out. That can feel fussy at first, but it rules out a family of silent truncation and rounding bugs; Section 3.1 has the details.

The math follows the definition of the curve: as the angle `t` advances, the wheel's center circles the ring, the wheel itself turns `k` times as fast in the opposite direction, and the pen, a distance `d` from the wheel's center, traces the sum of the two motions. `.pi` is short for `Double.pi`; since the type is known from context, the leading dot is enough. `sin` and `cos` come from the C math library, which `Foundation` makes available.

`Double.random(in:)` picks a random number from a range, and `randomElement()` picks a random element of an array. The latter returns an optional, `nil` for an empty array; we know the array isn't empty, so the `!` operator *force-unwraps* it, asserting that the value is present. If the assertion were wrong, the program would stop with an error, so `!` should be used only when you're certain.

Each point is formatted with `String(format:)`, Foundation's version of C's `printf`, which here rounds each coordinate to one decimal place. The SVG text is built with a *multiline string literal*, delimited by `"""` on lines of their own. The indentation of the closing delimiter is removed from every line, so the literal can be indented to match the surrounding code. Section 3.5 has more about string literals.

**Exercise 1.6:** Draw two curves with different random parameters and colors in the same image.

**Exercise 1.7:** Accept `r` and `d` as optional command-line arguments, choosing them at random only when they're absent.

## 1.5. Downloading a URL

Programs today talk to the network as routinely as they read files. Foundation's `URLSession` makes HTTP requests. The program below downloads each URL named on its command line and writes the response body to the standard output, the essence of tools like `curl`. It also reports the HTTP status and the size of each response on the standard error stream, so that the report doesn't get mixed into the downloaded data when the output is redirected to a file or piped into another program.

```swift
// swiftpl/ch1/download
// Download writes the content found at each URL to standard output.
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

for arg in CommandLine.arguments.dropFirst() {
    guard let url = URL(string: arg) else {
        printError("download: invalid URL: \(arg)")
        exit(1)
    }
    do {
        let (data, response) = try await URLSession.shared.data(from: url)
        FileHandle.standardOutput.write(data)
        if let http = response as? HTTPURLResponse {
            printError("\(arg): status \(http.statusCode), \(data.count) bytes")
        }
    } catch {
        printError("download: \(arg): \(error.localizedDescription)")
        exit(1)
    }
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

```
$ swift build
$ .build/debug/download https://www.swift.org > page.html
https://www.swift.org: status 200, 21342 bytes
```

Later examples use `download` as an ordinary command, running it as `download` rather than `.build/debug/download`, often in pipelines with other programs. To make that work, build an optimized copy and put it in a directory on your shell's `PATH`:

```
$ swift build -c release
$ mkdir -p ~/bin && cp .build/release/download ~/bin/   # ~/bin must be on your PATH
```

(SwiftPM can also do the copying: `swift package experimental-install` installs a package's executables in `~/.swiftpm/bin`, which you can add to your `PATH`.)

On Apple platforms, `URLSession` is part of Foundation. On Linux and Windows it lives in a separate module, `FoundationNetworking`, so that programs that don't use the network don't pay for it. The `#if canImport(...)` directive is *conditional compilation*: the `import` it encloses is compiled only where the module exists. You'll see this at the top of most portable Swift networking code.

`guard` is a statement for checking preconditions. It requires its condition to be true for execution to continue past it; otherwise the `else` block runs, and that block must leave the current scope, here by calling `exit` to end the program. With `let`, as here, `guard` unwraps an optional and makes the value available for the rest of the scope. `URL(string:)` returns `nil` for text that isn't a valid URL, so `url` has type `URL?` until the guard unwraps it to a `URL`.

A network request may take a long time, and while it's pending the program could be doing other work. So `data(from:)` is *asynchronous*: it's declared `async`, and calls to it are marked `await`. At an `await`, the current task may be suspended until the result arrives, freeing its thread to run other work in the meantime. Top-level code in `main.swift` can use `await` directly. The call also may throw, so it's written `try await`.

The result is a *tuple* of two values: the response body as a `Data` value, a buffer of bytes, and a `URLResponse` describing the response, which we unpack into `data` and `response` in one step. For HTTP requests, the response is really an `HTTPURLResponse`, a subclass with a status code; `as?` tries the conversion and gives `nil` if it doesn't apply, so `if let http = response as? HTTPURLResponse` both checks the type and unwraps the result (Section 7.10).

**Exercise 1.8:** Add an `-o` option that saves each download to a file named after the last component of its URL's path, instead of writing it to the standard output.

**Exercise 1.9:** Check the `Content-Type` header (`http.value(forHTTPHeaderField:)`) and refuse to write anything that isn't text to a terminal unless a `--force` option is given.

## 1.6. Timing Requests Concurrently

The real strength of asynchronous code is doing several things at once. The next program, `timeall`, requests each URL on its command line *concurrently*, measuring how long each one takes and how big it is, and when all have finished, prints them ranked from fastest to slowest:

```swift
// swiftpl/ch1/timeall
// Timeall requests URLs concurrently and ranks them by response time.
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

struct Timing: Sendable {
    var url: String
    var elapsed: Duration
    var bytes: Int?  // nil if the request failed
}

let start = ContinuousClock.now
var timings: [Timing] = []
await withTaskGroup(of: Timing.self) { group in
    for url in CommandLine.arguments.dropFirst() {
        group.addTask {
            await measure(url)  // each request runs as a separate child task
        }
    }
    for await timing in group {
        timings.append(timing)  // collect results as they finish
    }
}

for (rank, t) in timings.sorted(by: { $0.elapsed < $1.elapsed }).enumerated() {
    let size = t.bytes.map { "\($0) bytes" } ?? "failed"
    print("\(rank + 1). \(t.url)  \(t.elapsed)  \(size)")
}
print("total: \(ContinuousClock.now - start)")

func measure(_ url: String) async -> Timing {
    let start = ContinuousClock.now
    var bytes: Int? = nil
    if let u = URL(string: url), let result = try? await URLSession.shared.data(from: u) {
        bytes = result.0.count  // result is a (Data, URLResponse) tuple
    }
    return Timing(url: url, elapsed: ContinuousClock.now - start, bytes: bytes)
}
```

```
$ .build/debug/timeall https://www.swift.org https://forums.swift.org https://swiftpackageindex.com
1. https://www.swift.org  0.138212458 seconds  21342 bytes
2. https://forums.swift.org  0.391677125 seconds  86104 bytes
3. https://swiftpackageindex.com  0.702945667 seconds  148203 bytes
total: 0.704116083 seconds
```

The total is about the time of the slowest request, not the sum of all three, because they ran at the same time. (The timings are illustrative; yours will differ.)

The unit of concurrent work in Swift is a *task*. The top-level code runs in the program's main task, which calls `withTaskGroup` to create a *task group* for child tasks that each produce a `Timing`. The closure after `withTaskGroup` receives the group as `group`. For each URL, `group.addTask` starts a child task that runs `measure`, and all the children proceed concurrently. The `for await` loop then receives each child's result as it finishes, in whatever order they complete, and appends it to `timings`.

Task groups are *structured*: `withTaskGroup` doesn't return until every child task has finished, so no request can be left running in the background by accident, and by the time the program sorts the results, all of them are in.

`Timing` is a *struct*, a type that groups named values, declared on the spot (Section 4.4). Its `bytes` property is an optional, `nil` for a failed request. `try?` converts a throwing call into one that returns an optional, `nil` if an error was thrown. `ContinuousClock` measures elapsed time, and subtracting two of its instants gives a `Duration`, which prints in seconds. `map` on an optional transforms the value inside if there is one, and `??` supplies an alternative when there isn't, so `t.bytes.map { "\($0) bytes" } ?? "failed"` turns an `Int?` into a description.

The compiler is also checking something we haven't mentioned. Each child task runs concurrently with the code that created it, so anything a child task captures must be safe to share between concurrently running tasks. The children here capture only a `String`, and `Timing` is declared `Sendable`, which says it's safe to pass between tasks. Note that only the parent appends to `timings`. Had a child task tried to append to it directly, the program wouldn't compile, because two children could modify the array at once. Swift 6 checks for such *data races* at compile time, as Chapter 9 explains.

**Exercise 1.10:** Run `timeall` twice in a row on the same URLs. Do the times change? Why might they?

**Exercise 1.11:** Give each request a deadline of five seconds, and report requests that exceed it as timed out. (Hint: `URLRequest` has a `timeoutInterval` property; Section 8.7 shows a more general technique.)

## 1.7. A Web Server

Swift is used for servers as well as clients. Foundation includes an HTTP client but not a server, so for this section we'll use Hummingbird, a compact open-source web framework built on SwiftNIO, the networking library at the base of most server-side Swift. Using it means depending on another package, which the manifest declares:

```swift
// swift-tools-version: 6.4
import PackageDescription

let package = Package(
    name: "greet",
    platforms: [.macOS(.v15)],
    dependencies: [
        .package(url: "https://github.com/hummingbird-project/hummingbird.git", from: "2.0.0"),
    ],
    targets: [
        .executableTarget(
            name: "greet",
            dependencies: [.product(name: "Hummingbird", package: "hummingbird")]
        ),
    ]
)
```

The comment on the first line is read by the package manager; it gives the version of the manifest format. `dependencies` names each package we use and the range of versions we accept (any 2.x release here), and the target lists the libraries, or *products*, it uses. The first build downloads Hummingbird and its own dependencies and records the exact versions chosen in `Package.resolved`.

Our server greets visitors by name, taken from the path of the URL they request:

```swift
// swiftpl/ch1/greet1
// Greet1 is a web server that greets visitors by name.
import Hummingbird

let router = Router()
router.get("/", use: home)
router.get("hello/:name", use: hello)

let app = Application(
    router: router,
    configuration: .init(address: .hostname("localhost", port: 8080))
)
try await app.runService()

@Sendable func home(_ request: Request, _ context: BasicRequestContext) -> String {
    "Try /hello/yourname\n"
}

@Sendable func hello(_ request: Request, _ context: BasicRequestContext) -> String {
    let name = context.parameters.get("name") ?? "stranger"
    return "Hello, \(name)!\n"
}
```

A *router* matches each request's path against a list of patterns and calls the handler registered for the first that fits. In the pattern `hello/:name`, the component `:name` matches anything and captures it as a *parameter*, which the handler retrieves with `context.parameters.get("name")`. That returns an optional, so `?? "stranger"` provides a fallback. A handler receives the request and a *context* object (here `BasicRequestContext`, the router's default) and returns anything that can become a response; a `String` becomes a plain-text body with status `200 OK`. `runService` runs the server until it's stopped, for example by Control-C.

The `@Sendable` attribute marks a function that's safe to call from concurrently running tasks, which a request handler must be, since the server handles many requests at once. A function whose body is a single expression, like `home`, can omit `return`.

Start the server in the background, then try it with `download` from Section 1.5, or with a browser:

```
$ swift run greet1 &
$ download http://localhost:8080/hello/Ada
Hello, Ada!
http://localhost:8080/hello/Ada: status 200, 12 bytes
```

### 1.7.1. Remembering Visitors

Let's make the server count how many times each name has been greeted, and report the counts at `/stats`. That requires state shared by all the handlers. And because the server handles requests concurrently, two requests could try to update the counts at the same moment. Unprotected, that would be a *data race* (Section 9.1), and Swift 6 refuses to compile code that has one. If the counts were a plain global dictionary, both handlers below would be rejected.

The standard library's `Synchronization` module provides the protection we need: a `Mutex`, which holds a value and allows only one task at a time to access it. We declare it in its own file, since a `let` in a file other than `main.swift` is an ordinary global that any task may use, provided its type is safe to share, as a `Mutex` is:

```swift
// swiftpl/ch1/greet2/Sources/greet2/Greetings.swift
import Synchronization

/// greetings counts how many times each name has been greeted.
let greetings = Mutex<[String: Int]>([:])
```

The handlers reach the dictionary only through `withLock`, which locks the mutex, passes the protected value to a closure, and unlocks it when the closure is done:

```swift
// swiftpl/ch1/greet2/Sources/greet2/main.swift
// Greet2 greets visitors and keeps count.
import Hummingbird

let router = Router()
router.get("hello/:name", use: hello)
router.get("stats", use: stats)

let app = Application(
    router: router,
    configuration: .init(address: .hostname("localhost", port: 8080))
)
try await app.runService()

@Sendable func hello(_ request: Request, _ context: BasicRequestContext) -> String {
    let name = context.parameters.get("name") ?? "stranger"
    let n = greetings.withLock { counts in
        counts[name, default: 0] += 1
        return counts[name]!
    }
    return n == 1 ? "Hello, \(name)!\n" : "Welcome back, \(name)! (visit \(n))\n"
}

@Sendable func stats(_ request: Request, _ context: BasicRequestContext) -> String {
    let counts = greetings.withLock { $0 }  // a copy, taken under the lock
    var out = ""
    for (name, n) in counts.sorted(by: { $0.key < $1.key }) {
        out += "\(name)\t\(n)\n"
    }
    return out
}
```

`hello` updates the count and reads back the new value inside a single `withLock` call, so no other request can slip in between. `stats` takes a copy of the whole dictionary under the lock, `{ $0 }` simply returns the protected value, and then formats it after the lock is released. Because dictionaries are values, the copy can't be affected by later updates. Chapter 9 explores mutexes and their alternative, actors.

### 1.7.2. Serving Images

A handler can return images as well as text. Put the `spirograph` function from Section 1.4, with its constants, into another file of the package, and add a route for it in `main.swift`. Put it with the other routes, before the `Application` is created: `runService` doesn't return until the server stops, so code after it never runs while the server is up.

```swift
// swiftpl/ch1/greet2 (continued; also needs spirograph() and its constants from ch1/spirograph)
router.get("spirograph") { _, _ in
    Response(
        status: .ok,
        headers: [.contentType: "image/svg+xml"],
        body: ResponseBody(byteBuffer: ByteBuffer(string: spirograph()))
    )
}
```

This handler is a *trailing closure* written directly after the call, rather than a named function. It ignores both of its parameters, so it names them `_`. Since a plain string would be sent as text, the handler builds a full `Response` to set the `Content-Type` header, converting the SVG to a `ByteBuffer`, SwiftNIO's container for bytes. Each time you reload `http://localhost:8080/spirograph` in a browser, a new curve appears.

**Exercise 1.12:** Make `hello` accept a query parameter `?lang=es` (or `fr`, `de`, ...) and greet in that language. Read it with `request.uri.queryParameters["lang"]`.

**Exercise 1.13:** Let the spirograph route take `r` and `d` as query parameters, as in `/spirograph?r=0.35&d=0.7`, falling back to random values when they're absent or invalid.

## 1.8. Loose Ends

Here, briefly, are some features that the rest of the book uses before covering them fully.

**Switch:** A `switch` statement chooses among alternatives by matching a value against patterns. Cases are tried from top to bottom; the first match runs, and there's no fall-through into the next case, so no `break` is needed. Cases can match several values, ranges, and conditions:

```swift
func describe(status: Int) -> String {
    switch status {
    case 200, 204:
        return "success"
    case 300..<400:
        return "redirect"
    case 404:
        return "not found"
    case let code where code >= 500:
        return "server error \(code)"
    default:
        return "other"
    }
}
```

A `switch` must be *exhaustive*: some case has to match every possible value, which is why this one ends with `default`.

**Enumerations:** An `enum` defines a type with a fixed set of cases, each of which can carry *associated values*:

```swift
enum Shape {
    case circle(radius: Double)
    case rectangle(width: Double, height: Double)
}

func area(of shape: Shape) -> Double {
    switch shape {
    case .circle(let r):
        return .pi * r * r
    case .rectangle(let w, let h):
        return w * h
    }
}
```

This `switch` needs no `default`: the compiler knows that `circle` and `rectangle` are the only cases. If a case is added later, every such `switch` fails to compile until it handles the new one, which makes enums a reliable way to model data. `Optional` is itself an enum, with cases `.none` and `.some(value)`.

**Structs:** A `struct` groups named values into a new type:

```swift
struct Point {
    var x: Double
    var y: Double
}

var p = Point(x: 1, y: 2)
p.x += 0.5
```

The compiler supplies the *initializer* `Point(x:y:)`. Structs are values: assigning one to another variable copies it. Chapter 4 covers structs.

**Classes and references:** Swift has no pointers in everyday code. Instead it distinguishes *value types* (structs, enums, tuples, and all the standard collections) from *reference types*, chiefly classes and actors. Assigning a class instance to a second variable makes both refer to the same object. Objects are freed automatically when nothing refers to them any more, a scheme called *automatic reference counting*. For low-level work, Swift does have pointer types, as Chapter 13 shows.

**Methods and protocols:** A *method* is a function belonging to a type, and Swift lets you add methods to almost any type, including built-in ones like `Int` and `String`, using an *extension*. A *protocol* names a set of requirements, such as methods and properties, that conforming types must provide; `Equatable`, `Hashable`, `Comparable`, `Sequence`, and `Error` are protocols from the standard library. Chapter 6 covers methods, and Chapter 7 protocols.

**Packages:** Much of the work in real programs is done by libraries. Besides the standard library and Foundation, the Swift ecosystem offers thousands of open-source packages, searchable at `swiftpackageindex.com`. Documentation for the standard library and Foundation is at `developer.apple.com/documentation` and `swift.org`, and your editor can jump from any name to its declaration.

**Comments:** Besides `//` line comments, Swift has `/* ... */` block comments, which can be nested, so that commenting out code that already contains a block comment works. *Documentation comments* begin with `///` and use Markdown; editors display them, and the DocC tool turns them into documentation pages:

```swift
/// Returns the number of non-empty lines in `text`.
///
/// - Parameter text: Lines of text separated by `"\n"`.
func countNonEmptyLines(_ text: String) -> Int {
    text.split(separator: "\n").count
}
```
