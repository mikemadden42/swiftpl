# 5. Functions

A function lets us wrap up a sequence of statements as a unit that can be called from elsewhere in a program, perhaps multiple times. Functions make it possible to break a big job into smaller pieces that might well be written by different people separated by both time and space. A function hides its implementation details from its users. For all of these reasons, functions are a critical part of any programming language.

We've seen many functions already. Now let's take time for a more thorough discussion. The running example of this chapter is a web crawler, that is, the component of a web search engine responsible for fetching web pages, discovering the links within them, fetching the pages identified by those links, and so on. A web crawler gives us ample opportunity to explore recursion, closures, errors, `defer`, and traps.

## 5.1. Function Declarations

A function declaration has a name, a list of parameters, an optional list of effects (`async`, `throws`), an optional result type, and a body:

```swift
func name(parameter-list) effects -> result-type {
    body
}
```

The parameter list specifies the names and types of the function's *parameters*, which are the local constants whose values or *arguments* are supplied by the caller. The result type specifies the type of the value that the function returns. A function may return multiple values by returning a tuple. If the function returns nothing, the result type is omitted (it's implicitly `Void`, the empty tuple). Functions that have a result type must end with a `return` statement unless execution clearly can't reach the end of the function, perhaps because the function ends with a call to `fatalError` or an infinite loop with no `break`. As we've seen, a function whose body is a single expression may omit the `return`.

```swift
func hypot(_ x: Double, _ y: Double) -> Double {
    (x * x + y * y).squareRoot()
}
print(hypot(3, 4))  // "5.0"
```

### 5.1.1. Argument Labels

Each parameter has two names. The *argument label* is used at the call site, and the *parameter name* is used inside the function. By default, they're the same:

```swift
func greet(person: String) -> String {
    "Hello, \(person)!"
}
greet(person: "Ada")
```

To use a different label, write it before the parameter name; to omit the label at the call site, use `_`:

```swift
func move(from start: Point, to end: Point) { ... }
move(from: a, to: b)

func max(_ x: Int, _ y: Int) -> Int { ... }
max(1, 2)
```

Labels are part of a function's name. The functions above are referred to in documentation as `greet(person:)`, `move(from:to:)`, and `max(_:_:)`. Two functions may have the same base name and different labels, or the same labels and different parameter types; Swift picks the right *overload* by looking at the call. This is how `print(_:separator:terminator:)` and `print(_:separator:terminator:to:)` coexist, and how `Int(_:)` can convert from a `Double`, a `String`, or a `Bool`.

The API Design Guidelines describe when to use labels. Omit the first label when the function's base name already makes the role of the first argument clear, as in `max(x, y)` or `array.append(x)`; use labels when an argument plays a distinct role, as in `array.insert(x, at: 0)`; and use prepositions to make the call read as a phrase: `move(from:to:)`, `index(after:)`.

### 5.1.2. Default Arguments

A parameter may specify a default value, which is used if the caller omits the argument:

```swift
func join(_ parts: [String], separator: String = ", ") -> String {
    parts.joined(separator: separator)
}
join(["a", "b"])  // "a, b"
join(["a", "b"], separator: "/")  // "a/b"
```

Go has no default arguments and uses option structs or variadic option functions instead. In Swift, defaults are the idiom, and many standard library functions use them: `print`'s `separator` and `terminator`, `split`'s `maxSplits` and `omittingEmptySubsequences`, and so on.

### 5.1.3. Function Types

The type of a function is sometimes called its *signature*. Two functions have the same type if they have the same sequence of parameter types, the same effects, and the same result type. The names of parameters and their labels don't affect the type:

```swift
func add(_ x: Int, _ y: Int) -> Int { x + y }
func sub(x: Int, y: Int) -> Int { x - y }
func first(_ x: Int, _: Int) -> Int { x }
func zero(_: Int, _: Int) -> Int { 0 }

print(type(of: add))  // "(Int, Int) -> Int"
print(type(of: sub))  // "(Int, Int) -> Int"
```

Every function call must provide an argument for each parameter that has no default, in the order in which the parameters were declared, with the correct labels.

Arguments are passed *by value*, so the function receives a copy of each argument; it can't modify the caller's variables unless a parameter is declared `inout`. For reference types like class instances, the reference is copied, so the function can modify the object that the reference refers to.

You may occasionally encounter a function declaration without a body, indicating that the function is implemented in a language other than Swift, or that it's a protocol requirement. We'll see protocols in Chapter 7.

## 5.2. Recursion

Functions may be *recursive*, that is, they may call themselves, either directly or indirectly. Recursion is a powerful technique for many problems, and of course it's essential for processing recursive data structures. In Section 4.4, we used recursion to measure and print a tree of files. In this section, we'll use it again for processing HTML documents.

The example program below uses SwiftSoup, an open-source HTML parser for Swift (`https://github.com/scinfu/SwiftSoup`). It reads HTML text, tolerating the many errors found in real-world pages, and produces a tree of nodes. HTML has several kinds of nodes (text, comments, and so on), but here we are concerned only with *elements* of the form `<name key='value'>`. In SwiftSoup, every node is an instance of the class `Node`, and elements are instances of its subclass `Element`, whose `tagName()` method returns the element's name and whose `attr(_:)` method returns the value of an attribute.

The main function parses the standard input as HTML, extracts the links using a recursive `visit` function, and prints each discovered link:

```swift
// swiftpl/ch5/findlinks1
// Findlinks1 prints the links in an HTML document read from standard input.
import Foundation
import SwiftSoup

let input = FileHandle.standardInput.readDataToEndOfFile()
do {
    let doc = try SwiftSoup.parse(String(decoding: input, as: UTF8.self))
    var links: [String] = []
    visit(doc, appendingLinksTo: &links)
    for link in links {
        print(link)
    }
} catch {
    printError("findlinks1: \(error)")
    exit(1)
}

/// Appends to links each link found in n and its descendants.
func visit(_ n: Node, appendingLinksTo links: inout [String]) {
    if let e = n as? Element, e.tagName() == "a",
        let href = try? e.attr("href"), !href.isEmpty
    {
        links.append(href)
    }
    for child in n.getChildNodes() {
        visit(child, appendingLinksTo: &links)
    }
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

The `visit` function traverses an HTML node tree, extracts the link from the `href` attribute of each *anchor* element `<a href='...'>`, and appends the links to an array passed `inout`. To descend the tree for a node `n`, `visit` recursively calls itself for each of `n`'s children.

The condition of the `if` statement shows several features working together. The expression `n as? Element` is a *conditional downcast*: it produces the node as an `Element` if it is one, and `nil` otherwise, so `if let e = n as? Element` both tests the node's type and binds it. The comma-separated clauses then require the tag name to be `"a"` and the `href` attribute to be present and non-empty. `attr` is declared `throws`, and `try?` converts its result to an optional, `nil` if it threw.

Let's run `findlinks` on the Swift home page, piping the output of `fetch` (Section 1.5) to the input of `findlinks`. We've edited the output slightly for brevity.

```
$ swift build
$ fetch https://www.swift.org | findlinks1
/
/install/
/documentation/
/community/
/packages/
/blog/
https://github.com/swiftlang
https://forums.swift.org
...
```

Notice the variety of forms of links that appear on the page. Later we'll see how to resolve them relative to the base URL to make absolute URLs.

The next program uses recursion over the HTML node tree to print the structure of the tree in outline. As it encounters each element, it pushes the element's tag onto a stack, then prints the stack.

```swift
// swiftpl/ch5/outline
func outline(_ stack: [String], _ n: Node) {
    var stack = stack
    if let e = n as? Element {
        stack.append(e.tagName())  // push tag
        print(stack)
    }
    for child in n.getChildNodes() {
        outline(stack, child)
    }
}
```

Note one subtlety: although `outline` "pushes" an element on `stack`, there is no corresponding pop. When `outline` calls itself recursively, the callee receives a copy of `stack`. Thanks to copy-on-write, that copy is cheap, and since arrays are values, the callee's append can't modify the caller's array, so when the function returns, the caller's `stack` is as it was before the call.

Here's the outline of a typical page, again edited for brevity:

```
$ fetch https://www.swift.org | outline
["#root"]
["#root", "html"]
["#root", "html", "head"]
["#root", "html", "head", "meta"]
["#root", "html", "head", "title"]
["#root", "html", "body"]
["#root", "html", "body", "header"]
["#root", "html", "body", "header", "nav"]
...
```

As you can see by experimenting with `outline`, most HTML documents can be processed with only a few levels of recursion, but it's not hard to construct pathological web pages that require extremely deep recursion.

Many programming language implementations use a fixed-size function call stack. Swift is one of them: each thread has a stack of fixed size, typically 8 MB on the main thread and 512 KB to 1 MB on secondary threads and in tasks. Unlike Go, whose stacks grow dynamically, deep recursion in Swift can overflow the stack, which crashes the program. For data whose depth is controlled by an untrusted source, such as HTML from the web, it's prudent either to limit the depth or to replace the recursion with an explicit stack in an array.

**Exercise 5.1:** Change `findlinks` to traverse the `n.getChildNodes()` array using a loop with an explicit stack instead of recursion.

**Exercise 5.2:** Write a function to populate a dictionary that maps element names (`p`, `div`, `span`, and so on) to the number of elements with that name in an HTML document tree.

**Exercise 5.3:** Write a function to print the contents of all text nodes in an HTML document tree. Do not descend into `<script>` or `<style>` elements, since their contents are not visible in a web browser.

**Exercise 5.4:** Extend the `visit` function so that it extracts other kinds of links from the document, such as images, scripts, and style sheets.

## 5.3. Multiple Return Values

A function can return more than one result by returning a tuple. We've seen many examples of functions from the standard library that return two values, such as `quotientAndRemainder(dividingBy:)`, and `URLSession.data(from:)`, which returns the data and the response.

The program below is a variation of `findlinks` that makes the HTTP request itself so that we no longer need to run `fetch`. Because the HTTP and parsing operations can fail, `findLinks` is declared `throws`.

```swift
// swiftpl/ch5/findlinks2
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif
import SwiftSoup

for url in CommandLine.arguments.dropFirst() {
    do {
        for link in try await findLinks(url) {
            print(link)
        }
    } catch {
        printError("findlinks2: \(error)")
    }
}

enum FetchError: Error {
    case invalidURL(String)
    case badStatus(url: String, status: Int)
}

/// Performs an HTTP GET request for url, parses the response as HTML,
/// and extracts and returns the links.
func findLinks(_ url: String) async throws -> [String] {
    guard let u = URL(string: url) else {
        throw FetchError.invalidURL(url)
    }
    let (data, response) = try await URLSession.shared.data(from: u)
    if let http = response as? HTTPURLResponse, http.statusCode != 200 {
        throw FetchError.badStatus(url: url, status: http.statusCode)
    }
    let doc = try SwiftSoup.parse(String(decoding: data, as: UTF8.self))
    var links: [String] = []
    visit(doc, appendingLinksTo: &links)
    return links
}
```

There are three throwing operations in `findLinks`: the HTTP request, the parse, and our own check of the status code. Notice how much less code error handling takes than it does in Go. Each `try` expression propagates an error to the caller automatically, so there's no need for `if err != nil { return nil, err }` after each call. Yet the propagation isn't hidden: every point where an error might leave the function is marked with `try`, so you can see the control flow by reading the code.

Errors thrown with `throw` are values, and our own `FetchError` enum carries enough context to produce a useful message. We'll discuss how to design error types in the next section.

Because Swift returns multiple values as a tuple, a function's results can be given labels, which serve as documentation:

```swift
func countWordsAndImages(_ n: Node) -> (words: Int, images: Int) { ... }

