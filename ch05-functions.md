# 5. Functions

Functions are how programs are divided into understandable pieces. A function gives a name to a computation, hides how it's done, and lets the same work be invoked from many places, by its author or by people who have never seen its body. Nearly everything interesting about how a language helps or hinders the organization of large programs shows up in its functions: how they're declared, how they receive and return values, how they report failure, and how they can be treated as values themselves.

This chapter covers all of that for Swift. Much of it uses HTML documents as raw material, because their tree structure gives recursion and higher-order functions natural work to do, and the chapter ends by building a tool that checks a web site for broken links.

## 5.1. Function Declarations

A function declaration has a name, a parameter list, optional *effects* (`async`, `throws`), an optional result type, and a body:

```swift
func name(parameter-list) effects -> result-type {
    body
}
```

Parameters are local constants that receive the *arguments* a caller passes. The result type, if any, follows `->`. A function without one returns `Void`, the empty tuple, and needs no `return` statement. A function with a result type must return a value on every path, unless a path ends in something that never returns, such as `fatalError` or an infinite loop. If the body is a single expression, its value is returned without writing `return`:

```swift
func clamp(_ x: Double, to range: ClosedRange<Double>) -> Double {
    min(max(x, range.lowerBound), range.upperBound)
}
print(clamp(1.7, to: 0...1))  // "1.0"
```

### 5.1.1. Argument Labels

Every parameter has an *argument label*, used when calling the function, and a *parameter name*, used inside its body. If you write one name, it serves as both:

```swift
func greet(person: String) -> String {
    "Hello, \(person)!"
}
greet(person: "Grace")
```

Writing two names gives them separate roles, and writing `_` as the label means callers pass that argument without one:

```swift
func send(_ message: String, to recipient: String) { ... }
send("Build finished", to: "ops")

func hypotenuse(_ a: Double, _ b: Double) -> Double { ... }
hypotenuse(3, 4)
```

Labels are part of a function's full name. Documentation refers to these functions as `greet(person:)`, `send(_:to:)`, and `hypotenuse(_:_:)`, and two functions may share a base name as long as their labels or parameter types differ. Swift picks among such *overloads* by looking at each call. That's how `Int(_:)` can convert from a `Double`, a `String`, or another integer type.

Choosing labels well is much of what makes a Swift API pleasant. The API Design Guidelines suggest omitting the first label when the base name already says what the first argument is (`append(item)`, `max(a, b)`), adding labels where arguments play distinct roles (`insert(item, at: 0)`), and using prepositions to make calls read as phrases (`send(_:to:)`, `index(after:)`, `distance(from:to:)`).

### 5.1.2. Default Arguments

A parameter can specify a default value, used when a caller leaves the argument out:

```swift
func wrapped(_ text: String, width: Int = 72, indent: String = "") -> String { ... }

wrapped(paragraph)
wrapped(paragraph, width: 40)
wrapped(paragraph, indent: "    ")
```

Defaults let one function serve both simple and detailed uses without a flock of overloads or an options object, and the standard library uses them widely: `print`'s `separator` and `terminator`, and `split`'s `maxSplits` and `omittingEmptySubsequences`, are a few examples.

### 5.1.3. Function Types

A function's *type* consists of its parameter types, its effects, and its result type. Labels and parameter names aren't part of it:

```swift
func double(_ x: Int) -> Int { x * 2 }
func negate(value: Int) -> Int { -value }
func scale(_ x: Int, by factor: Int) -> Int { x * factor }

print(type(of: double))  // "(Int) -> Int"
print(type(of: negate))  // "(Int) -> Int"
print(type(of: scale))  // "(Int, Int) -> Int"
```

`double` and `negate` have the same type, even though one has a label and the other doesn't, and are interchangeable wherever an `(Int) -> Int` is wanted, as Section 5.5 shows.

Arguments are passed *by value*: each parameter gets a copy of its argument, which the function can't change. To let a function modify a caller's variable, declare the parameter `inout` (Section 2.3.2). Passing a class instance copies the *reference*, so the function can modify the shared object, though not make the caller's variable refer to a different one.

## 5.2. Recursion

A function is *recursive* if it calls itself, directly or by way of other functions. Recursion is the natural way to process recursive data structures, such as trees, nested lists, file systems, and documents, because the shape of the code follows the shape of the data. We used it on a file tree in Section 4.4. Here we'll apply it to HTML.

An HTML document is a tree of *nodes*. Some nodes are *elements*, like `<h2>` or `<a href="...">`, which can contain other nodes; others hold text or comments. We'll use SwiftSoup (`https://github.com/scinfu/SwiftSoup`), an open-source HTML parser that copes with the malformed markup common on real web pages. It represents every node as an instance of the class `Node`. Elements are instances of its subclass `Element`, which has methods such as `tagName()`, `attr(_:)`, and `text()`.

Our first program reads an HTML document and prints its headings, `<h1>` through `<h6>`, as an indented outline, a quick table of contents for any page:

```swift
// swiftpl/ch5/toc
// Toc prints the headings of an HTML document as an indented outline.
import Foundation
import SwiftSoup

let input = FileHandle.standardInput.readDataToEndOfFile()
do {
    let doc = try SwiftSoup.parse(String(decoding: input, as: UTF8.self))
    var headings: [(level: Int, text: String)] = []
    collectHeadings(in: doc, into: &headings)
    for h in headings {
        print(String(repeating: "  ", count: h.level - 1) + h.text)
    }
} catch {
    printError("toc: \(error)")
    exit(1)
}

/// Appends the level and text of each heading in the tree rooted at n.
func collectHeadings(in n: Node, into headings: inout [(level: Int, text: String)]) {
    if let e = n as? Element, let level = headingLevel(e.tagName()) {
        headings.append((level, (try? e.text()) ?? ""))
        return  // a heading's descendants are part of its text
    }
    for child in n.getChildNodes() {
        collectHeadings(in: child, into: &headings)
    }
}

/// Returns 1...6 for the tag names h1...h6, and nil for any other tag.
func headingLevel(_ tag: String) -> Int? {
    guard tag.count == 2, tag.first == "h", let d = tag.last?.wholeNumberValue, (1...6).contains(d) else {
        return nil
    }
    return d
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

Given a page with this structure:

```html
<h1>User Guide</h1>
  <h2>Installing</h2>
    <h3>On macOS</h3>
    <h3>On Linux</h3>
  <h2>First Steps</h2>
```

`toc` prints:

```
User Guide
  Installing
    On macOS
    On Linux
  First Steps
```

`collectHeadings` handles one node and recurses into its children. If the node is a heading, it records it and stops, since the heading's text already includes its children. Otherwise, it calls itself on each child in turn, which handles that child's subtree, and so on to the leaves. The results accumulate in an array passed `inout` all the way down.

Two small features are worth noticing. `n as? Element` is a *conditional cast*: it yields the node as an `Element` if it is one and `nil` otherwise, so `if let e = n as? Element` tests the node's class and binds the result in one step. And SwiftSoup's `text()` can throw; `try?` turns the call into an optional, and `?? ""` substitutes an empty string on failure.

The program reads standard input, so it combines naturally with `download` from Section 1.5:

```
$ download https://example.com/guide.html | toc
```

The next function asks a different question: what's the most deeply nested element in a document, and what's the path to it? It's a quick way to spot a page whose structure has gotten out of hand:

```swift
// swiftpl/ch5/deepest
/// Returns the tag names along the longest path from n down to a leaf element.
func deepestPath(from n: Node, _ path: [String] = []) -> [String] {
    var path = path
    if let e = n as? Element {
        path.append(e.tagName())
    }
    var best = path
    for child in n.getChildNodes() {
        let candidate = deepestPath(from: child, path)
        if candidate.count > best.count {
            best = candidate
        }
    }
    return best
}

print(deepestPath(from: doc).joined(separator: " > "))
// e.g. "#root > html > body > div > main > section > ul > li > a > span"
```

The function appends to its own `path` and passes it to each recursive call, yet never removes anything. It doesn't need to. Arrays are values, so each call gets its own copy of the path, and appending in a callee can't affect the caller's copy. Copy-on-write makes those copies cheap: storage is actually duplicated only when a callee appends to a path its caller is still using.

Deep recursion has a cost to keep in mind. Each active call occupies space on the thread's stack, and the stack has a fixed size, typically 8 MB for the main thread and much less for other threads and tasks. A recursion that goes too deep overflows the stack and crashes the program. Real documents rarely nest more than a few dozen levels, but input from an untrusted source can be built to nest thousands deep. For such data, either limit the depth or replace recursion with a loop and an explicit stack.

**Exercise 5.1:** Rewrite `collectHeadings` without recursion, using an array as an explicit stack of nodes still to visit. Make sure the headings still come out in document order.

**Exercise 5.2:** Modify `toc` to number the headings in outline form, as in `1`, `1.1`, `1.2`, `2`.

**Exercise 5.3:** Write a function that reports every place where heading levels skip, such as an `<h4>` directly following an `<h2>`, which confuses screen readers.

**Exercise 5.4:** Write a function that lists every `<img>` element that lacks an `alt` attribute, together with the image's `src`.

## 5.3. Multiple Return Values

A Swift function returns exactly one value, but that value can be a tuple, which makes returning several values easy. We've seen standard library examples like `quotientAndRemainder(dividingBy:)` and `URLSession.data(from:)`.

The next function downloads a page and summarizes it, returning the page's title along with the number of headings and links. It can fail in several ways, so it's declared `throws`:

```swift
// swiftpl/ch5/summary
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif
import SwiftSoup

enum PageError: Error {
    case invalidURL(String)
    case badStatus(url: String, status: Int)
}

/// Downloads the page at url and returns its title and counts of headings and links.
func summarize(_ url: String) async throws -> (title: String, headings: Int, links: Int) {
    guard let u = URL(string: url) else {
        throw PageError.invalidURL(url)
    }
    let (data, response) = try await URLSession.shared.data(from: u)
    if let http = response as? HTTPURLResponse, http.statusCode != 200 {
        throw PageError.badStatus(url: url, status: http.statusCode)
    }
    let doc = try SwiftSoup.parse(String(decoding: data, as: UTF8.self))
    let headings = try doc.select("h1, h2, h3, h4, h5, h6").count
    let links = try doc.select("a[href]").count
    return (try doc.title(), headings, links)
}

