# 1. Tutorial

This chapter is a tour of the basic parts of Swift. We hope to give you enough information and examples to get you off the ground and doing useful things as quickly as possible. The examples here, and indeed in the whole book, are aimed at tasks you might have to do in the real world. In this chapter we'll try to give you a taste of the variety of programs you can write in Swift, ranging from simple file processing and a bit of graphics to concurrent Internet clients and servers. We certainly won't explain everything in the first chapter, but studying such programs in a new language can be an effective way to get started.

When you're learning a new language, there's a natural tendency to write code as you would have written it in a language you already know. Be aware of this bias as you learn Swift and try to avoid it. We've tried to illustrate and explain how to write good Swift, so use the code here as a guide when you're writing your own.

## 1.1. Hello, World

We'll start with the now-traditional "hello, world" example, which appears at the beginning of *The C Programming Language*, published in 1978. C is one of the indirect influences on Swift, and "hello, world" illustrates a number of central ideas.

```swift
// swiftpl/ch1/helloworld
print("Hello, 世界")
```

That's the whole program. Swift is a compiled language. The Swift toolchain converts a source program and the things it depends on into instructions in the native machine language of a computer. These tools are accessed through a single command called `swift` that has a number of subcommands. The simplest is to hand a single source file to `swift` itself, which compiles it and runs the result immediately:

```
$ swift helloworld.swift
```

Not surprisingly, this prints

```
Hello, 世界
```

Swift natively handles Unicode, so it can process text in all the world's languages.

If the program is more than a one-shot experiment, you'll want to compile it once and save the compiled result for later use. That is done with `swiftc`:

```
$ swiftc helloworld.swift
```

This produces an executable binary file called `helloworld` that can be run any time without further processing:

```
$ ./helloworld
Hello, 世界
```

Real programs are made of more than one file and depend on libraries, so in practice you will usually use the Swift Package Manager, which we'll meet properly in Chapter 10. The commands

```
$ mkdir helloworld
$ cd helloworld
$ swift package init --type executable
```

create a directory structure like this:

```
helloworld/
    Package.swift
    Sources/
        helloworld/
            main.swift
```

`Package.swift` is the *package manifest*. It describes, in Swift, what the package builds and what it depends on. `Sources/helloworld/main.swift` holds the program. Replace its contents with our one-line program and then run

```
$ swift run
Building for debugging...
Build complete!
Hello, 世界
```

Each example in this book is labeled, like `swiftpl/ch1/helloworld` above, with the name of a package you can create to hold it.

Let's now talk about the program itself. Swift code is organized into *modules*, which are similar to libraries or packages in other languages. A module consists of one or more `.swift` files in a single directory. Every file in a module can see every declaration in every other file of that module without any special effort; there are no header files and no import statements between files of the same module.

The standard library, whose module is called `Swift`, is imported automatically into every file. It provides the basic types (`Int`, `Double`, `String`, `Array`, `Dictionary`, and many more) and a few hundred functions, among them `print`, which writes the textual representation of its arguments to the standard output followed by a newline. Other modules must be imported explicitly. The one you'll import most often is `Foundation`, which provides files, dates, URLs, networking, formatting, and much else. `Foundation` is part of every Swift toolchain on every platform.

A file called `main.swift` is special. Its statements are *top-level code* that runs, in order, when the program starts. That's why our hello program needs no `main` function: the whole file is the main function. In other files of a module, top-level statements are not allowed; only declarations may appear. An alternative, used in larger programs, is to mark one type with the `@main` attribute and give it a `static func main()` method:

```swift
@main
struct HelloWorld {
    static func main() {
        print("Hello, 世界")
    }
}
```

Both forms do the same thing. We'll use `main.swift` for most of the programs in this book because it is shorter.

Swift does not require semicolons at the ends of statements or declarations. A newline ends a statement unless the line is obviously incomplete, for example if it ends in a binary operator or an opening parenthesis. You may use a semicolon to separate two statements on the same line, but this is rare.

Swift takes a strong position on code formatting, though less dogmatic than Go's: the language does not require one style, but the toolchain ships with `swift format`, which rewrites code into a standard layout. We recommend running it routinely, as we have done for all the examples in this book.

```
$ swift format --in-place Sources/helloworld/main.swift
```

Many editors, through the SourceKit-LSP language server that comes with Swift, can run the formatter whenever you save a file.

## 1.2. Command-Line Arguments

Most programs process some input to produce some output; that's pretty much the definition of computing. But how does a program get input data on which to operate? Some programs generate their own data, but more often, input comes from an external source: a file, a network connection, the output of another program, a user at a keyboard, command-line arguments, or the like. The next few examples will discuss some of these alternatives, starting with command-line arguments.

The standard library provides an enum called `CommandLine` whose static property `arguments` holds the command-line arguments as an array of strings. An array is an ordered collection of values of one type, and `CommandLine.arguments` has the type `[String]`, which is shorthand for `Array<String>`. Individual elements can be accessed as `arguments[i]`, and the number of elements as `arguments.count`. Indexes start at zero.

The first element, `CommandLine.arguments[0]`, is the name of the command itself; the other elements are the arguments that were presented to the program when it started execution.

Here's an implementation of the Unix `echo` command, which prints its command-line arguments on a single line. It uses a few more features than it really needs, to give us something to talk about.