let counts = countWordsAndImages(doc)
print(counts.words, counts.images)
let (words, images) = countWordsAndImages(doc)
```

Swift has no "bare return" of named results as Go does. A tuple result is always returned with an explicit `return (words, images)`.

**Exercise 5.5:** Implement `countWordsAndImages`. (Section 4.3 shows how to split text into words.)

**Exercise 5.6:** In the `heatmap` program of Section 3.2.1, move the computation of a cell's `(x, y)` coordinates into a function that returns a labeled tuple `(x: Double, y: Double)`, and use the labels at the call site.

## 5.4. Errors

Some functions always succeed at their task. For example, `hasPrefix` and `"abc".count` are well defined for all possible inputs and can't fail, barring catastrophic and unpredictable scenarios like running out of memory, where the symptom is far from the cause and from which there's little hope of recovery.

Other functions always succeed so long as their preconditions are met. For example, `Int(exactly: x)` succeeds whenever `x` has an integral value, and an array subscript succeeds whenever the index is in bounds. Violating such a precondition is a *programming error*, a bug in the caller, and Swift's response to a programming error is to *trap*, as we'll discuss in Section 5.9.

For many other functions, even in a well-written program, success is not assured because it depends on factors beyond the programmer's control. Any function that does I/O, for example, must confront the possibility of error, and only a naïve programmer believes a simple read or write cannot fail. Indeed, it's when the most reliable operations fail unexpectedly that we most need to know why.

Swift has two mechanisms for these *expected* failures.

When a failure has only one possible cause, and the caller needs no further information, the function returns an *optional*, with `nil` meaning failure. `Int("42")` returns `Int?`, since the only way it can fail is if the string isn't a valid number; a dictionary lookup returns an optional, since the only way it can fail is if the key is absent.

More often, and especially for I/O, failure may have a variety of causes for which the caller will need an explanation. In such cases, the function is declared `throws`, and it reports failure by *throwing an error*:

```swift
func readConfig(path: String) throws -> Config
```

An error is any value whose type conforms to the `Error` protocol. `Error` has no requirements, so any enum, struct, or class can be an error. Enums are the most common choice, since they enumerate the ways something can fail, and their associated values can carry details:

```swift
enum ConfigError: Error {
    case missing(path: String)
    case malformed(path: String, line: Int, reason: String)
}
```

Swift's error handling looks superficially like exceptions in Java or Python, but it's quite different in practice. Errors are ordinary return values in disguise; throwing an error is as cheap as returning a value, with no stack unwinding tables or expensive stack trace capture. A function must say in its declaration that it can throw, so callers know; every call to a throwing function must be marked with `try`, so readers know; and a throwing call must be handled (by `do`-`catch`) or propagated (by a `throws` function), so the compiler knows. Errors can't escape silently.

### 5.4.1. Error-Handling Strategies

When a function call throws, it's the caller's responsibility to check it and take appropriate action. Depending on the situation, there may be a number of possibilities. Let's take a look at five of them.

First, and most common, is to *propagate* the error, so that a failure in a subroutine becomes a failure of the calling routine. We saw examples of this in the `findLinks` function. A plain `try` in a throwing function propagates any error to the caller unchanged:

```swift
let (data, response) = try await URLSession.shared.data(from: u)
```

In contrast, if the call to `SwiftSoup.parse` fails, `findLinks` doesn't return the parser's error directly, because it lacks two crucial pieces of information: that the error occurred in the parser, and the URL of the document that was being parsed. In this case, it's better to catch the error and throw a new one that includes both pieces of information as well as the underlying error:

```swift
struct ParseFailure: Error, CustomStringConvertible {
    var url: String
    var underlying: any Error
    var description: String { "parsing \(url) as HTML: \(underlying)" }
}

let doc: Document
do {
    doc = try SwiftSoup.parse(html)
} catch {
    throw ParseFailure(url: url, underlying: error)
}
```

Because error messages are frequently chained together, message strings should not be capitalized and newlines should be avoided. The resulting errors may be long, but they will be self-contained when found by tools like `grep`.

When designing error messages, be deliberate, so that each one is a meaningful description of the problem with sufficient and relevant detail, and be consistent, so that errors returned by the same function or by a group of functions in the same module are similar in form and can be dealt with in the same way.

In general, the call `f(x)` is responsible for reporting the attempted operation `f` and the argument value `x` as they relate to the context of the error. The caller is responsible for adding further information that it has but the call `f(x)` does not, such as the URL in the call to the HTML parser above.

Let's move on to the second strategy for handling errors. For errors that represent transient or unpredictable problems, it may make sense to *retry* the failed operation, possibly with a delay between tries, and perhaps with a limit on the number of attempts or the time spent trying before giving up entirely.

```swift
// swiftpl/ch5/wait
/// Attempts to contact the server of a URL. It tries for one minute
/// using exponential back-off. It throws an error if all attempts fail.
func waitForServer(_ url: URL) async throws {
    let timeout = Duration.seconds(60)
    let deadline = ContinuousClock.now + timeout
    var tries = 0
    while ContinuousClock.now < deadline {
        do {
            var request = URLRequest(url: url)
            request.httpMethod = "HEAD"
            _ = try await URLSession.shared.data(for: request)
            return  // success
        } catch {
            print("server not responding (\(error.localizedDescription)); retrying...")
            try await Task.sleep(for: .seconds(1 << tries))  // exponential back-off
            tries += 1
        }
    }
    throw WaitError.timedOut(url: url, after: timeout)
}

enum WaitError: Error {
    case timedOut(url: URL, after: Duration)
}
```

`Task.sleep(for:)` suspends the current task for the given duration without blocking a thread. It's declared `throws` because it ends early, throwing a `CancellationError`, if the task is cancelled, a topic for Section 8.9.

Third, if progress is impossible, the caller can print the error and stop the program gracefully, but this course of action should generally be reserved for the main package of a program. Library functions should usually propagate errors to the caller, unless the error is a sign of an internal inconsistency, that is, a bug.

```swift
// (In main.swift.)
do {
    try await waitForServer(url)
} catch {
    printError("Site is down: \(error)")
    exit(1)
}
```

If a `main.swift` simply lets an error propagate out of its top-level code, or out of a `static func main() throws`, the program prints the error and exits with a non-zero status. That's fine for quick tools, but the message is not always friendly.

Fourth, in some cases, it's sufficient just to log the error and then continue, perhaps with reduced functionality. Swift's standard logging API on servers is the `swift-log` package, whose `Logger` type supports levels, metadata, and pluggable back ends:

```swift
import Logging
let logger = Logger(label: "crawler")