for url in CommandLine.arguments.dropFirst() {
    do {
        let s = try await summarize(url)
        print("\(url): \"\(s.title)\", \(s.headings) headings, \(s.links) links")
    } catch {
        printError("summary: \(url): \(error)")
    }
}
```

The result type `(title: String, headings: Int, links: Int)` is a tuple with labeled elements, so callers can write `s.title` instead of `s.0`, or unpack the whole thing:

```swift
let (title, headingCount, linkCount) = try await summarize(url)
```

The labels serve as documentation in the signature. When a tuple result is used in many places, or grows past three or four elements, a small struct is clearer.

`summarize` contains four calls that can fail: the network request, the parse, and two queries, plus our own check of the HTTP status. Each `try` passes any error straight to the caller, so the function's main logic reads without interruption, yet every point where an error can escape is visibly marked. `select` finds the elements that match a CSS selector, a much shorter way to search a document than writing out the recursion ourselves.

**Exercise 5.5:** Extend `summarize` to also return the number of words of visible text on the page. (Section 4.3 shows how to split text into words; skip `<script>` and `<style>` elements.)

**Exercise 5.6:** In the `heatmap` program of Section 3.2.1, move the computation of a cell's `(x, y)` coordinates into a function that returns a labeled tuple `(x: Double, y: Double)`, and use the labels at the call site.

## 5.4. Errors

Some operations can't fail. `"abc".count` is always defined. Others can fail only if they're used incorrectly: an array subscript out of range is a mistake by the programmer, not a circumstance the program should plan for, and Swift responds to such mistakes by stopping the program, as Section 5.9 describes.

Many operations, though, can fail for reasons outside the program's control. Files go missing, networks go down, input arrives malformed. These are *expected* failures, and a robust program handles them deliberately. Swift offers two tools, and the choice between them depends on how much the caller needs to know.

When there's only one way to fail and no explanation is needed, return an *optional*. `Int("42")` returns `Int?`, because the only possible problem is that the text isn't a number. Looking up a dictionary key returns an optional because the only problem is that the key isn't there.

When failure can happen in different ways that the caller may need to distinguish or report, *throw an error*:

```swift
func loadSettings(from path: String) throws -> Settings
```

An error is a value of any type conforming to the `Error` protocol. The protocol has no requirements, so enums, structs, and classes can all be errors. Enums are the most common, because their cases list the ways an operation can fail, and associated values carry the details:

```swift
enum SettingsError: Error {
    case missing(path: String)
    case invalid(path: String, line: Int, reason: String)
}
```

Although `throw` and `catch` resemble exceptions in other languages, Swift's errors work differently. A throwing function says so in its declaration, so callers know. Every call to it is marked `try`, so readers know. And every error must be either handled with `do`-`catch` or propagated by a function that is itself `throws`, so nothing gets lost. Underneath, an error is returned much like an ordinary result, without the stack-unwinding machinery and stack-trace capture that make exceptions expensive in some languages.

### 5.4.1. Deciding What to Do with an Error

When a call throws, the code that made the call has to decide what to do. A handful of responses cover nearly every situation.

**Pass it on.** Most often, the function that encountered the failure can't do anything sensible about it, but its caller can. Writing `try` in a throwing function does exactly that: the error propagates to the caller unchanged.

```swift
let (data, response) = try await URLSession.shared.data(from: u)
```

**Add context, then pass it on.** An error from deep inside a library often lacks the information needed to make sense of it. A parse error that says "unexpected end of input" is much more useful when it also says *which* file was being parsed. Catch the error and throw a new one that wraps it:

```swift
struct LoadError: Error, CustomStringConvertible {
    var path: String
    var underlying: any Error
    var description: String { "loading \(path): \(underlying)" }
}

let settings: Settings
do {
    settings = try decoder.decode(Settings.self, from: data)
} catch {
    throw LoadError(path: path, underlying: error)
}
```

Since error messages are often chained like this, they read best when each part is a lowercase phrase without trailing punctuation, so that `loading config.json: missing key "port"` can be printed or logged as one line. The responsibility is shared: a function reports what *it* was trying to do and with what, and its caller adds what only it knows.

**Try again.** Some failures are transient, such as a busy server, a dropped connection, or a lock held by another process, and an operation that failed may succeed if retried after a pause. A general helper keeps the retry policy in one place:

```swift
// swiftpl/ch5/retry
/// Calls operation up to `attempts` times, doubling the delay after each failure.
/// Throws the last error if every attempt fails.
func withRetries<T>(
    attempts: Int = 4, initialDelay: Duration = .milliseconds(200),
    _ operation: () async throws -> T
) async throws -> T {
    var delay = initialDelay
    for attempt in 1... {
        do {
            return try await operation()
        } catch where attempt < attempts {
            print("attempt \(attempt) failed (\(error)); retrying in \(delay)")
            try await Task.sleep(for: delay)
            delay *= 2
        }
    }
    fatalError("unreachable")
}

let data = try await withRetries {
    try await URLSession.shared.data(from: statusURL).0
}
```

The loop runs over the unbounded range `1...`. A `catch` clause can have a `where` condition: while attempts remain, failures are caught, reported, and retried after a delay that doubles each time (*exponential backoff*, which keeps a struggling server from being hammered). On the last attempt the condition is false, nothing catches the error, and it propagates out of `withRetries`. The final `fatalError` is never reached; it satisfies the compiler that the function can't fall off its end. `Task.sleep` suspends without blocking a thread, and throws if the task is cancelled (Section 8.9). Retry only failures that might be transient; there's no point retrying an invalid URL.

**Stop the program.** If a failure means the program can't do its job, report it and exit. This is a decision for the program's top level, the `main.swift` or `main()` that knows what the program is for, not for library code, which should leave the decision to its callers:

```swift
do {
    try await run()
} catch {
    printError("backup: \(error)")
    exit(1)
}
```

An error that escapes from top-level code, or from a throwing `static func main()`, also ends the program with an error message and a failure status, which is fine for small tools.

**Log it and carry on.** Sometimes a failure means reduced service rather than none at all, as when an optional cache can't be reached or a single malformed record turns up in a large batch. Log what happened and continue. Server programs usually use the `swift-log` package, whose `Logger` supports levels and structured metadata:

```swift
import Logging
let logger = Logger(label: "importer")

