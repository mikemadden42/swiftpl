# 11. Testing

Maurice Wilkes, the designer of EDSAC, the first stored-program computer, had a startling realization in 1949 while climbing the stairs of his laboratory: "the realization came over me with full force that a good part of the remainder of my life was going to be spent in finding the errors in my own programs." Since then, generations of programmers have struggled to separate programs from bugs, and the techniques that have emerged for doing so are many.

The programs we write today are far larger and more complex than those of Wilkes's day, and a great deal of effort has gone into techniques that make that complexity manageable. Two of them stand out for their effectiveness. The first is routine peer review of programs before they are deployed. The second, the subject of this chapter, is testing.

Testing, by which we implicitly mean automated testing, is the practice of writing small programs that check that the code under test (the *production code*) behaves as expected for certain inputs, which are usually either carefully chosen to exercise certain features or randomly chosen to ensure broad coverage.

The field of software testing is enormous. The task of testing occupies all programmers some of the time and some programmers all of the time. The literature on testing includes thousands of printed books and millions of words of blog posts. In every mainstream programming language, there are dozens of software packages intended for test construction, some with a great deal of theory, and the field seems to attract more than a few elaborate frameworks and fashions, each with its own vocabulary and ritual.

Swift's approach to testing might seem rather low-tech by comparison. It relies on one command, `swift test`, and a small library for writing tests. The library has changed over time. For a decade, the standard tool was *XCTest*, a framework descended from Objective-C's OCUnit, built around test classes and a family of `XCTAssert...` functions. Since Swift 6, the toolchain also includes *Swift Testing*, a newer library designed for the language's modern features: tests are ordinary functions marked with a macro, and a single `#expect` macro replaces the whole assertion family. This chapter teaches Swift Testing, which is the recommended choice for new code, and notes where XCTest differs.

## 11.1. The `swift test` Tool

`swift test` is a test driver for Swift packages. In a package directory, test sources live in test targets, conventionally under `Tests/`, declared in the manifest as `.testTarget`:

```swift
.testTarget(name: "EvalTests", dependencies: ["Eval"])
```

Files in a test target can contain any code, but `swift test` looks for *tests*: declarations marked with `@Test`. It builds the test target, plus the targets it depends on, and then runs every test it finds. Tests run in parallel by default, and any test that fails is reported along with the reasons.

Compared with Go, where a test is a function with a magic name in a `_test.go` file, a Swift test is identified by an attribute, so names are free and tests can live in any file of a test target. And unlike in Go, tests live in a *separate module* from the code under test. The test module imports the production module just like any client would, and so it can see only `public` declarations. To test internal ones, use a *testable import*:

```swift
@testable import Eval
```

`@testable` makes internal declarations visible, but only when the module was compiled with testing enabled, which `swift build` and `swift test` do for debug builds. It's a great convenience, but it can tempt you into testing implementation details; prefer testing the public interface when you can.

Useful options include:

```
$ swift test                         # build and run all tests
$ swift test --filter EvalTests      # run tests whose names match a pattern
$ swift test --skip SlowTests        # run all but the matching tests
$ swift test --parallel              # (XCTest) run test classes in parallel
$ swift test --enable-code-coverage  # collect coverage data
$ swift test --sanitize=thread       # run under the Thread Sanitizer
$ swift test list                    # list the tests without running them
```

## 11.2. `@Test` Functions

A test is a function marked `@Test`. It can be a free function, or a method of a *suite*, a type whose members are tests. It passes if it returns normally, and fails if any of its expectations fail or it throws an uncaught error. Here's a first test, for the `palindrome` package, a function that reports whether a string reads the same forwards and backwards:

```swift
// swiftpl/ch11/word1/Sources/Word/Word.swift
/// Reports whether s reads the same forward and backward.
/// (Our first attempt.)
public func isPalindrome(_ s: String) -> Bool {
    for i in 0..<s.count / 2 {
        let a = s.index(s.startIndex, offsetBy: i)
        let b = s.index(s.endIndex, offsetBy: -i - 1)
        if s[a] != s[b] {
            return false
        }
    }
    return true
}
```

and its test file:

```swift
// swiftpl/ch11/word1/Tests/WordTests/WordTests.swift
import Testing
import Word

@Test func palindromes() {
    #expect(isPalindrome("detartrated"))
    #expect(isPalindrome("kayak"))
}

@Test func nonPalindrome() {
    #expect(!isPalindrome("palindrome"))
}
```

`#expect` is a macro that takes a boolean expression and records a failure if it's false. Unlike a plain `assert`, a failed expectation doesn't stop the test: the test keeps running, so one run can report several failures. (When a later step makes no sense after an earlier failure, use `try #require(...)`, described below.)