do {
    try ping()
} catch {
    logger.warning("ping failed; networking disabled", metadata: ["error": "\(error)"])
}
```

And fifth and finally, in rare cases we can safely ignore an error entirely. The `try?` operator converts a throwing call into an optional, `nil` if it threw, which is a concise way to say "I don't care why it failed":

```swift
let tempDir = FileManager.default.temporaryDirectory
// ...use temporary directory...
try? FileManager.default.removeItem(at: tempDir)  // ignore errors; the OS cleans up eventually
```

The call to `removeItem` could fail, but the program ignores it because the operating system periodically cleans out the temporary directory. In this case, discarding the error was intentional, but the program logic would be the same had we forgotten to deal with it. Get into the habit of considering errors after every function call, and when you deliberately ignore one, document your intention clearly.

There's also `try!`, which asserts that a call will not throw and traps if it does. Use it only when an error would indicate a bug, such as parsing a regular expression from a string literal in your own source code.

Error handling in Swift has a particular rhythm. After checking an error, failure is usually dealt with before success. If failure causes the function to return, the logic for success is not indented within an `else` block but follows at the outer level. `guard` is designed for this style. Functions tend to exhibit a common structure, with a series of initial checks to reject errors, followed by the substance of the function at the end, minimally indented.

### 5.4.2. Catching Specific Errors

A `catch` clause can contain a *pattern* that selects which errors it handles. Clauses are tried in order, and a `catch` with no pattern handles everything else, binding the error to the constant `error`:

```swift
do {
    let config = try readConfig(path: path)
    run(config)
} catch ConfigError.missing(let path) {
    print("no config at \(path); using defaults")
    run(.default)
} catch let error as ConfigError {
    printError("bad config: \(error)")
    exit(2)
} catch {
    printError("unexpected error: \(error)")
    exit(1)
}
```

The first clause matches one specific enum case and binds its associated value. The second matches any `ConfigError`, with `let error as ConfigError` binding the error with its concrete type. The last clause catches everything else.

By default, a `throws` function can throw any type of error, and inside a `catch` the error has type `any Error`. Swift 6 added *typed throws*, which lets a function declare the one type of error it can throw:

```swift
func parseConfig(_ text: String) throws(ConfigError) -> Config { ... }

do {
    let config = try parseConfig(text)
} catch {
    // error has type ConfigError here, not any Error
    switch error {
    case .missing(let path): ...
    case .malformed(let path, let line, let reason): ...
    }
}
```

Typed throws are useful in low-level code and in embedded Swift, where they avoid boxing the error, and for internal functions whose failures form a closed set. For most public APIs, though, untyped `throws` is still recommended, since it leaves the library free to add new kinds of errors without breaking clients.

To give an error a human-readable message, conform it to `CustomStringConvertible`, as `ParseFailure` did above. On Apple platforms and in Foundation generally, conforming to `LocalizedError` and implementing `errorDescription` controls what `error.localizedDescription` returns.

### 5.4.3. End of File

Usually, the variety of errors that a function may throw is interesting to the end user but not to the intervening program logic. On occasion, however, a program must take different actions depending on the kind of error that has occurred. Consider an attempt to read *n* bytes of data from a file. If *n* is chosen to be the length of the file, any error represents a failure. On the other hand, if the caller repeatedly tries to read fixed-size chunks until the file is exhausted, the caller must respond differently to an end-of-file condition than it does to all other errors.

Go's I/O libraries signal end of file with a distinguished error value, `io.EOF`. Swift APIs usually make end of input part of the ordinary return value instead, typically with an optional: `readLine()` returns `nil` at end of input, and the async iterator of a file's `bytes` or `lines` returns `nil` from `next()`. An actual error, like a disk failure, is thrown. So "no more data" and "something went wrong" are distinguished by the type system rather than by comparing error values:

```swift
while let line = readLine() {
    // ...use line...
}
// end of input reached
```

This is a general principle: if a condition is an expected outcome of normal operation, make it part of the result type, using an optional or an enum; reserve thrown errors for things that went wrong.

## 5.5. Function Values

Functions are *first-class values* in Swift: like other values, function values have types, and they may be assigned to variables or passed to or returned from functions. A function value may be called like any other function. For example:

```swift
func square(_ n: Int) -> Int { n * n }
func negative(_ n: Int) -> Int { -n }
func product(_ m: Int, _ n: Int) -> Int { m * n }

var f = square
print(f(3))  // "9"

f = negative
print(f(3))  // "-3"
print(type(of: f))  // "(Int) -> Int"

f = product  // compile error: can't assign (Int, Int) -> Int to (Int) -> Int
```

Function values are not `Equatable`, so they can't be compared with each other or used as keys in a dictionary.

Function values let us parameterize our functions over not just data, but behavior too. The standard library contains many examples. The *higher-order* functions `map`, `filter`, `reduce`, `sorted(by:)`, `first(where:)`, `contains(where:)`, `allSatisfy`, and `compactMap` all take a function value as an argument and apply it to the elements of a sequence:

```swift
func add1(_ c: Character) -> Character {
    guard let a = c.asciiValue else { return c }
    return Character(Unicode.Scalar(a &+ 1))
}