for record in records {
    do {
        try importRecord(record)
    } catch {
        logger.warning("skipping record", metadata: ["id": "\(record.id)", "error": "\(error)"])
    }
}
```

**Ignore it, deliberately.** Occasionally failure really doesn't matter, and `try?` discards the error, turning the result into an optional:

```swift
try? FileManager.default.removeItem(at: scratchDirectory)  // best effort; the OS cleans up eventually
```

Because ignoring an error looks the same whether it was intended or not, leave a comment explaining why it's safe. A related operator, `try!`, asserts that a call *can't* fail and stops the program if it does. Use it only where failure would be a bug in your own code, such as decoding a JSON resource bundled with the program.

Swift error-handling code has a characteristic shape. A function checks for problems first, usually with `guard`, and exits early on failure, so that the main work follows at the outer level of indentation rather than nested inside `if` blocks.

### 5.4.2. Catching Specific Errors

A `catch` clause can include a pattern, so that each clause handles only the errors it matches. Clauses are tried in order, and one with no pattern catches everything remaining, binding it to `error`:

```swift
do {
    let settings = try loadSettings(from: path)
    start(with: settings)
} catch SettingsError.missing(let path) {
    print("no settings at \(path); using defaults")
    start(with: .defaults)
} catch let error as SettingsError {
    printError("bad settings: \(error)")
    exit(2)
} catch {
    printError("unexpected error: \(error)")
    exit(1)
}
```

The first clause matches one case of the enum and binds its associated value; the second matches any `SettingsError`, binding it with that type; the third catches everything else.

A function declared plain `throws` may throw any error type, so a caught error has the type `any Error`. Swift 6 also supports *typed throws*, where a function declares the single error type it throws:

```swift
func parseSettings(_ text: String) throws(SettingsError) -> Settings { ... }

do {
    let settings = try parseSettings(text)
} catch {
    // error is a SettingsError here, so a switch can be exhaustive
    switch error {
    case .missing(let path): ...
    case .invalid(let path, let line, let reason): ...
    }
}
```

Typed throws suit internal code whose failures form a closed set, and low-level and embedded code, where they avoid allocating a box for the error. For public library APIs, plain `throws` is usually wiser, since it leaves room to add new kinds of failure later without breaking callers.

An error type controls how it's described by conforming to `CustomStringConvertible`, as `LoadError` does above. Foundation's `LocalizedError` protocol provides user-facing messages through `errorDescription`, which `error.localizedDescription` returns.

### 5.4.3. Expected Outcomes Are Not Errors

Not every unusual result is an error. Reaching the end of a file, finding no match for a search, or getting an empty response may be perfectly normal, and code shouldn't have to catch an error to detect them. Swift APIs generally express such outcomes in their result types: `readLine()` returns `nil` at end of input, `firstIndex(of:)` returns `nil` when there's no match, and asynchronous iterators return `nil` from `next()` when a sequence is exhausted. Genuine problems, such as a read failure or a broken connection, are thrown.

The principle is worth following in your own APIs. If a condition is a normal outcome of using a function correctly, make it part of the return type, as an optional or an enum. Throw only when something has actually gone wrong.

## 5.5. Function Values

Functions are values in Swift. A function can be stored in a variable, passed to another function, or returned as a result, and a variable holding a function can be called like the function itself:

```swift
var transform = double
print(transform(21))  // "42"

transform = negate
print(transform(21))  // "-21"

transform = scale  // compile error: cannot assign '(Int, Int) -> Int' to '(Int) -> Int'
```

Functions can't be compared with `==`, so they can't be dictionary keys or set elements.

Passing functions as arguments lets one piece of code be customized by another. The standard library's sequence algorithms are built this way: `map`, `filter`, `reduce`, `sorted(by:)`, `first(where:)`, `contains(where:)`, `allSatisfy`, and `compactMap` each take a function that says what to do with each element. Here's a classic letter-substitution cipher, ROT13, which replaces each letter with the one 13 places later in the alphabet. Since there are 26 letters, applying it twice restores the original:

```swift
func rot13(_ c: Character) -> Character {
    guard c.isLetter, let ascii = c.asciiValue else { return c }
    let base: UInt8 = c.isUppercase ? 65 : 97  // "A" or "a"
    return Character(Unicode.Scalar((ascii - base + 13) % 26 + base))
}