Run the tests:

```
$ swift test
Building for debugging...
Test run started.
Testing Library Version: 6.1
◇ Test palindromes() started.
◇ Test nonPalindrome() started.
✔ Test nonPalindrome() passed after 0.001 seconds.
✔ Test palindromes() passed after 0.001 seconds.
✔ Test run with 2 tests passed after 0.001 seconds.
```

Now suppose a bug report arrives: the function fails on a famous palindrome from 1795 (Napoleon-era wordplay, "A man, a plan, a canal: Panama"), which has punctuation and mixed case, and one in French with accents. We write a test that reproduces it, first, before fixing anything:

```swift
@Test func canalPanama() {
    #expect(isPalindrome("A man, a plan, a canal: Panama"))
}

@Test func frenchPalindrome() {
    #expect(isPalindrome("été"))
}
```

```
✘ Test canalPanama() recorded an issue at WordTests.swift:14:5: Expectation failed: isPalindrome("A man, a plan, a canal: Panama")
✔ Test frenchPalindrome() passed
```

The `#expect` macro's failure message shows the *source text* of the expression, and, for comparisons, the values of the operands: `#expect(a == b)` reports `a == b → 3 == 4` with both sides evaluated. That's the reason there are no `assertEqual` variants: with macros, one construct covers everything and still gives detailed messages.

Writing a test that fails before you fix the bug is good practice. It confirms that the test really catches the bug, and the test remains afterwards as a guard against regression. Now we fix the function to ignore everything that isn't a letter, and to compare letters ignoring case:

```swift
// swiftpl/ch11/word2/Sources/Word/Word.swift
/// Reports whether s reads the same forward and backward,
/// ignoring punctuation, spaces, and case.
public func isPalindrome(_ s: String) -> Bool {
    let letters = s.filter(\.isLetter).lowercased()
    return letters == String(letters.reversed())
}
```

The tests now pass. Notice that the second version is also simpler than the first and allocates less than it might seem: `filter` and `lowercased` each produce a string, and `reversed()` is lazy.

### 11.2.1. Test Names and Traits

A `@Test` can take a display name and *traits* that customize its behavior:

```swift
@Test("Palindromes ignore punctuation and case")
func punctuation() { ... }

@Test(.tags(.slow), .timeLimit(.minutes(1)))
func bigInput() { ... }

@Test(.disabled("waiting on bug #123"))
func knownBad() { ... }

@Test(.enabled(if: ProcessInfo.processInfo.environment["CI"] != nil))
func onlyOnCI() { ... }

@Test(.bug("https://example.com/issues/123"), .tags(.regression))
func issue123() { ... }
```

Tags, declared once with an extension and the `@Tag` macro, group tests across files so that `swift test --filter` and Xcode can select them:

```swift
extension Tag {
    @Tag static var slow: Self
    @Tag static var regression: Self
}
```

In the Go world, `testing.Short()` and build tags serve similar purposes. Swift's traits are declarative and appear in test reports.

### 11.2.2. Table-Driven Tests

The palindrome tests we have written so far are repetitive. The standard way to deal with that in Go is a *table-driven test*: a slice of inputs and expected outputs, and a loop. Swift Testing builds this in. A test can take a parameter, and the `arguments:` argument supplies a collection of values to run it with:

```swift
// swiftpl/ch11/word3/Tests/WordTests/WordTests.swift
import Testing
import Word

@Test(arguments: [
    "",
    "a",
    "aa",
    "ab",
    "kayak",
    "détartrated",
    "Évian?",
    "A man, a plan, a canal: Panama",
    "Evil I did dwell; lewd did I live.",
    "Able was I ere I saw Elba",
    "Et se resservir, ivresse reste.",
    "No 'x' in Nixon",
])
func isPalindromeTest(_ input: String) {
    let letters = input.filter(\.isLetter).lowercased()
    let expected = letters == String(letters.reversed())  // a deliberately naive oracle
    #expect(isPalindrome(input) == expected, "input: \(input)")
}
```

Each argument is run as a separate *test case*, in parallel, and reported individually, so a failure names the input that failed. If a test takes several parameters, pass tuples or use `zip`; with two collections as arguments, Swift Testing runs the *cartesian product*:

```swift
@Test(arguments: [1, 2, 3], ["a", "b"])
func combos(n: Int, s: String) { ... }  // six test cases

@Test(arguments: zip([1, 2, 3], ["one", "two", "three"]))
func pairs(n: Int, name: String) { ... }  // three test cases
```

Here's a more typical table, pairing inputs with expected outputs. This one tests the expression parser of Section 7.9:

```swift
struct Case: Sendable, CustomTestStringConvertible {
    let input: String
    let env: Env
    let want: String
    var testDescription: String { input }
}

@Test(arguments: [
    Case(input: "sqrt(A / pi)", env: ["A": 87616, "pi": .pi], want: "167"),
    Case(input: "pow(x, 3) + pow(y, 3)", env: ["x": 12, "y": 1], want: "1729"),
    Case(input: "pow(x, 3) + pow(y, 3)", env: ["x": 9, "y": 10], want: "1729"),
    Case(input: "5 / 9 * (F - 32)", env: ["F": -40], want: "-40"),
    Case(input: "5 / 9 * (F - 32)", env: ["F": 32], want: "0"),
    Case(input: "5 / 9 * (F - 32)", env: ["F": 212], want: "100"),
])
func evalTable(_ c: Case) throws {
    let expr = try parse(c.input)
    let got = String(format: "%.6g", expr.eval(c.env))
    #expect(got == c.want, "\(c.input) in \(c.env)")
}
```

Because the test function is `throws`, we can use plain `try` on the parser; if the parser throws, the test fails and reports the error. The `testDescription` makes each case readable in the report.

The output shows the results for each case, and since the cases run independently, a failure in one doesn't hide the results of the others, the problem that Go programmers solve with `t.Run` subtests:

```
✘ Test evalTable(_:) with 6 test cases failed after 0.002 seconds with 1 issue.
✘ Test evalTable(_:) recorded an issue with 1 argument c → 5 / 9 * (F - 32): Expectation failed: (got → "55.5556") == (c.want → "56")
```

### 11.2.3. Throwing and Required Expectations

Errors are a normal part of behavior, so we need to test for them. `#expect(throws:)` checks that an operation throws:

```swift
#expect(throws: SyntaxError.self) {
    try parse("x % 2")
}

#expect(throws: ParseError.unexpectedEnd) {  // a specific, Equatable error value
    try parse("1 +")
}

#expect(throws: Never.self) {  // asserts that nothing is thrown
    try parse("1 + 2")
}
```

The macro returns the thrown error (as an optional) so that you can inspect it further:

```swift
let error = #expect(throws: SyntaxError.self) { try parse("1 +") }
#expect(error?.description.contains("unexpected") == true)
```

`try #require(...)` is the stricter form: if the condition is false (or an optional is `nil`), the test *stops* immediately by throwing. It's meant for preconditions of the rest of the test. It also unwraps optionals, which makes it a tidy replacement for Go's `if err != nil { t.Fatal(err) }` pattern:

```swift
@Test func lookup() throws {
    let user = try #require(db.user(named: "ada"))  // fails and stops if nil
    #expect(user.id == 1)  // safe to continue: user is a User, not a User?
}
```

Use `#expect` for ordinary checks, so one run can report many failures, and `#require` when continuing would be pointless or would crash.

### 11.2.4. Suites, Setup, and Teardown

Group related tests in a struct, class, or actor, called a *suite*. Each test method runs on a *fresh instance* of the suite, so state created in `init` is private to the test, and `deinit` (for classes and actors) serves as teardown. This replaces XCTest's `setUp` and `tearDown` and Go's `TestMain`:

```swift
@Suite("Temporary directory tests")
struct FileTests {
    let dir: URL

    init() throws {
        dir = FileManager.default.temporaryDirectory.appending(path: UUID().uuidString)
        try FileManager.default.createDirectory(at: dir, withIntermediateDirectories: true)
    }

    @Test func writesFile() throws {
        let file = dir.appending(path: "a.txt")
        try "hello".write(to: file, atomically: true, encoding: .utf8)
        #expect(try String(contentsOf: file, encoding: .utf8) == "hello")
    }
}
```

Because each test gets its own instance, tests in a suite don't interfere with each other even though they run in parallel. Mark a suite `@Suite(.serialized)` to run its tests one at a time when they share an unavoidable resource such as a database or a fixed port.

### 11.2.5. Testing Asynchronous Code

Tests may be `async`, and may use `await` freely, which makes testing the concurrent programs of Chapters 8 and 9 straightforward:

```swift
@Test func memoComputesOnce() async throws {
    let counter = Mutex(0)
    let memo = Memo<String, Int> { key in
        counter.withLock { $0 += 1 }
        return key.count
    }
    async let a = memo.get("hello")
    async let b = memo.get("hello")
    let results = try await [a, b]
    #expect(results == [5, 5])
    #expect(counter.withLock { $0 } == 1)  // the function ran only once
}
```

Callback-based APIs can be tested with `confirmation`, which checks how many times an event occurs:

```swift
@Test func notifiesTwice() async {
    await confirmation("handler called", expectedCount: 2) { confirm in
        let n = Notifier(handler: { confirm() })
        n.fire()
        n.fire()
    }
}
```