print(String("HAL-9000".map(add1)))  // "IBM.:111"
print(String("VMS".map(add1)))  // "WNT"
```

`map` applies a function to each element of a sequence and collects the results in an array. Here the sequence is the characters of a string, and we convert the resulting `[Character]` back to a `String`.

Notice that we passed `add1` by name, without calling it; the expression `add1` is a function value. Methods are function values too, and initializers can be referred to as `Type.init`: `["1", "2", "x"].compactMap(Int.init)` produces `[1, 2]`, calling `Int("1")`, `Int("2")`, and `Int("x")`, and dropping the `nil`.

The `findlinks` function from Section 5.2 uses a helper function, `visit`, to visit all the nodes in an HTML document and apply an action to each one. Using a function value, we can separate the logic for tree traversal from the logic for the action to be applied to each node, letting us reuse the traversal with different actions.

```swift
// swiftpl/ch5/outline2
/// Calls the functions pre(x) and post(x) for each node x in the tree
/// rooted at n. Both functions are optional. pre is called before the
/// children are visited (preorder) and post is called after (postorder).
func forEachNode(_ n: Node, pre: ((Node) -> Void)? = nil, post: ((Node) -> Void)? = nil) {
    pre?(n)
    for child in n.getChildNodes() {
        forEachNode(child, pre: pre, post: post)
    }
    post?(n)
}
```

The `forEachNode` function accepts two function arguments, one to call before a node's children are visited and one to call after. This arrangement gives the caller a great deal of flexibility. Each parameter has type `((Node) -> Void)?`, an optional function, with a default of `nil`, and `pre?(n)` calls the function only if it's present.

The functions `startElement` and `endElement` print the start and end tags of an HTML element like `<b>...</b>`:

```swift
var depth = 0

func startElement(_ n: Node) {
    if let e = n as? Element {
        print(String(repeating: " ", count: depth * 2) + "<\(e.tagName())>")
        depth += 1
    }
}

func endElement(_ n: Node) {
    if let e = n as? Element {
        depth -= 1
        print(String(repeating: " ", count: depth * 2) + "</\(e.tagName())>")
    }
}
```

The functions also indent the output using another `String` initializer, `String(repeating:count:)`. If we call `forEachNode` on an HTML document, like this:

```swift
forEachNode(doc, pre: startElement, post: endElement)
```

we get a more elaborate variation on the output of our earlier `outline` program:

```
$ fetch https://www.swift.org | outline2
<#root>
  <html>
    <head>
      <meta>
      </meta>
      <title>
      </title>
      ...
    </head>
    <body>
    ...
```

(As written, `depth` is a top-level variable modified by two functions. That's acceptable in a small, single-threaded program, but in Swift 6 language mode the compiler will insist that a global `var` be isolated to an actor, typically `@MainActor`, if it is used from functions declared outside `main.swift`. Closures, the subject of the next section, offer a neater solution.)

**Exercise 5.7:** Develop `startElement` and `endElement` into a general HTML pretty-printer. Print comment nodes, text nodes, and the attributes of each element (`<a href='...'>`). Use short forms like `<img/>` instead of `<img></img>` when an element has no children.

**Exercise 5.8:** Modify `forEachNode` so that the `pre` and `post` functions return a `Bool` result indicating whether to continue the traversal. Use it to write a function `elementByID(_ doc: Node, id: String) -> Element?` that finds the first HTML element with the specified `id` attribute. The function should stop the traversal as soon as a match is found.

**Exercise 5.9:** Write a function `expand(_ s: String, _ f: (String) -> String) -> String` that replaces each substring `"$foo"` within `s` by the text returned by `f("foo")`.

## 5.6. Closures

Named functions can be declared only at the module level or nested inside other functions. But Swift also has *closure expressions*, unnamed functions that may be written anywhere a function value is needed. A closure expression is enclosed in braces, with its parameters and result type, if any, before the keyword `in`:

```swift
let doubled = [1, 2, 3].map({ (x: Int) -> Int in return x * 2 })
```

That's the long form, and you'll rarely see it, because Swift lets you omit nearly everything that can be inferred. The parameter and result types are inferred from context; a single-expression body needs no `return`; parameters may be referred to by position as `$0`, `$1`, and so on, without being declared; and if the closure is the last argument, it may be written as a *trailing closure* after the parentheses, which may then be omitted if empty. So the line above is usually written:

```swift
let doubled = [1, 2, 3].map { $0 * 2 }
```

Here's a sampler of the forms you'll encounter:

```swift
names.sorted(by: { a, b in a.count < b.count })
names.sorted { $0.count < $1.count }
names.sorted(by: <)  // an operator is a function too
names.map(\.count)  // a key path can stand in for a closure
```

More importantly, closures defined in this way have access to the entire lexical environment, so the inner function can refer to variables from the enclosing function, as this example shows:

```swift
// swiftpl/ch5/squares
/// Returns a function that returns the next square number each time it is called.
func squares() -> () -> Int {
    var x = 0
    return {
        x += 1
        return x * x
    }
}

let f = squares()
print(f())  // "1"
print(f())  // "4"
print(f())  // "9"
print(f())  // "16"
```

The function `squares` returns another function, of type `() -> Int`. A call to `squares` creates a local variable `x` and returns a closure that, each time it is called, increments `x` and returns its square. A second call to `squares` would create a second variable `x` and return a new closure which increments that variable.

The `squares` example demonstrates that function values are not just code but can have state. The closure *captures* the variable `x` by reference, so it can update it. These hidden variable references are why closures are reference types, and why function values are not comparable. Here again we see an example where the lifetime of a variable is not determined by its scope: the variable `x` exists after `squares` has returned within `main`, even though `x` is hidden inside `f`.

### 5.6.1. Escaping Closures

Swift distinguishes two kinds of closure parameters. A closure passed to `map` or `sorted(by:)` is called during the call and never used afterward; it is *non-escaping*. A closure returned from a function, as in `squares`, or stored in a property, or passed to a function that saves it for later, *escapes* the call that received it. Function parameters are non-escaping by default. A parameter that will escape must be annotated `@escaping`:

```swift
final class Button {
    var onClick: (() -> Void)?