```swift
// swiftpl/ch1/echo1
// Echo1 prints its command-line arguments.

let arguments = CommandLine.arguments
var s = ""
var sep = ""
var i = 1
while i < arguments.count {
    s += sep + arguments[i]
    sep = " "
    i += 1
}
print(s)
```

Comments begin with `//`. All text from a `//` to the end of the line is commentary for programmers and is ignored by the compiler. Swift also has C-style `/* ... */` block comments, which, unlike C's, can be nested. By convention, we describe each program in a comment immediately preceding its code.

The program declares four names with `let` and `var`. A `let` declaration introduces a *constant*: a name bound to a value that can never change. A `var` declaration introduces a *variable*, whose value may be changed by assignment. The Swift compiler notices when a `var` is never modified and warns you to make it a `let`, and you should. Using `let` by default makes programs easier to reason about, because when you read `let x = ...` you know that `x` will have that value for its whole lifetime.

None of these declarations says what *type* its name has. Swift is statically typed (every variable and constant has a type that is fixed at compile time) but the compiler infers the type from the initial value. `s` and `sep` are `String`s because they are initialized with string literals, and `i` is an `Int` because it is initialized with an integer literal. You can always write the type explicitly if you want to:

```swift
var s: String = ""
var i: Int = 1
```

Swift provides the usual arithmetic and logical operators. When applied to strings, however, the `+` operator *concatenates* the values, so the expression

```swift
sep + arguments[i]
```

represents the concatenation of the strings `sep` and `arguments[i]`. The statement

```swift
s += sep + arguments[i]
```

is an *assignment operator* statement that concatenates the old value of `s` with `sep` and `arguments[i]` and assigns it back to `s`; it is equivalent to

```swift
s = s + sep + arguments[i]
```

The statement `i += 1` adds 1 to `i`. Notice that Swift has no `++` or `--` operators. They were removed in Swift 3 because they encouraged writing code with subtle side effects, and `+= 1` says the same thing plainly.

The `while` loop repeats its body as long as its condition is true. Swift's condition expressions need no parentheses, but the braces around the body are mandatory, even for a one-statement body. The condition must have type `Bool`; Swift will never treat a number or a pointer as true or false.

Swift has no C-style three-part `for` loop either. Instead, its `for`-`in` loop iterates over any *sequence*: the elements of an array, the characters of a string, the keys and values of a dictionary, the lines of a file, or a *range* of numbers. Here's a second version of `echo` that uses one:

```swift
// swiftpl/ch1/echo2
// Echo2 prints its command-line arguments.

var s = ""
var sep = ""
for arg in CommandLine.arguments.dropFirst() {
    s += sep + arg
    sep = " "
}
print(s)
```

In each iteration of the loop, `arg` is bound to the next element of the sequence. Notice that the loop variable is a fresh constant in each iteration; you cannot assign to `arg` inside the loop.

The method `dropFirst()` returns everything but the first element of the array, so we skip the program's name without index arithmetic. It doesn't copy the elements. It returns an `ArraySlice`, a view onto part of the original array. We'll see much more about slices in Section 4.2.

If you do want a loop over integers, iterate over a range. The *half-open range* operator `a..<b` produces the integers from `a` up to but not including `b`, and the *closed range* `a...b` includes `b`:

```swift
for i in 1..<arguments.count {
    print(arguments[i])
}
```

A loop that should run forever is written `while true { ... }`, and it may be exited with a `break` or `return` statement. `continue` skips the rest of the current iteration and starts the next one.

Each time around the loop in `echo2`, the statement `s += sep + arg` builds a new string. In many languages that would make the loop quadratic. Swift's strings are mutable values, and `+=` on a `String` variable usually appends in place, growing the string's storage geometrically, so the cost is amortized linear. Still, it's simpler and clearer to use the standard library's `joined(separator:)` method, which works on any sequence of strings:

```swift
// swiftpl/ch1/echo3
print(CommandLine.arguments.dropFirst().joined(separator: " "))
```

Finally, if we don't care about format but just want to see the values, perhaps for debugging, we can let `print` format the array for us:

```swift
print(CommandLine.arguments.dropFirst())
```

The output of this statement is like what we would get from `echo3`, except that the elements are shown as string literals, surrounded by brackets:

```
$ ./echo3 one two three
["one", "two", "three"]
```

`print` accepts any number of arguments and separates them with spaces, so `print("a", 1, true)` prints `a 1 true`. Its optional `separator:` and `terminator:` arguments change the space and the trailing newline: `print(x, terminator: "")` prints `x` with no newline after it.

**Exercise 1.1:** Modify the `echo` program to also print `CommandLine.arguments[0]`, the name of the command that invoked it.

**Exercise 1.2:** Modify the `echo` program to print the index and value of each of its arguments, one per line. (Hint: look up the `enumerated()` method of sequences.)

**Exercise 1.3:** Experiment to measure the difference in running time between our potentially inefficient version and the one that uses `joined(separator:)`. (Section 1.6 shows how to use `ContinuousClock` to time code, and Section 11.4 shows how to write benchmarks for systematic performance evaluation.)

## 1.3. Finding Duplicate Lines