### 11.2.6. Randomized Testing

Table-driven tests are only as good as the examples we think of. A complementary technique is *randomized* testing, which explores a broader range of inputs by constructing them at random. How do we know what output to expect from our random input? There are two strategies. The first is to write an alternative implementation of the function that uses a less efficient but simpler and clearer algorithm, and check that both implementations give the same result. The second is to create input values according to a pattern so that we know what output to expect.

The example below uses the second approach: the `randomPalindrome` function constructs words that are known to be palindromes by building the first half and then reflecting it.

```swift
// swiftpl/ch11/word3
/// Returns a palindrome whose length and contents are derived from the generator.
func randomPalindrome(using rng: inout some RandomNumberGenerator) -> String {
    let n = Int.random(in: 0..<25, using: &rng)  // random length up to 24
    var half: [Character] = []
    for _ in 0..<(n + 1) / 2 {
        // A random letter from the Latin range U+0061...U+024F.
        let scalar = Unicode.Scalar(UInt32.random(in: 0x61...0x24F, using: &rng)) ?? "a"
        half.append(Character(scalar))
    }
    // Reflect the first half; drop the middle letter when the length is odd.
    let tail = n % 2 == 0 ? half.reversed() : half.dropLast().reversed()
    return String(half) + String(tail)
}

@Test func randomPalindromes() {
    var rng = SeededGenerator(seed: 42)  // a deterministic generator
    for _ in 0..<1000 {
        let p = randomPalindrome(using: &rng)
        #expect(isPalindrome(p), "isPalindrome(\(p.debugDescription)) = false")
    }
}
```

Some of the generated characters aren't letters (U+00D7 is a multiplication sign, for example), and `isPalindrome` ignores those, which is fine: a string that is a palindrome stays one when some characters are removed.

Since the program is random, we want to be able to reproduce any failure. That's why we use a *seeded* generator rather than the system's default: the sequence of values is fully determined by the seed, so a failing run can be repeated exactly. Swift's standard library has no seedable generator built in, but the protocol `RandomNumberGenerator` has a single requirement, so a small one is easy to write:

```swift
/// A fast, deterministic generator (SplitMix64) for reproducible tests.
struct SeededGenerator: RandomNumberGenerator {
    private var state: UInt64
    init(seed: UInt64) { state = seed }

    mutating func next() -> UInt64 {
        state &+= 0x9E37_79B9_7F4A_7C15
        var z = state
        z = (z ^ (z >> 30)) &* 0xBF58_476D_1CE4_E5B9
        z = (z ^ (z >> 27)) &* 0x94D0_49BB_1331_11EB
        return z ^ (z >> 31)
    }
}
```

If you want fresh inputs on each run, but still need reproducibility, choose the seed from the clock, and *print it* so that a failure message includes it.

Wherever one function has a simple inverse, randomized *round-trip* tests are especially effective: encode and then decode a random value and check that you get the original back. This is how you'd test the JSON `Codable` types of Section 4.5.

### 11.2.7. Testing a Command

The `swift test` tool is useful for testing library code, but with a little effort we can use it to test commands too. Package `echo` of Section 2.3.2 is split into two parts: a library function `echo` that does the real work, and an executable `main` that parses flags and calls it. A function that writes to the standard output is hard to test, so we give it an *output parameter*, a `TextOutputStream` (Section 7.1), and test with a string:

```swift
// swiftpl/ch11/echo
func echo(_ args: [String], separator: String, newline: Bool,
          to out: inout some TextOutputStream) {
    out.write(args.joined(separator: separator))
    if newline {
        out.write("\n")
    }
}

@Test(arguments: [
    (newline: true, sep: "", args: [String](), want: "\n"),
    (newline: false, sep: "", args: [], want: ""),
    (newline: true, sep: "\t", args: ["one", "two", "three"], want: "one\ttwo\tthree\n"),
    (newline: true, sep: ",", args: ["a", "b", "c"], want: "a,b,c\n"),
    (newline: false, sep: ":", args: ["1", "2", "3"], want: "1:2:3"),
])
func echoTest(newline: Bool, sep: String, args: [String], want: String) {
    var out = ""
    echo(args, separator: sep, newline: newline, to: &out)
    #expect(out == want)
}
```

Notice that the test passes a `String` as the output. It's *dependency injection* of the simplest kind: the function depends on an abstraction, not on the real standard output, so the test can substitute a stand-in. The same idea applies broadly. A function that needs the current time, a random number, the file system, or the network can take a parameter, a protocol, or a closure, that a test replaces with a deterministic version.

### 11.2.8. White-Box Testing