    func setAction(_ action: @escaping () -> Void) {
        onClick = action  // stored for later, so it escapes
    }
}
```

The distinction lets the compiler allocate non-escaping closures and their captured variables on the stack, and it tells the reader something important: a non-escaping closure can't create a reference cycle or outlive the objects it uses.

Escaping closures that capture `self`, by contrast, are the most common cause of reference cycles in Swift. If an object stores a closure that refers to the object strongly, neither will ever be freed. A *capture list* breaks the cycle by capturing `self` weakly:

```swift
button.setAction { [weak self] in
    guard let self else { return }
    self.save()
}
```

Inside an escaping closure, Swift requires `self.` to be written explicitly (or `self` to appear in the capture list) when a class's members are used, precisely so that the capture is visible. Structs and enums don't need this, since value types can't form cycles.

### 5.6.2. Example: Topological Sort

Consider the problem of computing a sequence of computer science courses that satisfies the prerequisite requirements of each one. The prerequisites are given in the `prereqs` table below, which is a mapping from each course to the list of courses that must be completed before it.

```swift
// swiftpl/ch5/toposort
/// prereqs maps computer science courses to their prerequisites.
let prereqs: [String: [String]] = [
    "algorithms": ["data structures"],
    "calculus": ["linear algebra"],
    "compilers": [
        "data structures",
        "formal languages",
        "computer organization",
    ],
    "data structures": ["discrete math"],
    "databases": ["data structures"],
    "discrete math": ["intro to programming"],
    "formal languages": ["discrete math"],
    "networks": ["operating systems"],
    "operating systems": ["data structures", "computer organization"],
    "programming languages": ["data structures", "computer organization"],
]
```

This kind of problem is known as topological sorting. Conceptually, the prerequisite information forms a directed graph with a node for each course and edges from each course to the courses that it depends on. The graph is acyclic: there is no path from a course that leads back to itself. We can compute a valid sequence using depth-first search through the graph with the code below:

```swift
for (i, course) in topoSort(prereqs).enumerated() {
    print("\(i + 1):\t\(course)")
}

func topoSort(_ m: [String: [String]]) -> [String] {
    var order: [String] = []
    var seen = Set<String>()

    func visitAll(_ items: [String]) {
        for item in items where !seen.contains(item) {
            seen.insert(item)
            visitAll(m[item] ?? [])
            order.append(item)
        }
    }

    visitAll(m.keys.sorted())
    return order
}
```

The inner function `visitAll` is a *nested function*, declared inside `topoSort`. Like a closure, a nested function can refer to the local variables of its enclosing function (here `order`, `seen`, and `m`), and unlike a closure expression assigned to a variable, it can call itself recursively without any special arrangement. (In Go, a recursive anonymous function must be declared in two steps, first the variable and then the function; Swift's nested functions avoid that.)

In the `topoSort` function, the keys of the `prereqs` dictionary are sorted before being passed to `visitAll`. The output of the program, shown below, is therefore deterministic, an often-overlooked property, even though the input is a dictionary whose iteration order is random.

```
1:  intro to programming
2:  discrete math
3:  data structures
4:  algorithms
5:  linear algebra
6:  calculus
7:  formal languages
8:  computer organization
9:  compilers
10: databases
11: operating systems
12: networks
13: programming languages
```

**Exercise 5.10:** Rewrite `topoSort` to use dictionaries instead of arrays and eliminate the initial sort. Verify that the results, though nondeterministic, are valid topological orderings.

**Exercise 5.11:** The instructor of the linear algebra course decides that calculus is now a prerequisite. Extend the `topoSort` function to report cycles.

### 5.6.3. Example: Web Crawler

Let's now build the crawler. First we need a function that extracts the links from a page, resolving each relative link against the page's URL. We'll put it in a small library module, `Links`, so that we can reuse it in Chapter 8.

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

Instead of our own recursive `visit`, this version uses SwiftSoup's `select` method, which finds elements matching a CSS selector, here every `<a>` that has an `href` attribute. Rather than append the raw `href` attribute value to the result, it resolves it as a URL relative to the base URL of the document, `response.url`, which may differ from the requested URL if there were redirects. The resulting link is in absolute form, suitable for fetching.

Finding links is the core of a web crawler, but we need a way to explore the web graph. Here's a function that does a *breadth-first* traversal: it calls a function value `f` for each item in a worklist, and any items returned by `f` are added to the worklist. Each item is visited at most once. The function is generic over the item type `T`, which must be `Hashable` so that items can be kept in a set:

```swift
// swiftpl/ch5/findlinks3
/// Calls f for each item in the worklist.
/// Any items returned by f are added to the worklist.
/// f is called at most once for each item.
func breadthFirst<T: Hashable>(_ worklist: [T], _ f: (T) async -> [T]) async {
    var worklist = worklist
    var seen = Set<T>()
    while !worklist.isEmpty {
        let items = worklist
        worklist = []
        for item in items where seen.insert(item).inserted {
            worklist += await f(item)
        }
    }
}
```

The argument `f` is a function value, here of type `(T) async -> [T]`, an asynchronous function. Because `breadthFirst` calls it with `await`, `breadthFirst` must itself be `async`.

In our crawler, items are URLs. The `crawl` function we'll supply to `breadthFirst` prints the URL, extracts its links, and returns them so that they too are visited:

```swift
func crawl(_ url: String) async -> [String] {
    print(url)
    do {
        return try await extract(url)
    } catch {
        printError("\(error)")
        return []
    }
}

// Crawl the web breadth-first, starting from the command-line arguments.
await breadthFirst(Array(CommandLine.arguments.dropFirst()), crawl)
```

Let's crawl the web starting from `https://www.swift.org`. Here are some of the resulting links:

```
$ swift run findlinks3 https://www.swift.org
https://www.swift.org
https://www.swift.org/install/
https://www.swift.org/documentation/
https://www.swift.org/community/
https://www.swift.org/packages/
https://www.swift.org/blog/
https://github.com/swiftlang
https://forums.swift.org/
...
```

The process ends when all reachable web pages have been crawled or the memory of the computer is exhausted. Since it fetches one page at a time, it's slow; in Chapter 8 we'll make it concurrent.

**Exercise 5.12:** Modify `crawl` to make local copies of the pages it finds, creating directories as necessary. Don't make copies of pages that come from a different domain.

**Exercise 5.13:** Write `breadthFirst` without the `items` copy, using an index into the worklist instead. Which is clearer?

## 5.7. Variadic Functions

A *variadic function* is one that can be called with varying numbers of arguments. The most familiar example is `print`, which takes any number of items.

To declare a variadic function, follow the type of the parameter with an ellipsis, `...`, which indicates that the function may be called with any number of arguments of this type.