Programs for file copying, printing, searching, sorting, counting, and the like all have a similar structure: a loop over the input, some computation on each element, and generation of output on the fly or at the end. We'll show three variants of a program called `dup`; it is partly inspired by the Unix `uniq` command, which looks for adjacent duplicate lines. The structures and modules used are models that can be easily adapted.

The first version of `dup` prints each line that appears more than once in the standard input, preceded by its count. This program introduces the `if` statement, the `Dictionary` type, and optionals.

```swift
// swiftpl/ch1/dup1
// Dup1 prints the text of each line that appears more than
// once in the standard input, preceded by its count.

var counts: [String: Int] = [:]
while let line = readLine() {
    counts[line, default: 0] += 1
}
for (line, n) in counts {
    if n > 1 {
        print("\(n)\t\(line)")
    }
}
```

As with `while`, an `if` statement needs no parentheses around its condition, but braces are required for the body. There may be an optional `else` part that is executed if the condition is false.

A *dictionary* holds a set of key/value pairs and provides constant-time operations to store, retrieve, or test for an item in the set. The key may be of any type whose values can be compared with `==` and hashed, and strings are the most common example; the value may be of any type at all. In this example, the keys are strings and the values are `Int`s. The type is written `[String: Int]`, which is shorthand for `Dictionary<String, Int>`, and `[:]` is an empty dictionary literal.

Each time `dup` reads a line of input, the line is used as a key into the dictionary and the corresponding value is incremented. The statement `counts[line, default: 0] += 1` is equivalent to these statements:

```swift
let old = counts[line] ?? 0
counts[line] = old + 1
```

Looking up a key in a dictionary gives back a value that might not be there, and Swift encodes this possibility in the type. The expression `counts[line]` does not have type `Int`; it has type `Int?`, pronounced "optional `Int`", whose values are either an `Int` or `nil`, which means "no value." Swift never lets you use an optional as if it were the value inside. You must first *unwrap* it, and the language provides several ways to do so. The `??` operator supplies a default for the `nil` case. The `default:` subscript of `Dictionary` does the same, and also lets you modify the stored value in place, which is why it's the idiomatic way to count.

Optionals are how Swift eliminates the null pointer errors that plague other languages. A variable of type `String` always holds a string; only a variable of type `String?` can be `nil`, and the compiler makes you deal with that case before you get at the string.

That brings us to `readLine()`, which reads one line from the standard input and returns it without its trailing newline. At end of input, there is no line to return, so `readLine` returns `nil`. Its result type is therefore `String?`. The loop

```swift
while let line = readLine() {
    ...
}
```

uses *optional binding*: each time around, it calls `readLine()`, and if the result is not `nil`, it binds the unwrapped string to the new constant `line` and executes the body. When `readLine` returns `nil`, the loop ends. The same form works with `if`:

```swift
if let n = counts["hello"] {
    print("hello appeared \(n) times")
} else {
    print("no hellos")
}
```

To print the results, we use another `for`-`in` loop, this time over the dictionary. Each iteration produces a key/value pair, which we destructure into the two constants `line` and `n`. The order of dictionary iteration is not specified, and in practice it is effectively random, varying from one run to another. This is intentional, since it prevents programs from relying on an ordering that the implementation doesn't guarantee. If you want the output in order, sort it first, for example with `for (line, n) in counts.sorted(by: { $0.key < $1.key })`. We'll explain that syntax in Section 5.6.

The output uses *string interpolation*: inside a string literal, `\(` *expression* `)` is replaced by the textual representation of the expression's value. In `"\(n)\t\(line)"` the count is followed by a tab character, written `\t`, and then the line. Interpolation is the everyday way to format output in Swift, and it works with values of any type. We'll see in Section 4.6 how to extend it.

To test `dup1`, you can type lines and then signal end of input with Control-D. Or, more usefully, redirect a file into it:

```
$ swift run dup1 < input.txt
```

The next version of `dup` can read from the standard input or handle a list of file names, reading each file in its entirety and then splitting it into lines.