let secret = String("Hello, Swift!".map(rot13))
print(secret)  // "Uryyb, Fjvsg!"
print(String(secret.map(rot13)))  // "Hello, Swift!"
```

`map` calls `rot13` on each character of the string and collects the results in an array of characters, which `String(_:)` joins back together. Note that we pass `rot13` by name, without parentheses: the name of a function, used as a value, refers to the function itself. Initializers can be used the same way: `["3", "x", "7"].compactMap(Int.init)` calls `Int("3")`, `Int("x")`, and `Int("7")` and keeps the successes, `[3, 7]`.

Function values also let us separate *how* a structure is traversed from *what's done* at each node. Here's a general traversal for HTML trees that calls a function for every node, telling it how deep the node is:

```swift
// swiftpl/ch5/walk
/// Calls visit for each node in the tree rooted at n, in document order,
/// along with the node's depth below n.
func walk(_ n: Node, depth: Int = 0, _ visit: (Node, Int) -> Void) {
    visit(n, depth)
    for child in n.getChildNodes() {
        walk(child, depth: depth + 1, visit)
    }
}
```

Now each question about a document needs only the code that's specific to it. Counting how often each tag is used:

```swift
var tagCounts: [String: Int] = [:]
walk(doc) { node, _ in
    if let e = node as? Element {
        tagCounts[e.tagName(), default: 0] += 1
    }
}
for (tag, n) in tagCounts.sorted(by: { $0.value > $1.value }).prefix(5) {
    print("\(tag)\t\(n)")
}
```

Printing the structure of the page's lists, indented by depth:

```swift
walk(doc) { node, depth in
    if let e = node as? Element, ["ul", "ol", "li"].contains(e.tagName()) {
        print(String(repeating: "  ", count: depth) + "<\(e.tagName())>")
    }
}
```

The braces after each call to `walk` are *closures*, functions written inline, as the last argument. That's the subject of the next section. The point here is that `walk` is written once, and each use supplies only its own logic.

**Exercise 5.7:** Change `walk` so that `visit` returns a `Bool`, and children are visited only when it returns `true`. Use the new version to print the visible text of a page, skipping the contents of `<script>` and `<style>` elements.

**Exercise 5.8:** Write `firstElement(in:where:)`, which returns the first element satisfying a predicate and stops searching as soon as it finds one. Use it to find the element with a given `id`.

**Exercise 5.9:** Write a function `compose(_ fs: [(Double) -> Double]) -> (Double) -> Double` that returns a single function applying each of `fs` in turn, and use it to build a temperature conversion out of a scaling and an offset.

## 5.6. Closures

Named functions are declared with `func`, at the top level or inside other functions. Functions can also be written as *closure expressions*, anonymous functions that can appear wherever a function value is needed. A closure is written in braces, with its parameters and result type, if any, before the keyword `in`:

```swift
let lengths = ["fig", "banana", "kiwi"].map({ (s: String) -> Int in return s.count })
```

That's the long form. In practice, Swift lets you leave out nearly everything the compiler can infer. Parameter and result types come from context; a single-expression body needs no `return`; parameters can be referred to by position as `$0`, `$1`, and so on; and a closure that's the last argument can follow the call's parentheses as a *trailing closure*, with the parentheses omitted entirely if nothing else is inside them. So the line above is usually written:

```swift
let lengths = ["fig", "banana", "kiwi"].map { $0.count }
```

These forms are all common:

```swift
words.sorted(by: { a, b in a.count < b.count })
words.sorted { $0.count < $1.count }
words.sorted(by: >)  // operators are functions too
words.map(\.count)  // a key path stands in for { $0.count }
```

What makes closures more than syntax is that they *capture* the variables around them. A closure can use, and even modify, the local variables of the function in which it was created, and those variables live as long as the closure does:

```swift
// swiftpl/ch5/ids
/// Returns a function that produces IDs like "INV-0001", "INV-0002", ...
func makeIDGenerator(prefix: String) -> () -> String {
    var count = 0
    return {
        count += 1
        let digits = String(count)
        return prefix + "-" + String(repeating: "0", count: max(0, 4 - digits.count)) + digits
    }
}

let invoiceID = makeIDGenerator(prefix: "INV")
print(invoiceID(), invoiceID())  // "INV-0001 INV-0002"
let orderID = makeIDGenerator(prefix: "ORD")
print(orderID(), invoiceID())  // "ORD-0001 INV-0003"
```

Each call to `makeIDGenerator` creates a new `count` variable and returns a closure that captures it. The two generators have separate counts, and each count persists from call to call, long after `makeIDGenerator` has returned. A closure, in other words, is code plus captured state. That's why closures are reference types, and why two function values can't meaningfully be compared.

### 5.6.1. Escaping Closures

Swift distinguishes two ways a function can use a closure it's given. `map` and `sorted(by:)` call their closure during the call and then forget it; the closure is *non-escaping*. A closure that's stored for later (in a property, a collection, or a task), or returned, *escapes* the call. Closure parameters are non-escaping by default, and a parameter whose closure may escape must say so with `@escaping`:

```swift
final class Timer {
    private var onFire: (() -> Void)?

    func schedule(_ action: @escaping () -> Void) {
        onFire = action  // stored for later, so it escapes
    }
}
```

The distinction helps both the compiler, which can keep non-escaping closures and their captured variables on the stack, and the reader: a non-escaping closure can't outlive the call, so it can't cause surprises later.

Escaping closures that capture `self` are the most common source of reference cycles (Section 2.3.4). If an object stores a closure that holds the object strongly, neither can ever be freed. A *capture list* fixes this by capturing `self` weakly:

```swift
timer.schedule { [weak self] in
    guard let self else { return }  // the object may have been freed in the meantime
    self.refresh()
}
```

In an escaping closure inside a class, Swift insists that uses of the object's members be written with an explicit `self.` (or that `self` appear in the capture list), so that the capture can't go unnoticed. Value types don't need this, since they can't form reference cycles.

### 5.6.2. Example: Build Order

A package's targets depend on each other, and before a target can be compiled, everything it depends on must be compiled first. Given the dependencies, what's a valid order in which to build everything? This is *topological sorting*: arranging the nodes of a directed acyclic graph so that every edge points forward.

Here are the targets of a hypothetical app and their direct dependencies:

```swift
// swiftpl/ch5/buildorder
let dependencies: [String: [String]] = [
    "App": ["Networking", "Storage", "UI"],
    "AppTests": ["App", "TestSupport"],
    "Core": ["Logging"],
    "Logging": [],
    "Networking": ["Core", "Logging"],
    "Storage": ["Core"],
    "TestSupport": ["Core"],
    "UI": ["Core"],
]
```

A *depth-first search* produces a valid order: before emitting a target, emit (recursively) everything it depends on, skipping anything already emitted.

```swift
func buildOrder(_ deps: [String: [String]]) -> [String] {
    var order: [String] = []
    var done = Set<String>()

    func visit(_ targets: [String]) {
        for target in targets where !done.contains(target) {
            done.insert(target)
            visit(deps[target] ?? [])
            order.append(target)
        }
    }

    visit(deps.keys.sorted())
    return order
}