```swift
// swiftpl/ch5/sum
func sum(_ vals: Int...) -> Int {
    var total = 0
    for val in vals {
        total += val
    }
    return total
}
```

The `sum` function above returns the sum of zero or more `Int` arguments. Within the body of the function, the type of `vals` is an `[Int]` array. When `sum` is called, any number of values may be provided for its `vals` parameter.

```swift
print(sum())  // "0"
print(sum(3))  // "3"
print(sum(1, 2, 3, 4))  // "10"
```

Implicitly, the caller allocates an array, copies the arguments into it, and passes the array to the function. Unlike Go, however, Swift has no syntax for passing an existing array to a variadic parameter. There's no equivalent of Go's `sum(values...)`. The idiom is to write the function to take an array, and provide the variadic version as a convenience that calls it:

```swift
func sum(_ vals: [Int]) -> Int {
    vals.reduce(0, +)
}

func sum(_ vals: Int...) -> Int {
    sum(vals)
}
```

`reduce` combines all the elements of a sequence using a function, here the `+` operator, starting from an initial value. Unlike Go, Swift allows several variadic parameters in one function, as long as each is followed by a labeled parameter, and a variadic parameter need not be last.

Variadic functions are often used for string formatting. The `errorf` function below constructs a formatted message with a line number at the beginning. The suffix `f` is a widely followed naming convention for variadic functions that accept a `printf`-style format string.

```swift
import Foundation

func errorf(line: Int, _ format: String, _ args: any CVarArg...) {
    let message = String(format: format, arguments: args)
    printError("Line \(line): \(message)")
}

errorf(line: 12, "expected %d arguments, got %d", 2, 3)
// "Line 12: expected 2 arguments, got 3"
```

`CVarArg` is a protocol for values that can be passed to C variadic functions, and `String(format:arguments:)` takes an array of them. In idiomatic Swift, string interpolation is preferred to format strings: a function would usually take a `String` and let the caller build it with `\(...)`.

**Exercise 5.14:** Write variadic functions `max` and `min`, analogous to `sum`. What should these functions do when called with no arguments? Write variants that require at least one argument.

**Exercise 5.15:** Write a variadic version of `joined(separator:)` as a free function.

## 5.8. Deferred Statements

Our `findLinks` examples used the output of `URLSession` as the input to the HTML parser. Because `URLSession.data(from:)` reads the entire response into memory, it takes care of releasing the connection's resources for us. But many resources are not managed automatically: a file that we open must be closed, a lock that we acquire must be released, a temporary file that we create must be removed. And they must be released on every path out of the function, including early returns and thrown errors.

Swift's `defer` statement solves this. A `defer` statement contains a block of code that is executed when control leaves the *enclosing scope*, by any means: reaching the end, `return`, `break`, `continue`, or a thrown error. If there are several `defer` statements in a scope, they're executed in the reverse of the order in which they were reached.

A `defer` statement is often used with paired operations like open and close, connect and disconnect, or lock and unlock to ensure that resources are released in all cases, no matter how complex the control flow. The right place for a `defer` statement that releases a resource is immediately after the resource has been successfully acquired. In the `title` function below, a single `defer` closes the file handle:

```swift
// swiftpl/ch5/title
import Foundation
import SwiftSoup

func title(path: String) throws -> String {
    guard let file = FileHandle(forReadingAtPath: path) else {
        throw CocoaError(.fileNoSuchFile)
    }
    defer { try? file.close() }

    let data = try file.readToEnd() ?? Data()
    let doc = try SwiftSoup.parse(String(decoding: data, as: UTF8.self))
    let title = try doc.title()
    if title.isEmpty {
        throw TitleError.noTitle(path)
    }
    return title
}

enum TitleError: Error {
    case noTitle(String)
}
```

Whether the function returns normally or throws from any of the `try` expressions, the file is closed.

There's one important difference from Go: a Swift `defer` runs at the end of its *scope*, not the end of the function. A `defer` inside a loop body runs at the end of each iteration, which is usually exactly what you want:

```swift
for path in paths {
    guard let file = FileHandle(forReadingAtPath: path) else { continue }
    defer { try? file.close() }  // closed at the end of each iteration
    // ...process file...
}
```

In Go, the equivalent code is a well-known trap, since all the deferred closes wait until the function returns and the program can run out of file descriptors.

The `defer` statement can also be used to pair "on entry" and "on exit" actions when debugging a complex function. The `bigSlowOperation` function below calls `trace` immediately, which does the "on entry" action and then returns a function value that, when called, does the corresponding "on exit" action. By deferring a call to the returned function in this way, we can instrument the entry point and all exit points of a function in a single statement and even pass values, like the start time, between the two actions.

```swift
// swiftpl/ch5/trace
func bigSlowOperation() async throws {
    let exit = trace("bigSlowOperation")
    defer { exit() }
    // ...lots of work...
    try await Task.sleep(for: .seconds(10))  // simulate slow operation by sleeping
}

func trace(_ msg: String) -> () -> Void {
    let start = ContinuousClock.now
    print("enter \(msg)")
    return { print("exit \(msg) (\(ContinuousClock.now - start))") }
}
```

Each time `bigSlowOperation` is called, it logs its entry and exit and the elapsed time between them:

```
$ swift run trace
enter bigSlowOperation
exit bigSlowOperation (10.002113 seconds)
```

A `defer` block can read the function's local variables but cannot change the function's result, since Swift has no named results. It also can't contain `return`, `break`, or `throw` statements that would transfer control out of it; a `defer` exists to clean up, not to change the outcome.

As with any resource cleanup, it's worth asking whether `defer` is needed at all. For classes, `deinit` can release resources deterministically when the last reference goes away, and noncopyable types (marked `~Copyable`, like the `Mutex` of Section 9.2) let the compiler guarantee that a resource has exactly one owner and is consumed exactly once. Many APIs also provide a *with* function that manages a resource for the duration of a closure, like `Mutex.withLock` in Section 1.7, which is often the cleanest option of all.

**Exercise 5.16:** Write a function that copies one file to another, using `defer` to close both files. Make sure that if the copy fails partway, the partially written destination file is removed.