One way of categorizing tests is by the level of knowledge they require of the internal workings of the code under test. A *black-box* test assumes nothing about the module other than what is exposed by its public API and documented by its specification. In contrast, a *white-box* test has privileged access to the internal functions and data structures of the module and can make observations and changes that an ordinary client cannot. For example, a white-box test can check that the invariants of the module's data types are maintained after every operation.

The two approaches are complementary. Black-box tests are usually more robust, needing little updating as the software evolves, and they help the test author empathize with the client of the module and can reveal flaws in the API design. White-box tests can provide more detailed coverage of the trickier parts of the implementation.

In Swift, plain `import` gives you a black-box test, and `@testable import` a white-box one. For example, `Memo`'s internal cache could be inspected by a test with `@testable` to confirm that it never holds more than one task per key.

Sometimes you can make a white-box test unnecessary by injecting a *fake* for a dependency. Consider a function that sends a quota-warning email when a user exceeds 90% of their allowance. Testing it by really sending email would be unacceptable, so we make the sender a parameter:

```swift
// swiftpl/ch11/storage
protocol Notifier: Sendable {
    func notify(user: String, message: String) async
}

func checkQuota(user: String, used: Int, quota: Int, notifier: some Notifier) async {
    guard used * 100 >= quota * 90 else { return }
    await notifier.notify(user: user, message: "You are using \(used * 100 / quota)% of your quota.")
}
```

The test substitutes a recording fake:

```swift
actor RecordingNotifier: Notifier {
    private(set) var sent: [(user: String, message: String)] = []
    func notify(user: String, message: String) {
        sent.append((user, message))
    }
}

@Test func warnsAtNinetyPercent() async {
    let fake = RecordingNotifier()
    await checkQuota(user: "ada@example.com", used: 980_000_000, quota: 1_000_000_000, notifier: fake)
    let sent = await fake.sent
    #expect(sent.count == 1)
    #expect(sent.first?.user == "ada@example.com")
    #expect(sent.first?.message.contains("98%") == true)
}
```

The fake is an actor so that it's `Sendable` and can safely record from any task.

### 11.2.9. Good Tests

Go's authors describe tests that report failures by printing the function called, its inputs, the actual result, and the expected result, so that the message is a complete bug report. `#expect` does much of that automatically, but it's worth adding context, in its optional message, when the expression alone isn't enough:

```swift
#expect(got == want, "isPalindrome(\(input.debugDescription)) = \(got), want \(want)")
```

Good tests share a few qualities. They're *deterministic*: a test that sometimes fails is worse than none, because people learn to ignore it. They're *independent*: no test should depend on another having run first, since tests run in parallel and in no guaranteed order. They're *fast*, so that they're run constantly. And they test *behavior*, not implementation, so that they needn't change whenever the code is refactored.

Avoid being fooled by tests that don't test anything: a test that compares a function's output with the function's own output, or one whose expected value was copied from a first, unchecked run. And don't try to test everything. Tests cost maintenance, and testing trivial code (simple getters, one-line wrappers) is rarely worth it.

**Exercise 11.1:** Write `randomNonPalindrome(using:)`, which produces a string that's guaranteed not to be a palindrome, and test that `isPalindrome` rejects 1,000 of them.

**Exercise 11.2:** Write a randomized test for the `IntSet` of Section 6.5 by comparing its behavior against `Set<Int>` over a long random sequence of `insert`, `contains`, and `formUnion` operations.

**Exercise 11.3:** Write tests for the `comma` function of Section 3.5, including negative numbers and decimals once you've completed Exercise 3.11.

**Exercise 11.4:** Write a test that checks the round trip `decode(encode(x)) == x` for random `Movie` values from Section 4.5.

**Exercise 11.5:** Write a table-driven test for `topoSort` (Section 5.6.2) that checks, for each edge, that the prerequisite comes first in the output.

**Exercise 11.6:** Write a test for the concurrent `Memo` actor of Section 9.7 that starts 100 concurrent requests for the same key and checks that the function was called once.

## 11.3. Coverage

By its nature, testing is never complete. As the influential computer scientist Edsger Dijkstra put it, "Testing shows the presence, not the absence of bugs." No quantity of tests can ever prove a package free of bugs. At best, they increase our confidence that the package works well in a wide range of important scenarios.

The degree to which a test suite exercises the package under test is called the test's *coverage*. Coverage can't be quantified directly, since the dynamic behavior of all but the most trivial programs is beyond precise measurement, but there are heuristics that can help us direct our testing efforts to where they are more likely to be useful.

*Statement coverage* is the simplest and most widely used of these heuristics. The statement coverage of a test suite is the fraction of source statements that are executed at least once during the test. In this section, we'll use `swift test`'s coverage tool to measure statement (really, *line* and *region*) coverage and help identify obvious gaps in the tests.