for (i, target) in buildOrder(dependencies).enumerated() {
    print("\(i + 1). \(target)")
}
```

```
1. Logging
2. Core
3. Networking
4. Storage
5. UI
6. App
7. TestSupport
8. AppTests
```

`visit` is a *nested function*, declared inside `buildOrder`. Like a closure, it can use the enclosing function's local variables, here `order`, `done`, and `deps`. Unlike a closure stored in a variable, it can call itself recursively by name, which is just what a depth-first search needs. Starting the search from the *sorted* list of targets makes the output the same on every run, despite dictionaries' unpredictable iteration order.

**Exercise 5.10:** If the dependencies contain a cycle, say `Core` depends on `UI`, `buildOrder` silently produces an invalid order. Make it detect cycles and report the targets involved. (Hint: track which targets are *in progress* as well as which are done.)

**Exercise 5.11:** Targets with no dependency between them can be built in parallel. Write a function that groups the targets into *levels*, where every target in a level depends only on targets in earlier levels, so that each level can be built all at once.

### 5.6.3. Example: Link Checker

Web sites accumulate broken links as pages move and other sites disappear. Let's build a tool that visits every page of a site and reports links that no longer work.

The first piece is a function that downloads a page and returns its links. We'll put it in a small library module, `Links`, since Chapter 8 uses it as well:

```swift
// swiftpl/ch5/links/Sources/Links/Links.swift
// Package Links provides a link-extraction function.
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif
import SwiftSoup

public enum LinksError: Error {
    case invalidURL(String)
    case badStatus(url: String, status: Int)
}

/// Makes an HTTP GET request to the specified URL, parses the response
/// as HTML, and returns the links in the HTML document, as absolute URLs.
public func extract(_ url: String) async throws -> [String] {
    guard let base = URL(string: url) else {
        throw LinksError.invalidURL(url)
    }
    let (data, response) = try await URLSession.shared.data(from: base)
    if let http = response as? HTTPURLResponse, http.statusCode != 200 {
        throw LinksError.badStatus(url: url, status: http.statusCode)
    }
    let doc = try SwiftSoup.parse(String(decoding: data, as: UTF8.self))
    var links: [String] = []
    for a in try doc.select("a[href]") {
        let href = try a.attr("href")
        guard let link = URL(string: href, relativeTo: response.url ?? base) else {
            continue  // ignore bad URLs
        }
        links.append(link.absoluteString)
    }
    return links
}
```

`select("a[href]")` finds every `<a>` element that has an `href` attribute. Links in HTML are often *relative*, like `../about.html`, so each one is resolved against the URL of the page it came from, giving an absolute URL that can be requested directly. `response.url` is used as the base rather than the URL we asked for, because the server may have redirected us elsewhere.

The checker itself keeps a *worklist* of pages to visit, starting with the URL given on the command line. For each page, it extracts the links and checks each one it hasn't already checked. Links to pages on the same site that work are added to the worklist, so that the whole site is eventually covered; links to other sites are checked but not followed:

```swift
// swiftpl/ch5/linkcheck
// Linkcheck visits every page of a site and reports broken links.
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif
import Links

guard let start = CommandLine.arguments.dropFirst().first,
    let site = URL(string: start)?.host()
else {
    printError("usage: linkcheck URL")
    exit(2)
}

var worklist = [start]
var visited = Set<String>()
var statuses: [String: Int] = [:]  // HTTP status of each link checked; 0 if unreachable

while !worklist.isEmpty {
    let page = worklist.removeFirst()
    guard visited.insert(page).inserted else { continue }
    let links: [String]
    do {
        links = try await extract(page)
    } catch {
        printError("\(page): \(error)")
        continue
    }
    for link in links {
        if statuses[link] == nil {
            statuses[link] = await status(of: link)
        }
        let code = statuses[link]!
        if !(200..<400).contains(code) {
            print("\(page): broken link \(link) (\(code == 0 ? "unreachable" : String(code)))")
        } else if URL(string: link)?.host() == site {
            worklist.append(link)  // a working page on the same site: visit it too
        }
    }
}
print("checked \(statuses.count) links on \(visited.count) pages")

/// Returns the HTTP status for a HEAD request to link, or 0 if it can't be reached.
func status(of link: String) async -> Int {
    guard let url = URL(string: link) else { return 0 }
    var request = URLRequest(url: url)
    request.httpMethod = "HEAD"  // we need only the status, not the body
    guard let result = try? await URLSession.shared.data(for: request) else { return 0 }
    return (result.1 as? HTTPURLResponse)?.statusCode ?? 0
}
```

```
$ swift run linkcheck https://example.org/
https://example.org/docs/: broken link https://example.org/docs/old-guide.html (404)
https://example.org/about/: broken link https://defunct.example.net/ (unreachable)
checked 412 links on 37 pages
```

(The site and results are illustrative.) Each link is checked only once, however many pages contain it, thanks to the `statuses` dictionary, and each page is visited only once, thanks to `visited`. The checker uses `HEAD` requests, which return a response's status and headers without its body, to avoid downloading every linked file just to see whether it exists.

Visiting pages in the order they were added to the worklist makes the search *breadth-first*: the start page, then everything it links to, then everything those link to. `removeFirst()` on an array takes time proportional to the array's length; for very large sites, a double-ended queue such as `Deque` from the `swift-collections` package does it in constant time.

The checker is also slow, because it does one request at a time, and spends nearly all its time waiting on the network. Chapter 8 shows how to make a crawler like this one concurrent.

**Exercise 5.12:** Links that differ only in their fragment (`page.html#intro` and `page.html#usage`) refer to the same page. Strip fragments before checking or visiting, so that each page is requested once.