```swift
// swiftpl/ch1/dup2
// Dup2 prints the count and text of lines that appear more than once
// in the input. It reads from stdin or from a list of named files.
import Foundation

var counts: [String: Int] = [:]
let files = CommandLine.arguments.dropFirst()
if files.isEmpty {
    while let line = readLine() {
        counts[line, default: 0] += 1
    }
} else {
    for path in files {
        do {
            let text = try String(contentsOfFile: path, encoding: .utf8)
            countLines(text, into: &counts)
        } catch {
            printError("dup2: \(path): \(error.localizedDescription)")
        }
    }
}
for (line, n) in counts where n > 1 {
    print("\(n)\t\(line)")
}

func countLines(_ text: String, into counts: inout [String: Int]) {
    for line in text.split(separator: "\n") {
        counts[String(line), default: 0] += 1
    }
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

Reading a file can fail. The file might not exist, or we might not have permission to read it, or its contents might not be valid UTF-8. The initializer `String(contentsOfFile:encoding:)` reports failure by *throwing an error*. Its declaration is marked `throws`, and Swift requires every call to a throwing function to be marked with `try`, so that anyone reading the code can see which operations might fail. A call marked with `try` must appear inside a `do` block that has a `catch` clause, or inside a function that is itself marked `throws`. If the call throws, control transfers immediately to the `catch` clause, where the error is available in a constant named `error`.

In our program, if the file can't be read, we print a message on the standard error stream and continue with the next file. `print` writes to the standard output, so we wrote a small helper, `printError`, that writes to the standard error stream using Foundation's `FileHandle`. `FileHandle.write` takes raw bytes as a `Data` value, so we convert the string's UTF-8 encoding to `Data` first. We will see more sophisticated ways of handling errors in Section 5.4.

Notice that the two function declarations appear after the top-level code that calls them. In `main.swift`, functions may be declared anywhere and used throughout the file; it's only top-level variables that must be declared before use.

The function `countLines` needs to update the caller's dictionary. Function parameters in Swift are constants, and dictionaries, like all Swift collections, are *values*: passing `counts` to a function normally gives the function its own logical copy, and changes made to that copy wouldn't be visible to the caller. To let a function modify its caller's variable, declare the parameter `inout` and pass the variable with a leading `&`. When the function returns, the modified value is written back to the caller's variable. This is the closest thing Swift has to passing a pointer, and it's the subject of Section 2.3.

This version of `countLines` uses `split(separator:)`, which breaks a string into pieces wherever the separator character appears. The pieces have type `Substring`, which, like `ArraySlice`, shares storage with the original string. A substring is cheap to make but keeps the whole original string alive, so we convert each piece to a `String` before storing it in the dictionary. Note also that `split` omits empty pieces by default, so blank lines are not counted. That's a reasonable choice for `dup`, but it's not what `dup1` does, which is the subject of the first exercise below.

Finally, in the loop that prints the results, a `where` clause filters the elements of the sequence, so the body executes only for pairs whose count exceeds one.

The version of `dup2` that reads the standard input processes it line by line, while the version that reads files loads each one entirely into memory and then splits it. That's fine for files of modest size. For enormous inputs, a streaming approach is better, and Foundation's `FileHandle` and the asynchronous `lines` sequences we'll see in Chapter 8 make one possible. In practice, `String(contentsOfFile:encoding:)` and its sibling `Data(contentsOf:)` are what you'll reach for first.

**Exercise 1.4:** Modify `countLines` so that it counts blank lines, as `dup1` does. (Hint: `split` has an `omittingEmptySubsequences:` parameter. What happens to the last line of a file that ends with a newline?)

**Exercise 1.5:** Modify `dup2` to print the names of all files in which each duplicated line occurs.

## 1.4. Drawing Lissajous Figures

The next program demonstrates basic use of Swift's numeric types and math functions, along with a little more string formatting. It generates an animated image of a Lissajous figure, the kind of curve that early science fiction films used to make their oscilloscopes look busy. The figure is traced by two sine waves, one on the x axis and one on the y axis, and changing the relative frequency and phase of the two waves produces a hypnotic variety of shapes.

Swift's standard library has no image-encoding facilities of its own, but there's no need for them: we'll produce an SVG file, a text format for vector graphics that every web browser can display. Each frame of the animation is a polyline through a few thousand points, and SVG's `<animate>` element cycles through the frames.

```swift
// swiftpl/ch1/lissajous
// Lissajous generates an animated SVG of random Lissajous figures.
import Foundation

let cycles = 5.0  // number of complete x oscillator revolutions
let res = 0.01  // angular resolution
let size = 100.0  // image canvas covers [-size..+size]
let nframes = 64  // number of animation frames
let delay = 0.08  // delay between frames in seconds

func lissajous() -> String {
    let freq = Double.random(in: 0..<3)  // relative frequency of y oscillator
    var phase = 0.0  // phase difference
    var frames: [String] = []
    for _ in 0..<nframes {
        var points: [String] = []
        for t in stride(from: 0, to: cycles * 2 * .pi, by: res) {
            let x = sin(t)
            let y = sin(t * freq + phase)
            points.append(String(format: "%.1f,%.1f", size + x * size, size - y * size))
        }
        frames.append(points.joined(separator: " "))
        phase += 0.1
    }
    let width = Int(2 * size)
    let duration = Double(nframes) * delay
    return """
        <svg xmlns="http://www.w3.org/2000/svg" width="\(width)" height="\(width)" \
        style="background: black">
        <polyline fill="none" stroke="lime" stroke-width="0.4" points="\(frames[0])">
        <animate attributeName="points" dur="\(duration)s" repeatCount="indefinite"
            calcMode="discrete" values="\(frames.joined(separator: ";"))"/>
        </polyline>
        </svg>
        """
}