The code below is a table-driven test for the expression evaluator we built in Chapter 7:

```swift
// swiftpl/ch11/eval
@Test(arguments: [
    ("x + x", "x + x"),
    ("sqrt(4)", "2"),
    ("-3 + 5", "2"),
    ("2 * (3 + 4)", "14"),
    ("1 +", nil),  // syntax error
])
func parseAndEval(_ input: String, _ want: String?) throws { ... }
```

Let's check how much of the evaluator these tests cover. Run the tests with coverage enabled, then ask `llvm-cov` for a report:

```
$ swift test --enable-code-coverage
$ swift test --show-codecov-path
/home/user/eval/.build/debug/codecov/eval.json

$ llvm-cov report \
    .build/debug/evalPackageTests.xctest \
    -instr-profile=.build/debug/codecov/default.profdata \
    -ignore-filename-regex='\.build|Tests'
Filename        Regions  Missed  Cover  Functions  Executed  Lines  Missed  Cover
-----------------------------------------------------------------------------------
Eval.swift          38       7  81.58%         9      88.89%    96      10  89.58%
Parser.swift        52      11  78.85%        12     100.00%   141      14  90.07%
-----------------------------------------------------------------------------------
TOTAL               90      18  80.00%        21      95.24%   237      24  89.87%
```

(On macOS, the tool is invoked as `xcrun llvm-cov`, and the test bundle path ends in `.xctest/Contents/MacOS/...`; on Linux, as shown. Your numbers will differ.)

Our test coverage is 90% of lines, which is decent, but we'd like to see *which* lines were missed. The `show` subcommand annotates the source with execution counts:

```
$ llvm-cov show .build/debug/evalPackageTests.xctest \
    -instr-profile=.build/debug/codecov/default.profdata \
    Sources/Eval/Eval.swift
   ...
   54|      3|        case .binary(let op, let x, let y):
   55|      3|            switch op {
   56|      1|            case "+": return x.eval(env) + y.eval(env)
   57|      0|            case "-": return x.eval(env) - y.eval(env)
   58|      1|            case "*": return x.eval(env) * y.eval(env)
   59|      0|            case "/": return x.eval(env) / y.eval(env)
```

The count column shows that subtraction and division never ran. Lines with a count of zero are the ones the tests never exercised. Adding two rows to the table fixes them:

```swift
    ("10 - 4", "6"),
    ("9 / 3", "3"),
```

An HTML report with the same information, in color, is produced by `llvm-cov show -format=html -output-dir=coverage`, and Xcode shows coverage inline in its editor.

Achieving 100% statement coverage sounds like a noble goal, but it's almost always impractical and unlikely to be a good use of effort. Just because a statement is executed doesn't mean it's bug-free; statements containing complex expressions must be executed many times with different inputs to cover the interesting cases. Some statements, like the `fatalError` in the default branch of a `switch`, can't be reached by definition. Others, such as error-handling code for hardware failures, are hard to reach but also hard to test.

Testing is fundamentally a pragmatic endeavor, a trade-off between the cost of writing tests and the cost of failures that tests might have prevented. Coverage tools can help identify the weakest spots, but devising good test cases demands the same rigorous thinking as programming in general.

## 11.4. Benchmarks

Benchmarking is the practice of measuring the performance of a program on a fixed workload. In Go, benchmarks are a built-in kind of function in the `testing` package. Swift Testing doesn't provide benchmarks; it's focused on correctness. Performance measurement in Swift uses one of three approaches.

The simplest is to time a block of code with `ContinuousClock`, whose `measure` method returns a `Duration`:

```swift
let clock = ContinuousClock()
let elapsed = clock.measure {
    for _ in 0..<1000 {
        _ = isPalindrome("A man, a plan, a canal: Panama")
    }
}
print("1000 calls took \(elapsed)")
```

That's fine for a rough comparison, but naïve timing is fragile. The first iteration may be slower because of cold caches and lazy initialization; the optimizer may notice that the result is never used and delete the loop altogether; and a single measurement says nothing about variance. Run such experiments in a release build (`swift run -c release`), use the result, and run them several times.

To avoid the optimizer's cleverness, pass the result through a function that the compiler can't see through. A convention is a function named `blackHole`, marked `@inline(never)`:

```swift
@inline(never) func blackHole<T>(_ x: T) {}

for _ in 0..<1000 {
    blackHole(isPalindrome(input))
}
```

The second approach is the `XCTest` framework's `measure` method, which runs a block ten times and records the average and standard deviation, and which on Apple platforms integrates with Xcode's baselines for regression detection:

```swift
import XCTest

final class PalindromeBenchmarks: XCTestCase {
    func testPerformance() {
        measure {
            for _ in 0..<1000 { _ = isPalindrome("A man, a plan, a canal: Panama") }
        }
    }
}
```

The third approach, for serious benchmarking, is the `package-benchmark` package from Ordo One. It provides a separate `swift package benchmark` command and a `Benchmark` API, runs each benchmark many times, and reports statistics across several *metrics*: wall-clock time, CPU time, memory allocations, retain/release counts, and context switches. Allocation and reference-counting metrics are especially valuable in Swift, where hidden costs are as often in ARC traffic as in raw computation:

```swift
// Benchmarks/Palindrome/Palindrome.swift
import Benchmark
import Word

let benchmarks = {
    Benchmark("isPalindrome") { benchmark in
        for _ in benchmark.scaledIterations {
            blackHole(isPalindrome("A man, a plan, a canal: Panama"))
        }
    }
}
```

```
$ swift package benchmark
Palindrome:isPalindrome
╒═════════════════════════╤═════════╤═════════╤═════════╤═════════╤═════════╤═════════╤═════════╤═════════╕
│ Metric                  │      p0 │     p25 │     p50 │     p75 │     p90 │     p99 │    p100 │ Samples │
╞═════════════════════════╪═════════╪═════════╪═════════╪═════════╪═════════╪═════════╪═════════╪═════════╡
│ Time (wall clock) (ns)  │     480 │     498 │     512 │     531 │     558 │     640 │    1920 │  100000 │
│ Malloc (total)          │       3 │       3 │       3 │       3 │       3 │       3 │       3 │  100000 │
...
```

Whichever approach you use, the guidance is the same. Measure before you optimize, since intuitions about performance are frequently wrong; measure in release mode, with representative inputs; and compare *before and after* on the same machine. Benchmarks are most useful as a comparison between alternative implementations. For example, we can measure the first, loop-based `isPalindrome` of Section 11.2 against the filter-and-reverse one. Which is faster? Which allocates less?

**Exercise 11.7:** Benchmark the two versions of `isPalindrome`, and then a third that uses two indices walking inward over the `unicodeScalars` of an ASCII-lowercased copy. How does performance depend on the length of the input?

**Exercise 11.8:** Benchmark `IntSet` against `Set<Int>` for `insert`, `contains`, and `formUnion` at several set sizes. Where does the bit vector win?

**Exercise 11.9:** Measure the cost of `ArraySlice` against `Array` copies in a recursive algorithm like merge sort, with `-c release`.

## 11.5. Profiling

Benchmarks are useful for measuring the performance of specific operations, but when we're trying to make a slow program faster, we often have no idea where to begin. Every programmer knows Donald Knuth's aphorism about premature optimization, which appeared in "Structured Programming with go to Statements" in 1974. Although it is often misquoted to mean that performance doesn't matter, in its original context we can see the intended meaning: "There is no doubt that the grail of efficiency leads to abuse. Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We *should* forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%."

When we wish to look carefully at the speed of our programs, the best technique for identifying the critical code is *profiling*. Profiling is an automated approach to performance measurement based on sampling a number of profile events during execution, then extrapolating from them during a post-processing step; the resulting statistical summary is called a profile.

On Apple platforms, the profiler of choice is *Instruments*, which is part of Xcode. Its *Time Profiler* samples the call stacks of every thread at high frequency and presents the result as a call tree, showing which functions the program spends the most time in, both themselves ("self time") and including their callees. Other instruments measure allocations, leaks, system calls, disk and network I/O, thread states, and energy use. Because Swift's performance is often dominated by memory allocation and reference counting, the *Allocations* instrument and the *Swift Tasks* instrument (which shows when tasks are created, suspended, and resumed) are particularly useful for Swift programs.

On Linux, the standard tool is `perf`, together with a flame-graph generator:

```
$ swift build -c release
$ perf record -g .build/release/palindrome-bench
$ perf report
# Overhead  Command  Shared Object   Symbol
# ........  .......  ..............  ...........................................
    31.42%  bench    libswiftCore.so [.] swift_retain
    22.87%  bench    libswiftCore.so [.] swift_release
    11.30%  bench    libswiftCore.so [.] swift_allocObject
     9.05%  bench    bench           [.] $s4Word12isPalindromeySbSSF
```

The symbol names are *mangled*; run them through `swift demangle` to read them. The profile above is illustrative, but typical: a large share of the time is in `swift_retain`, `swift_release`, and `swift_allocObject`, the reference-counting and allocation routines. That's a common finding in Swift code, and it suggests where to look. The tool `valgrind --tool=callgrind` and Google's `heaptrack` work on Linux as well, and `swift-inspect` lets you look at the allocations and metadata of a running Swift process.