**Exercise 5.13:** Some servers don't support `HEAD` requests and answer them with an error. When a `HEAD` request fails with status 405, retry the link with `GET` before reporting it broken.

## 5.7. Variadic Functions

A *variadic* function accepts any number of arguments of one type, like `print`. To declare one, follow the parameter's type with `...`:

```swift
// swiftpl/ch5/average
func average(_ values: Double...) -> Double? {
    guard !values.isEmpty else { return nil }
    return values.reduce(0, +) / Double(values.count)
}

print(average(3, 4, 8) as Any)  // "Optional(5.0)"
print(average() as Any)  // "nil"
```

Inside the function, `values` is an ordinary array, `[Double]`. Callers write the arguments individually, and the compiler collects them into the array.

Unlike some languages, Swift has no syntax for passing an existing array to a variadic parameter, so if `readings` is an array, `average(readings)` doesn't compile. The usual pattern is to write the real function to take an array and add a variadic convenience that forwards to it:

```swift
func average(_ values: [Double]) -> Double? {
    values.isEmpty ? nil : values.reduce(0, +) / Double(values.count)
}

func average(_ values: Double...) -> Double? {
    average(values)
}
```

A variadic parameter doesn't have to come last, and a function can have more than one, as long as the parameter after each has a label. That makes this a natural signature for a small logging function:

```swift
func log(_ items: Any..., level: String = "INFO") {
    let message = items.map { "\($0)" }.joined(separator: " ")
    print("[\(level)] \(message)")
}

log("cache miss for", "user:42", "after", 3, "attempts", level: "DEBUG")
// "[DEBUG] cache miss for user:42 after 3 attempts"
```

**Exercise 5.14:** Write a variadic `maximum` function that requires at least one argument, by declaring its signature as `maximum(_ first: Int, _ rest: Int...)`.

**Exercise 5.15:** Write a variadic function `commonPrefix(_ strings: String...) -> String` that returns the longest prefix shared by all its arguments.

## 5.8. Deferred Statements

Programs constantly acquire resources that must be given back: files to close, locks to release, temporary directories to delete. The difficulty is giving them back on *every* path out of a function, including early returns and thrown errors, without duplicating the cleanup code at each exit.

Swift's `defer` statement solves this. A `defer` block runs when control leaves the scope that contains it, however that happens: falling off the end, `return`, `break`, `continue`, or a thrown error. Several `defer` blocks in one scope run in reverse order, so resources are released in the opposite order to their acquisition. The natural place for a `defer` is right after the resource has been acquired, where the pairing is obvious:

```swift
// swiftpl/ch5/firstline
import Foundation

/// Returns the first line of the file at path, or nil if the file is empty.
func firstLine(of path: String) throws -> String? {
    guard let file = FileHandle(forReadingAtPath: path) else {
        throw CocoaError(.fileNoSuchFile)
    }
    defer { try? file.close() }

    let chunk = try file.read(upToCount: 4096) ?? Data()
    return String(decoding: chunk, as: UTF8.self)
        .split(separator: "\n", omittingEmptySubsequences: false)
        .first
        .map(String.init)
}
```

Whether the function returns a line, returns `nil`, or throws from `read`, the file is closed.

A Swift `defer` belongs to its enclosing *scope*, not to the whole function. A `defer` inside a loop body runs at the end of each iteration, which is usually what you want:

```swift
for path in paths {
    guard let file = FileHandle(forReadingAtPath: path) else { continue }
    defer { try? file.close() }  // closed at the end of this iteration
    // ...read the file...
}
```

(In some languages, `defer` waits until the function returns, and a loop like this one can run out of file descriptors before any of them is closed.)

`defer` is also handy for anything that must happen on exit, such as recording how long a function took:

```swift
func rebuildIndex() throws {
    let start = ContinuousClock.now
    defer { print("rebuildIndex finished in \(ContinuousClock.now - start)") }
    // ...work that may return early or throw...
}
```

A `defer` block can read the scope's variables, but it can't `return`, `break`, or `throw` out of itself; it exists to clean up, not to change how the scope ends.

Before reaching for `defer`, check whether the resource can manage itself. A class can release what it owns in `deinit` when its last reference disappears; noncopyable types (marked `~Copyable`, like the `Mutex` of Section 9.2) let the compiler guarantee that a resource has exactly one owner and is consumed exactly once; and many APIs offer a *with* function that holds a resource only for the duration of a closure, like `Mutex.withLock` in Section 1.7. When one of those fits, it's usually cleaner still.

**Exercise 5.16:** Write a function that copies a file in chunks, using `defer` to close both files. If the copy fails partway through, the partly written destination should be deleted.

## 5.9. Traps

The compiler catches many mistakes, but some can be detected only while the program runs: an array index out of range, a force-unwrapped `nil`, an integer overflow, an integer conversion that loses information. When Swift detects one of these, the program *traps*: it stops at once, with a message describing what went wrong.

A trap isn't an exception, and it can't be caught. No `catch` clause sees it, and `defer` blocks and `deinit`s don't run. The program prints a diagnostic and a backtrace and terminates:

```
$ swift run inventory
Fatal error: Index out of range
...
Thread 0 crashed:
0 Swift/ContiguousArrayBuffer.swift in _checkSubscript(_:wasNativeTypeChecked:)
1 inventory at Sources/inventory/main.swift:18:24
...
```

(On Linux, the Swift runtime prints a symbolicated backtrace like this one; on macOS, the system records a crash report, and a debugger stops at the failing instruction. The exact format varies.)