## 5.9. Traps

Swift's type system catches many mistakes at compile time, but others, like an out-of-bounds array access, a force-unwrap of `nil`, an arithmetic overflow, or a failed integer conversion, can be detected only at run time. When Swift detects one of these mistakes, it *traps*: the program stops immediately and the operating system reports a crash.

A trap is not an exception. There is no stack unwinding, no running of `defer` blocks or `deinit`s, and no way to catch it. The program prints a diagnostic message and a backtrace, and terminates:

```
$ swift run crash
Fatal error: Index out of range
Crash: crash at /home/user/crash/Sources/crash/main.swift:3:10
...
Thread 0 crashed:
0 Swift/ContiguousArrayBuffer.swift:691 in _checkSubscript(_:wasNativeTypeChecked:)
1 crash at /home/user/crash/Sources/crash/main.swift:3:10
...
```

(On Linux, the Swift runtime's built-in backtracer prints a symbolicated backtrace like this one. On macOS, you'll find a crash report in Console, and in either case a debugger will stop at the faulting instruction.)

This design reflects a philosophy: a trap signals a *programming error*, a situation the programmer believed impossible. Once such a belief has been proven false, the program's state is suspect, and continuing could corrupt data or open security holes. The safest thing to do is to stop. Errors that the program *expects* and can recover from should be thrown, as described in Section 5.4.

You can also trap deliberately. Swift provides a family of functions for checking assumptions, which differ in when they're active:

```swift
precondition(index >= 0, "negative index")
assert(cache.isConsistent, "cache corrupted")
fatalError("unreachable")
```

`precondition(condition, message)` checks a condition that callers must satisfy, and traps if it's false. It's checked in both debug and release builds. Use it to validate arguments in a public API, in the same circumstances where a Go function would `panic`.

`assert(condition, message)` checks an internal invariant. It's checked only in debug builds and compiled out of release builds, so it can be used for expensive consistency checks.

`fatalError(message)` traps unconditionally, in all builds. Its return type is `Never`, a type with no values, so the compiler knows that control never continues past a call to it. That makes it useful in positions where a value is needed:

```swift
switch s {
case "Spades": ...
case "Hearts": ...
case "Diamonds": ...
case "Clubs": ...
default:
    fatalError("invalid suit \(s)")  // Joker?
}
```

`preconditionFailure` and `assertionFailure` are unconditional versions of the first two.

It's good practice to assert that the preconditions of a function hold, but this can easily be done to excess. Unless you can provide a more informative error message or detect an error sooner, there's no point checking something the runtime will check for you, like array bounds:

```swift
func reset(_ x: inout [Int], _ i: Int) {
    precondition(i < x.count, "index out of range")  // unnecessary!
    x[i] = 0
}
```

A function that receives input from outside the program, such as user input, network data, or file contents, must never trap on bad input. That's a misuse of a mechanism meant for bugs; use thrown errors and optionals instead.

The compiler's `-Ounchecked` optimization mode removes most precondition checks, including overflow and some bounds checks, turning violations into undefined behavior. It's rarely appropriate; the checks are cheap, and the safety they provide is the reason for using Swift in the first place.

**Exercise 5.17:** Write a program that traps by each of these means: an array index out of range, a force-unwrapped `nil`, an integer overflow, a failed `Int8(_:)` conversion, a `precondition`, and `fatalError`. Compare the messages, and observe which checks remain in a release build (`-c release`).

## 5.10. Recovering from Failure

In Go, `recover` can stop a panic from terminating the program, and some Go programs use it to make a web server survive a bug in one handler, or to simplify error handling deep inside a recursive parser. Swift has no equivalent. Once a Swift program traps, it's over.

What do Swift programs do instead?

For the recursive-parser case, where a Go programmer might panic deep inside and recover at the top, Swift's ordinary `throws` is the answer. Thrown errors are cheap to propagate through many levels of recursion, and each level needs only a `try`. Here's the skeleton of a recursive-descent parser that reports the first syntax error it finds:

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

Each parsing function is declared with typed throws, `throws(SyntaxError)`, since a syntax error is the only thing that can go wrong. When the innermost function finds a problem, it throws, and the error propagates up through all the `try`s to the caller of `parseExpr`. We'll build a complete expression parser in Section 7.9.

Sometimes it's useful to capture the outcome of a throwing operation as a value, to store it or pass it somewhere else, rather than handle it immediately. The standard library's `Result` type is an enum with two cases, `.success(value)` and `.failure(error)`:

```swift
let result = Result { try parse(text) }  // captures the value or the error

switch result {
case .success(let expr):
    print(expr)
case .failure(let error):
    print("error: \(error)")
}

let expr = try result.get()  // converts back to a throwing call
```

Before Swift had `async`/`await`, `Result` was widely used to deliver the outcome of asynchronous operations to completion handlers. Today it's used less, but it's still handy for collecting results from many operations, some of which may have failed.

For the web server case, where a Go server recovers from a panic in one handler and continues serving others, Swift's answer is *process isolation*. A server built on Hummingbird or Vapor doesn't try to survive a trap; instead, the server runs under a supervisor such as `systemd`, a container orchestrator, or a load balancer with several replicas, which restarts it promptly. Because Swift's ARC-based memory management has no garbage-collection pauses, and because Swift programs start quickly, a restart is cheap. Meanwhile, the bug that caused the trap has been caught by a check that, in a memory-unsafe language, might instead have silently corrupted memory.

This is a real trade-off, and it's been debated in the Swift community. But the overall message is clear: in Swift, expected failures are thrown errors or optionals, which can always be handled, and bugs are traps, which can't be. Choosing the right mechanism for each situation is a large part of writing robust Swift code.

**Exercise 5.18:** Write a function that never uses a `return` statement but still returns a non-`Void` value. (Hint: consider a `switch` expression, a `Never`-returning function, and an implicit-return body.)

**Exercise 5.19:** Use `Result` to modify `fetchall` from Section 1.6 so that the task group's children return `Result<(Int, Duration), any Error>` and the parent counts successes and failures separately.