Build with optimization (`-c release`), but with debug symbols so the profiler can map addresses to source lines; SwiftPM does this by default. Profile a realistic workload, not a toy.

### 11.5.1. Common Culprits

A few patterns recur in Swift profiles, and are worth knowing about before you start.

*Excess reference counting.* Every copy of a class reference, or a value containing one, such as a `String`, an array, or a closure, may involve an atomic increment and decrement. Passing values as `borrowing` or `consuming` parameters (Swift 5.9) can eliminate some of it, as can using value types containing no references, and avoiding unnecessary copies in tight loops.

*Existentials and dynamic dispatch.* Calling a method through `any Protocol` is an indirect call through a witness table, can't be inlined, and may allocate if the value doesn't fit in the existential's inline buffer. Generic code (`some Protocol`) that is specialized by the optimizer avoids all of this.

*Unspecialized generics.* When generic code is called across a module boundary without `@inlinable`, the optimizer can't specialize it, and it runs the slow path, passing around type metadata. If a profile shows generic functions with names like `swift_getGenericMetadata` or `swift_conformsToProtocol`, that's the symptom.

*Strings.* Character-based string processing is more expensive than byte processing, because it must find grapheme cluster boundaries. For ASCII-oriented data, working with the `utf8` view can be many times faster, as we noted in Section 3.5.

*Copy-on-write surprises.* A mutation of an array that isn't uniquely referenced silently copies the whole buffer. The classic way this happens is holding a second reference to the array, for example in a local `let` that's still in scope, or in a closure, while you mutate it in a loop, turning an O(*n*) loop into O(*n*²).

*Blocking in async code.* As we saw in Section 9.8, a blocking call inside a task occupies one of a small number of threads. A profile showing threads idle in `read` or `sleep` while tasks queue up is the sign.

**Exercise 11.10:** Profile the `charcount` program of Section 4.3 with a large input file. What dominates? Rewrite the hot loop to use the `utf8` view for ASCII-only input and measure the difference.

**Exercise 11.11:** Construct a program that unintentionally copies an array on every iteration of a loop, as described in the copy-on-write paragraph above. Find it with a profiler, then fix it.

## 11.6. Documentation Examples

The last kind of test we'll discuss is an *example*. In Go, an `Example` function in a test file is compiled, run, and (if it has an `// Output:` comment) verified, and it's also shown in the generated documentation. Swift has no direct counterpart in the test runner, but the documentation system fills the gap.

*DocC* documentation, introduced in Section 10.7, supports Swift code blocks in doc comments, which are shown with syntax highlighting in the generated site:

```swift
/// Reports whether `s` reads the same forward and backward,
/// ignoring punctuation, spaces, and case.
///
/// ```swift
/// isPalindrome("A man, a plan, a canal: Panama")  // true
/// isPalindrome("palindrome")  // false
/// ```
public func isPalindrome(_ s: String) -> Bool { ... }
```

Examples like this serve as documentation, and, being text, can go out of date. To keep them honest, DocC *tutorials* and *articles* built with the `swift-docc` toolchain compile code listings that are stored as separate files in the documentation catalog. Another approach is to mirror each important documentation example in a real test. Swift Testing makes that cheap, and the tests can be named for what they demonstrate:

```swift
@Test("isPalindrome ignores punctuation and case")
func example() {
    #expect(isPalindrome("A man, a plan, a canal: Panama"))
    #expect(!isPalindrome("palindrome"))
}
```

The `swift-snapshot-testing` package takes a different approach: it records the output of a function the first time it runs and compares later runs against the recording. Snapshot tests are most useful for large, structured outputs like rendered HTML, a pretty-printed syntax tree, or a command's help text, where writing the expected value by hand would be tedious. Review the recorded snapshot as carefully as you would any other code, since the test is only as good as the first output you accepted.

Documentation examples have three purposes. Primarily, they serve as documentation: an example can be a more succinct or intuitive way to convey the behavior of a library function than its prose description, especially when used as a reminder or quick reference. Second, they are verified, either by the compiler as part of the build or by a test. And third, they are hands-on experiments: the Swift REPL (`swift repl`), playgrounds in Xcode, and the Swift Playground app on iPad all let readers copy an example and try changing it, and so learn the language by doing.

**Exercise 11.12:** Add documentation comments with examples to the `IntSet` of Section 6.5, and write tests that verify each example.

**Exercise 11.13:** Write a snapshot-style test for the `Expr` pretty-printer of Exercise 7.13, storing the expected output in a file alongside the test. How would you make updating the stored snapshots deliberate rather than automatic?