The reasoning is that a trap signals a *bug*: something the programmer was sure couldn't happen has happened. From that point, the program's state can't be trusted, and continuing risks corrupting data or creating a security hole. Stopping is the safe response. Failures that are part of normal operation, such as bad input or a missing file, should be optionals or thrown errors, which the program can handle.

You can also trap deliberately, to check your own assumptions. Swift provides several functions for this, which differ in when they're checked:

```swift
precondition(items.count <= capacity, "too many items")
assert(isSorted(index), "index out of order")
fatalError("not implemented")
```

`precondition(_:_:)` checks a condition that callers are responsible for meeting, and traps if it's false, in debug and release builds alike. It's the right way for a function to reject arguments that violate its documented requirements.

`assert(_:_:)` checks an internal assumption, but only in debug builds. In optimized builds it's removed entirely, so it's suitable for expensive consistency checks.

`fatalError(_:)` traps unconditionally, in every build. Its return type is `Never`, a type with no values, so the compiler knows that execution can't continue past it. That makes it usable wherever a value is required but can't actually be produced:

```swift
func compassPoint(for degrees: Int) -> String {
    precondition((0..<360).contains(degrees), "invalid heading \(degrees)")
    switch degrees {
    case 0..<90: return "northeast"
    case 90..<180: return "southeast"
    case 180..<270: return "southwest"
    case 270..<360: return "northwest"
    default: fatalError("unreachable: \(degrees)")
    }
}
```

`preconditionFailure(_:)` and `assertionFailure(_:)` are the unconditional forms of the first two.

Checking preconditions is good practice, but there's no point in duplicating checks that Swift already makes, such as testing an array index just before subscripting with it, unless your check produces a clearer message or catches the problem earlier. And never use a trap to handle data that comes from outside the program: user input, files, and network responses can be wrong without anyone having made a programming mistake, so they deserve errors, not crashes.

The `-Ounchecked` optimization level removes many of these checks, including overflow checks and preconditions, so that violations cause undefined behavior instead of traps. It's almost never worth it. The checks are cheap, and the safety they buy is a large part of the reason to use Swift.

**Exercise 5.17:** Write a program that can trap in six different ways, chosen by a command-line argument: an out-of-range index, a force-unwrapped `nil`, an integer overflow, a failed `Int8(_:)` conversion, a failed `precondition`, and `fatalError`. Compare the messages, and find out which checks remain in a release build (`-c release`).

## 5.10. Recovering from Failure

Swift has no way to catch a trap and keep going. Some languages offer one, and programs in those languages use it for two main purposes: to unwind from deep inside a recursive algorithm when something goes wrong, and to keep a server running when one request hits a bug. Swift handles both differently.

For the first, ordinary thrown errors are the answer. They pass cheaply through any number of nested calls, needing only a `try` at each level. Here's the skeleton of a recursive-descent parser that stops at the first syntax error:

```swift
// swiftpl/ch5/parse
struct SyntaxError: Error, CustomStringConvertible {
    var position: Int
    var message: String
    var description: String { "at offset \(position): \(message)" }
}

struct Parser {
    var tokens: [Token]
    var pos = 0

    mutating func parseExpr() throws(SyntaxError) -> Expr {
        var lhs = try parseTerm()
        while let op = peek(), op == .plus || op == .minus {
            advance()
            let rhs = try parseTerm()
            lhs = .binary(op, lhs, rhs)
        }
        return lhs
    }

    mutating func parsePrimary() throws(SyntaxError) -> Expr {
        guard let tok = peek() else {
            throw SyntaxError(position: pos, message: "unexpected end of input")
        }
        // ...
    }
    // ...
}
```

The parsing functions declare `throws(SyntaxError)`, since a syntax error is the only thing that can go wrong. When the innermost function detects a problem, it throws, and the error passes back through every `try` to whoever called `parseExpr`. Section 7.9 builds a complete parser along these lines.

Sometimes it's useful to hold on to the *outcome* of a throwing operation, success or failure, as a value, to store it, return it, or examine it later. The standard library's `Result` enum does that, with cases `.success(value)` and `.failure(error)`:

```swift
let outcome = Result { try parse(input) }  // captures either the value or the error

switch outcome {
case .success(let expr):
    print(expr)
case .failure(let error):
    print("error: \(error)")
}

let expr = try outcome.get()  // back to a throwing call
```

`Result` was the standard way to report the outcome of asynchronous work before Swift had `async`/`await`. It's still useful for collecting the outcomes of many independent operations, some of which may have failed.

For the second case, a server that shouldn't go down because of one bad request, the Swift answer is *isolation at the level of processes*. A server built with Hummingbird or Vapor doesn't try to survive a trap. It runs under a supervisor, such as `systemd`, a container orchestrator, or a platform that keeps several replicas behind a load balancer, which restarts it if it exits. Swift programs start quickly and have no garbage collector to warm up, so restarts are cheap. And the trap stopped the bug at the moment it was detected, rather than letting it quietly corrupt memory and fail mysteriously later.

The resulting division of labor is easy to state: expected failures are optionals or thrown errors, which callers can always handle; bugs are traps, which end the program before they can do further damage. Much of writing robust Swift is deciding, for each thing that can go wrong, which kind it is.

**Exercise 5.18:** Write a function with a non-`Void` result that contains no `return` statement. (Hint: there are at least three ways: a single-expression body, a `switch` expression, and a call to a `Never`-returning function.)

**Exercise 5.19:** Modify `timeall` from Section 1.6 so that each child task returns a `Result<Int, any Error>` holding the response size or the error, and the parent counts successes and failures separately.