print(lissajous())
```

After compiling, run it like this to make an image you can open in a browser:

```
$ swift run lissajous > out.svg
```

The program begins with four constants, `cycles`, `res`, `size`, and `delay`, all of type `Double` because they're initialized with floating-point literals, and one, `nframes`, of type `Int`. Swift never converts between numeric types implicitly. You can't multiply an `Int` by a `Double` without saying which one to convert, so the expression `Double(nframes) * delay` converts `nframes` to a `Double` first. This rule can seem fussy at first, but it rules out a whole family of bugs involving truncation, sign extension, and overflow. Section 3.1 discusses numeric conversions in detail.

The function `lissajous` has two nested loops. The outer loop runs for 64 iterations, each producing a single frame of the animation. The variable `_` in `for _ in 0..<nframes` says that we don't need the loop's index; we just want the loop to run that many times. The inner loop iterates over a `stride`, a sequence of floating-point values from 0 up to (but not including) `cycles * 2 * .pi`, separated by `res`. The constant `.pi` is short for `Double.pi`; Swift infers the type from context, so the leading dot is enough.

The two oscillators produce an x and a y between -1 and +1, which we scale to the size of the canvas. SVG's y axis points downwards, so we subtract to flip the image. Each point is formatted by `String(format:)`, a Foundation initializer that uses the same `%` formatting conventions as C's `printf`; `%.1f` formats a floating-point number with one digit after the decimal point.

The `random(in:)` method generates a random value within a range; here it picks a relative frequency between 0 and 3. Each time the program runs, the frequency is different, and so is the figure. The phase increases by 0.1 with each frame, which makes the figure appear to rotate.

The SVG document is assembled with a *multiline string literal*, which begins and ends with three double-quote marks on their own lines. The indentation of the closing `"""` determines how much leading whitespace is removed from each line, so the literal can be indented to match the surrounding code. A backslash at the end of a line joins it with the next line, and interpolation works as in ordinary strings. We'll return to strings in Section 3.5.

The `<animate>` element changes the polyline's `points` attribute to each value in its semicolon-separated `values` list in turn, spending `delay` seconds on each; `calcMode="discrete"` makes it jump from frame to frame instead of trying to interpolate between them.

**Exercise 1.6:** Modify the Lissajous program to use several stroke colors, chosen at random, by adding a `<animate attributeName="stroke">` element with a palette of values.

**Exercise 1.7:** Modify the program to accept the relative frequency on the command line instead of choosing it at random. Use `Double(someString)`, which returns `nil` if the string is not a valid number.

## 1.5. Fetching a URL

For many applications, access to information from the Internet is as important as access to the local file system. Foundation provides a networking client, `URLSession`, for sending and receiving data over HTTP and HTTPS.

To illustrate the minimum necessary to retrieve information over HTTP, here's a simple program called `fetch` that fetches the content of each specified URL and prints it as uninterpreted text; it's inspired by the invaluable utility `curl`. Obviously one would usually do more with such data, but this shows the basic idea. We will use this program frequently in the book.

```swift
// swiftpl/ch1/fetch
// Fetch prints the content found at each specified URL.
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

for arg in CommandLine.arguments.dropFirst() {
    guard let url = URL(string: arg) else {
        printError("fetch: invalid URL: \(arg)")
        exit(1)
    }
    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        FileHandle.standardOutput.write(data)
    } catch {
        printError("fetch: \(arg): \(error.localizedDescription)")
        exit(1)
    }
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

On Apple platforms, `URLSession` is part of Foundation itself, but on Linux and Windows it lives in a separate module called `FoundationNetworking`, so that programs that don't need networking don't have to link it. The `#if canImport(...)` block is a *conditional compilation directive*: it imports `FoundationNetworking` only on platforms where that module exists. You'll see this incantation at the top of most cross-platform Swift networking code.

The `guard` statement is a kind of inside-out `if`. It states a condition that must be true for execution to continue. If the condition is false, the `else` block runs, and that block is required to leave the current scope, here by calling `exit`, which terminates the process with the given status code. Combined with optional binding, `guard let` unwraps a value and makes it available for *the rest of the scope*, not just inside a block, which keeps the main line of the code unindented. `URL(string:)` returns `nil` if the string isn't a syntactically valid URL, so `url` has type `URL?` until the `guard` unwraps it.

Fetching a resource over the network takes time, perhaps a long time, and during that time the program could be doing other useful work. So `URLSession`'s `data(from:)` method is *asynchronous*. Its declaration is marked `async`, and a call to it must be marked with `await`. The `await` marks a *suspension point*: the current task may pause there while the request is in flight, freeing its thread to run other work, and resume when the response arrives. We'll see how to take advantage of this in the next section. For now, notice that top-level code in `main.swift` may use `await` directly, which makes the whole program asynchronous.

The `data(from:)` method returns a *tuple* of two values: the response body as `Data`, and a `URLResponse` object describing the response, which we ignore by binding it to `_`. It can also throw, so the call is marked `try await` and appears inside a `do` block. We write the body to the standard output using `FileHandle`, without trying to interpret it as text.

```
$ swift build
$ .build/debug/fetch https://swift.org
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Swift.org - Welcome to Swift.org</title>
...
```

If the request fails, `fetch` reports the failure instead:

```
$ .build/debug/fetch https://bad.swift.org
fetch: https://bad.swift.org: A server with the specified hostname could not be found.
```

In either error case, `exit(1)` causes the process to exit with a status code of 1.

**Exercise 1.8:** Modify `fetch` to add the prefix `https://` to each argument URL if it is missing. You might want to use `hasPrefix(_:)`.

**Exercise 1.9:** Modify `fetch` to also print the HTTP status code. The response returned by `data(from:)` is really an `HTTPURLResponse`; you can test for that and get at it with `if let http = response as? HTTPURLResponse`. We'll explain `as?` in Section 7.10.

## 1.6. Fetching URLs Concurrently

One of the most interesting and novel aspects of Swift is its support for concurrent programming. This is a large topic, to which Chapters 8 and 9 are devoted, so for now we'll give you just a taste of Swift's main concurrency mechanisms, *tasks* and *structured concurrency*.

The next program, `fetchall`, does the same fetch of a URL's contents as the previous example, but it fetches many URLs, all concurrently, so that the process will take no longer than the longest fetch rather than the sum of all the fetch times. This version of `fetchall` discards the responses but reports the size and elapsed time for each one:

```swift
// swiftpl/ch1/fetchall
// Fetchall fetches URLs in parallel and reports their times and sizes.
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

let start = ContinuousClock.now
await withTaskGroup(of: String.self) { group in
    for url in CommandLine.arguments.dropFirst() {
        group.addTask {
            await fetch(url)  // start a child task
        }
    }
    for await result in group {
        print(result)  // receive each result as it completes
    }
}
print("\(ContinuousClock.now - start) elapsed")

func fetch(_ url: String) async -> String {
    let start = ContinuousClock.now
    guard let u = URL(string: url) else {
        return "invalid URL: \(url)"
    }
    do {
        let (data, _) = try await URLSession.shared.data(from: u)
        let elapsed = ContinuousClock.now - start
        return "\(elapsed)  \(data.count)  \(url)"
    } catch {
        return "while reading \(url): \(error.localizedDescription)"
    }
}
```

Here's an example:

```
$ swift build
$ .build/debug/fetchall https://swift.org https://forums.swift.org https://swiftpackageindex.com
0.141228 seconds  21342  https://swift.org
0.402871 seconds  86104  https://forums.swift.org
0.735512 seconds  148203  https://swiftpackageindex.com
0.736020 seconds elapsed
```

A *task* is a unit of asynchronous work. Every asynchronous function runs as part of some task, and tasks can run concurrently with one another, spread across a small pool of threads. Our program's top-level code runs in the main task, which creates a *task group* by calling `withTaskGroup`. The argument `of: String.self` says that each child task in the group will produce a `String`. The trailing braces form a *closure*, an anonymous function, that receives the group as its parameter, here named `group`.

For each command-line argument, `group.addTask` starts a new *child task* that calls `fetch` asynchronously. The child tasks run concurrently with each other and with the parent. After starting them all, the parent loops over the group with `for await`, which waits for and receives each child's result in the order the children *finish*, not the order they started. Since every result is printed by the parent, no two tasks write to the standard output at once, and the lines are not interleaved.

The `fetch` function measures its own elapsed time using `ContinuousClock`, a clock that measures elapsed time without being affected by changes to the system's wall-clock time. Subtracting one instant of a clock from another produces a `Duration`, which prints in seconds.

Task groups are *structured*: `withTaskGroup` does not return until every child task has finished. A child task can't outlive the scope that created it, so there's no way to accidentally leave work running in the background, and no need to wait explicitly for the tasks to complete. If the parent is cancelled, the cancellation propagates automatically to all its children. We'll explore what that means in Section 8.9.

Notice also what the compiler is checking here, even though we didn't have to say anything about it. The closure passed to `addTask` will run concurrently with the code that created it, so everything it captures must be safe to share between concurrent tasks. Strings are values, so sharing one is safe. If we had tried to capture a mutable variable, perhaps to accumulate a total, the Swift 6 compiler would have rejected the program with an error about a data race. Swift checks at compile time that concurrent code doesn't share mutable state unsafely, a topic we'll study in Chapter 9.

**Exercise 1.10:** Find a web site that produces a large amount of data. Investigate caching by running `fetchall` twice in succession to see whether the reported time changes much. Do you get the same content each time? Modify `fetchall` to write its output to a file so it can be examined.

**Exercise 1.11:** Try `fetchall` with longer argument lists, such as samples from the top million web sites available on the web. How does the program behave if a web site just doesn't respond? (Section 8.9 describes mechanisms for coping in such cases.)

## 1.7. A Web Server

Swift's server ecosystem makes it easy to write a web server that responds to requests like those made by `fetch`. In this section, we'll show a minimal server that returns the path component of the URL used to access the server. That is, if the request is for `http://localhost:8000/hello`, the response will be `URL.Path = /hello`.

The standard library and Foundation include an HTTP *client* but not an HTTP *server*. For that we'll use Hummingbird, a lightweight open-source web framework built on SwiftNIO, Apple's high-performance networking library. Using it is our first example of depending on another package, so the manifest needs to say so:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "server1",
    platforms: [.macOS(.v15)],
    dependencies: [
        .package(url: "https://github.com/hummingbird-project/hummingbird.git", from: "2.0.0"),
    ],
    targets: [
        .executableTarget(
            name: "server1",
            dependencies: [.product(name: "Hummingbird", package: "hummingbird")]
        ),
    ]
)
```

The first line is a comment, but a meaningful one: it tells the package manager which version of the manifest format to use. The `dependencies` array lists the packages we need and the range of versions we accept (here, any 2.x version), and the target lists the *products* (libraries) it uses from them. The first time you build, `swift build` downloads Hummingbird and its own dependencies and records the exact versions it chose in a file called `Package.resolved`. Chapter 10 has the details.

Here's the server:

```swift
// swiftpl/ch1/server1
// Server1 is a minimal "echo" server.
import Hummingbird

let router = Router()
router.get("/", use: handler)
router.get("**", use: handler)  // any other path

let app = Application(
    router: router,
    configuration: .init(address: .hostname("localhost", port: 8000))
)
try await app.runService()

// handler echoes the Path component of the requested URL.
@Sendable func handler(_ request: Request, _ context: BasicRequestContext) -> String {
    "URL.Path = \(request.uri.path)\n"
}
```

The program is only a handful of lines long because the library does most of the work. A *router* maps request paths to handler functions. We register `handler` for the root path `/` and for `**`, a wildcard that matches any other path. Then we create an application that listens on port 8000 of the local machine and run it. `runService` doesn't return until the server is shut down, for example by Control-C, which it handles gracefully.

A handler receives the request and a *context* (here `BasicRequestContext`, the router's default, which carries per-request information such as a logger) and returns something that can be turned into a response. Here, that's a `String`, which Hummingbird sends as a plain-text body with status `200 OK`. The `@Sendable` attribute says that the function is safe to call from concurrent tasks, which it must be because the server handles many requests at once. The function's body is a single expression, so we can omit the `return` keyword.

We need to start the server in the background. On macOS or Linux, add an ampersand (`&`) to the command:

```
$ swift run server1 &
```

We can then make client requests from the command line:

```
$ .build/debug/fetch http://localhost:8000
URL.Path = /
$ .build/debug/fetch http://localhost:8000/help
URL.Path = /help
```

Alternatively, we can access the server from a web browser.

It's easy to add features to the server. One useful addition is a specific URL that returns a status of some sort. For example, this version does the same echo but also counts the number of requests; a request to the URL `/count` returns the count so far, excluding `/count` requests themselves:

```swift
// swiftpl/ch1/server2
// Server2 is a minimal "echo" and counter server.
import Hummingbird
import Synchronization

let count = Mutex(0)

let router = Router()
router.get("/count", use: counter)
router.get("/", use: handler)
router.get("**", use: handler)

let app = Application(
    router: router,
    configuration: .init(address: .hostname("localhost", port: 8000))
)
try await app.runService()

// handler echoes the Path component of the requested URL.
@Sendable func handler(_ request: Request, _ context: BasicRequestContext) -> String {
    count.withLock { $0 += 1 }
    return "URL.Path = \(request.uri.path)\n"
}

// counter echoes the number of calls so far.
@Sendable func counter(_ request: Request, _ context: BasicRequestContext) -> String {
    let n = count.withLock { $0 }
    return "Count \(n)\n"
}
```

The server has two handlers, and the request URL determines which one is called: a request for `/count` invokes `counter` and all others invoke `handler`. Behind the scenes, the server runs each incoming request in a separate task so that it can serve multiple requests simultaneously. However, if two concurrent requests try to update `count` at the same time, it might not be incremented consistently; the program would have a serious bug called a *data race* (Section 9.1). Swift 6 won't compile a program that has one: if `count` were a plain global `Int` variable, the compiler would reject both functions because they access shared mutable state from concurrent code.

To make the counter safe, we store it in a `Mutex`, from the standard library's `Synchronization` module. The only way to get at the value inside a mutex is to call `withLock`, passing a closure that receives the protected value as an `inout` parameter. The mutex ensures that at most one task at a time is running such a closure. In the closure, `$0` refers to the closure's first parameter, so `{ $0 += 1 }` increments the count and `{ $0 }` returns its value. We'll look at mutexes and their alternative, *actors*, in Chapter 9.

As a richer example, the handler function can report on the headers and query parameters that it receives, which is useful for inspecting and debugging requests:

```swift
// swiftpl/ch1/server3
// handler echoes the HTTP request.
@Sendable func handler(_ request: Request, _ context: BasicRequestContext) -> String {
    var out = "\(request.method) \(request.uri)\n"
    for field in request.headers {
        out += "Header[\(field.name)] = \(field.value)\n"
    }
    for (key, value) in request.uri.queryParameters {
        out += "Query[\(key)] = \(value)\n"
    }
    return out
}
```

This builds up the response in the string variable `out`, starting with the request method and URI and appending a line for each header and query parameter:

```
GET /?q=query HTTP/1.1
Header[Host] = localhost:8000
Header[User-Agent] = curl/8.7.1
Header[Accept] = */*
Query[q] = query
```

We've now seen three handlers, each a function. Swift functions are values like any other: they can be stored in variables, passed to functions, and returned from them. That's why we can register `handler` with the router by passing its name. We'll come back to this in Section 5.5.

The server can serve images too. Let's combine the web server with the `lissajous` function so that animated images are drawn not on the standard output, but for a web client. Copy the `lissajous` function and its constants into a second file, `lissajous.swift`, in the same module (remember that all files of a module can see each other's declarations, so no import is necessary), and add this route to the server:

```swift
router.get("/lissajous") { _, _ in
    Response(
        status: .ok,
        headers: [.contentType: "image/svg+xml"],
        body: ResponseBody(byteBuffer: ByteBuffer(string: lissajous()))
    )
}
```

This time the handler is written as a *trailing closure*. Its two parameters (the request and context) are unused, so we name them both `_`. Because a string body would be sent as plain text, we construct a `Response` explicitly to set its `Content-Type` header, and convert the string to a `ByteBuffer`, the efficient byte container used by SwiftNIO. When you visit `http://localhost:8000/lissajous` in a browser, you'll see an animation.

**Exercise 1.12:** Modify the Lissajous server to read parameter values from the URL. For example, you might arrange it so that a URL like `http://localhost:8000/lissajous?cycles=20` sets the number of cycles to 20 instead of the default 5. Use `request.uri.queryParameters["cycles"]` to get the parameter value, and convert it to a number with `Double(String(...))`.

## 1.8. Loose Ends

There is a lot more to Swift than we've covered in this quick introduction. Here are some topics we've barely touched upon or omitted entirely, with just enough discussion that they will be familiar when they make brief appearances before the full treatment.

**Control flow:** We covered the two fundamental control-flow statements, `if` and `for`, plus `while` and `guard`, but not the `switch` statement, which is a multi-way branch. Here's a small example:

```swift
switch coinflip() {
case "heads":
    heads += 1
case "tails":
    tails += 1
default:
    print("landed on edge!")
}
```

The result of calling `coinflip` is compared to the value of each case. Cases are evaluated from top to bottom, so the first matching one is executed. Unlike C, there is no implicit fall-through from one case into the next, so no `break` is needed (though there is a rarely used `fallthrough` statement that overrides this behavior). A `switch` must be *exhaustive*: every possible value must be handled by some case, which is why this one needs a `default`. Cases may list several values separated by commas, match ranges, bind variables, and add conditions:

```swift
func signum(_ x: Int) -> Int {
    switch x {
    case let n where n > 0:
        return +1
    case 0:
        return 0
    default:
        return -1
    }
}
```

The real power of `switch` appears when it's used with enumerations, which we'll describe in a moment, and with tuples and other patterns. Chapter 7 shows several examples.

**Enumerations:** An `enum` declares a type with a fixed set of possible values. Swift's enums are much richer than C's, because each case may carry *associated values*:

```swift
enum Shape {
    case circle(radius: Double)
    case rectangle(width: Double, height: Double)
}

func area(_ s: Shape) -> Double {
    switch s {
    case .circle(let r):
        return .pi * r * r
    case .rectangle(let w, let h):
        return w * h
    }
}
```

No `default` is needed, because the compiler knows that `circle` and `rectangle` are the only possibilities. If someone later adds a `.triangle` case, every exhaustive `switch` on `Shape` will fail to compile until it is updated. That property makes enums one of the most useful tools for modeling data in Swift. `Optional` itself is just an enum with two cases, `.none` and `.some(value)`.

**Named types:** A `struct` declaration gives a name to a group of related values. For example, the declaration

```swift
struct Point {
    var x: Int
    var y: Int
}

var p = Point(x: 1, y: 2)
p.x += 1
```

defines a type called `Point` with two *stored properties*, and the compiler automatically writes an *initializer*, `Point(x:y:)`, that sets them. Structs are values: assigning `p` to another variable copies it. Structs are covered in Chapter 4.

**Reference types:** Swift has no general pointers in ordinary code. Instead, it distinguishes *value types*, such as structs, enums, and tuples, from *reference types*, chiefly classes and actors. A variable of class type holds a reference to an object, and assigning it to another variable makes the two refer to the same object. Objects are allocated on the heap and freed automatically when the last reference to them disappears, a scheme called *automatic reference counting*. For low-level work, Swift does have pointers, with names like `UnsafeMutablePointer`, which we'll meet in Chapter 13.

**Methods and protocols:** A method is a function associated with a named type, and Swift is unusual in that methods may be attached to almost any type, including structs and enums, and even existing types like `Int`, by an *extension*. A *protocol* is an abstract type that names a set of required methods and properties; a type *conforms* to a protocol by declaring that it does and providing those members. `Hashable`, `Comparable`, `Sequence`, and `Error` are some of the standard library's protocols. Methods are covered in Chapter 6 and protocols in Chapter 7.

**Modules and packages:** Swift comes with an extensive standard library, Foundation, and a growing ecosystem of open-source packages. Programming is often more about using existing packages than about writing original code of one's own. Throughout the book, we will point out a couple of dozen of the most important ones, but there are many more. The Swift Package Index (`swiftpackageindex.com`) is the place to search.

Documentation for the standard library and Foundation is available at `developer.apple.com/documentation` and from `swift.org`. You can also see the declaration of any symbol directly from your editor by asking the language server to "jump to definition," and in the REPL with `:type lookup`:

```
$ swift repl
  1> :type lookup Array.joined
```

**Comments:** We have already mentioned comments at the beginning of each program. Before the declaration of each function, it's good style to write a comment that specifies its behavior. Documentation comments start with three slashes, `///`, and use Markdown; the editor and the documentation compiler, DocC, display them, and conventions like `- Parameter name:` and `- Returns:` describe the parts:

```swift
/// Returns the number of lines in `text` that are not blank.
///
/// - Parameter text: A string whose lines are separated by `"\n"`.
func countNonBlankLines(_ text: String) -> Int {
    text.split(separator: "\n").count
}
```

Comments that span multiple lines can be written as `/* ... */`, which, unlike in C, may be nested, so you can comment out a block of code that already contains block comments.
