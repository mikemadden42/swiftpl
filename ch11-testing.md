# 11. Testing

Every program of any size contains bugs; the only question is who finds them first. If it's the programmer, a bug costs a few minutes. If it's a user, it costs time, data, and trust, and then the programmer's time as well, now spent reconstructing what went wrong from a vague report. Most of the practices of professional software development, from code review to static types to staged rollouts, are ways of moving bug discovery earlier. This chapter is about the most direct of them: *automated testing*, writing code whose job is to check other code.

A test is a small program that runs some part of the *code under test* with chosen inputs and checks that the results are what they should be. The inputs might be picked by hand to probe particular cases, generated at random to probe many cases, or recorded from real use. Run often, tests catch mistakes moments after they're made, while the change that caused them is still fresh in mind. Kept around, they guard against old bugs coming back. And written early, they force a question that's easy to skip: what, exactly, is this code supposed to do?

The Swift toolchain provides two testing libraries. *XCTest*, the older one, comes from the Objective-C world. Tests are methods of subclasses of `XCTestCase`, and checks are made with a family of functions like `XCTAssertEqual` and `XCTAssertNil`. *Swift Testing*, included in the toolchain since Swift 6, was designed around the modern language. A test is any function marked with the `@Test` macro, and nearly every check is written with a single macro, `#expect`, which inspects the expression you give it to produce a detailed failure message. Both libraries are run by the same command, `swift test`, and can coexist in one package. This chapter uses Swift Testing throughout, with notes where XCTest does things differently.

## 11.1. The `swift test` Tool

Tests live in *test targets*, which the manifest declares with `.testTarget` and whose sources go, by convention, under `Tests/`:

```swift
.testTarget(name: "EvalTests", dependencies: ["Eval"])
```

`swift test` builds every test target, along with the code it depends on, finds every function marked `@Test`, and runs them all, in parallel unless told otherwise. It reports each failure with its source location and finishes with a summary.

A test target is a separate module from the code it tests. It imports the module under test like any other client, so by default it sees only `public` declarations. To reach `internal` ones, import the module with `@testable`:

```swift
@testable import Eval
```

A testable import works only when the module under test was compiled with testing enabled, as debug builds are by default. It's convenient, but use it with some restraint: tests that reach into internals have to change whenever the internals do. Testing through the public interface keeps tests valid across refactoring.

These are the options you'll use most:

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

A test is a function marked `@Test`, either at the top level of a file or inside a type that groups tests, called a *suite*. The test passes if it runs to completion with no failed expectations and no error escapes it.

As a running example, we'll test a function that turns the title of an article into a *slug*, the short, lowercase, hyphenated form used in URLs, so that "Hello, World" can be found at `/blog/hello-world`. Here's a first attempt:

```swift
// swiftpl/ch11/slug1/Sources/Slug/Slug.swift
/// Converts a title to a URL slug: "Hello World" becomes "hello-world".
/// (A first attempt.)
public func slugify(_ title: String) -> String {
    title.lowercased().split(separator: " ").joined(separator: "-")
}
```

and a test file to go with it:

```swift
// swiftpl/ch11/slug1/Tests/SlugTests/SlugTests.swift
import Testing
import Slug

@Test func simpleTitle() {
    #expect(slugify("Hello World") == "hello-world")
}

@Test func extraSpaces() {
    #expect(slugify("  Swift   on   Linux ") == "swift-on-linux")
}
```

`#expect` takes a Boolean expression and records a failure if it's false. A failed expectation doesn't end the test, so a test with several checks reports every one that fails, not just the first. (For checks that later code depends on, there's `#require`, which does stop the test; see Section 11.2.3.)

```
$ swift test
Building for debugging...
Test run started.
Testing Library Version: 6.1
◇ Test simpleTitle() started.
◇ Test extraSpaces() started.
✔ Test simpleTitle() passed after 0.001 seconds.
✔ Test extraSpaces() passed after 0.001 seconds.
✔ Test run with 2 tests passed after 0.001 seconds.
```

(The exact format of the output varies a little between toolchain versions.)

Both tests pass. Then the bug reports start arriving. A post titled "Hello, World!" got the slug `hello,-world!`, which is a legal but ugly URL, and one titled "Crème brûlée" got `crème-brûlée`, which some older systems mangle. Before touching the function, we write tests that capture the reports:

```swift
@Test func punctuation() {
    #expect(slugify("Hello, World!") == "hello-world")
}

@Test func accents() {
    #expect(slugify("Crème brûlée") == "creme-brulee")
}
```

Both fail, as they should:

```
✘ Test punctuation() recorded an issue at SlugTests.swift:13:5: Expectation failed: (slugify("Hello, World!") → "hello,-world!") == "hello-world"
✘ Test accents() recorded an issue at SlugTests.swift:17:5: Expectation failed: (slugify("Crème brûlée") → "crème-brûlée") == "creme-brulee"
```

Look at what the failure message contains: the source text of the expression, and the value of each side of the `==`. The macro sees the expression's structure at compile time and arranges to capture the operands, so a plain `==` reports as much as a specialized `assertEqual` would. That's why Swift Testing needs only one checking macro.

Writing the failing test first is a habit worth building. It proves the test can detect the bug (a test that passes before the fix tests nothing), and afterwards it stays in the suite to catch a regression. Now the fix: fold away diacritics, then keep runs of letters and digits, with anything else acting as a separator:

```swift
// swiftpl/ch11/slug2/Sources/Slug/Slug.swift
import Foundation

/// Converts a title to a URL slug: lowercase letters and digits,
/// with diacritics removed and words separated by single hyphens.
public func slugify(_ title: String) -> String {
    let folded = title.folding(options: .diacriticInsensitive, locale: nil).lowercased()
    var words: [String] = []
    var word = ""
    for c in folded {
        if c.isLetter || c.isNumber {
            word.append(c)
        } else if !word.isEmpty {
            words.append(word)
            word = ""
        }
    }
    if !word.isEmpty {
        words.append(word)
    }
    return words.joined(separator: "-")
}
```

Foundation's `folding(options:locale:)` maps "è" to "e", "û" to "u", and so on. Because the loop tests `isLetter` and `isNumber` rather than checking for ASCII, letters from other scripts survive: a Japanese title keeps its characters. All four tests pass.

### 11.2.1. Test Names and Traits

`@Test` accepts a display name, for reports, and any number of *traits*, which adjust how or whether the test runs:

```swift
@Test("Punctuation becomes a word break")
func punctuation() { ... }

@Test(.tags(.slow), .timeLimit(.minutes(1)))
func hugeInput() { ... }

@Test(.disabled("waiting on bug #123"))
func knownBad() { ... }

@Test(.enabled(if: ProcessInfo.processInfo.environment["CI"] != nil))
func onlyOnCI() { ... }

@Test(.bug("https://example.com/issues/123"), .tags(.regression))
func issue123() { ... }
```

*Tags* label tests across files and suites. Each is declared once, with the `@Tag` macro, and can then be used to select tests with `swift test --filter` or in Xcode's test navigator:

```swift
extension Tag {
    @Tag static var slow: Self
    @Tag static var regression: Self
}
```

### 11.2.2. Parameterized Tests

Our four slug tests are nearly identical: call the function, compare with an expected string. As more cases arrive, writing a function for each becomes tedious. In many languages, the remedy is a hand-written loop over a table of cases. Swift Testing does the looping for you. Give the test a parameter, and supply the values with `arguments:`:

```swift
// swiftpl/ch11/slug3/Tests/SlugTests/SlugTests.swift
import Testing
import Slug

@Test(arguments: [
    ("Hello World", "hello-world"),
    ("  Swift   on   Linux ", "swift-on-linux"),
    ("Hello, World!", "hello-world"),
    ("Crème brûlée", "creme-brulee"),
    ("Ünïcödé", "unicode"),
    ("Swift 6.2 released", "swift-6-2-released"),
    ("C++ vs. C#", "c-vs-c"),
    ("日本語", "日本語"),
    ("Straße", "straße"),  // ß is a letter of its own, not an accented s
    ("---", ""),
    ("", ""),
])
func slugs(_ title: String, _ want: String) {
    #expect(slugify(title) == want)
}
```

Swift Testing runs each pair as its own *test case*. The cases run in parallel, each passes or fails on its own, and a failure is reported with the argument that caused it, so a single bad row doesn't hide the results of the rest.

Two of those rows record *decisions* rather than obvious truths. Should "C++" and "C#" really both become "c"? Should "ß" stay as it is? Reasonable people could disagree; the table makes the choice explicit and visible to the next person who changes the function. Every expected value in the table was written by hand, and that matters. It's tempting to compute expected results with a copy of the code being tested, but a test that compares a function with itself can never fail.

Tests may take more than one parameter. With two collections of arguments, Swift Testing runs every combination; to pair them up instead, pass a `zip`:

```swift
@Test(arguments: [1, 2, 3], ["a", "b"])
func combos(n: Int, s: String) { ... }  // six test cases

@Test(arguments: zip([1, 2, 3], ["one", "two", "three"]))
func pairs(n: Int, name: String) { ... }  // three test cases
```

For a larger table, a small struct reads better than a tuple, and conforming it to `CustomTestStringConvertible` controls how each case is named in reports. Here is a table for the expression evaluator of Section 7.9:

```swift
struct Case: Sendable, CustomTestStringConvertible {
    let input: String
    let env: Env
    let want: String
    var testDescription: String { input }
}

@Test(arguments: [
    Case(input: "sqrt(x * x + y * y)", env: ["x": 3, "y": 4], want: "5"),
    Case(input: "max(a, b) - min(a, b)", env: ["a": 3, "b": 10], want: "7"),
    Case(input: "abs(t - 20)", env: ["t": 17.5], want: "2.5"),
    Case(input: "price * (1 + tax)", env: ["price": 24.5, "tax": 0.08], want: "26.46"),
    Case(input: "-(1 - 2) * 3", env: [:], want: "3"),
    Case(input: "1 / 0", env: [:], want: "inf"),
])
func evalTable(_ c: Case) throws {
    let expr = try parse(c.input)
    let got = String(format: "%.6g", expr.eval(c.env))
    #expect(got == c.want, "\(c.input) in \(c.env)")
}
```

Since `evalTable` is declared `throws`, it can call the parser with a plain `try`. If parsing fails, the error fails the test case and appears in the report. A failing case is identified by its description. If the expected result for the tax calculation had been mistyped as `26.5`, for instance, the report would say:

```
✘ Test evalTable(_:) with 6 test cases failed after 0.002 seconds with 1 issue.
✘ Test evalTable(_:) recorded an issue with 1 argument c → price * (1 + tax): Expectation failed: (got → "26.46") == (c.want → "26.5")
```

### 11.2.3. Throwing and Required Expectations

Failing correctly is part of a function's behavior, and deserves tests of its own. `#expect(throws:)` passes only if its closure throws an error of the expected kind:

```swift
#expect(throws: SyntaxError.self) {
    try parse("x % 2")
}

#expect(throws: ParseError.unexpectedEnd) {  // a specific, Equatable error value (Section 7.8)
    try parseNumber("")
}

#expect(throws: Never.self) {  // asserts that nothing is thrown
    try parse("1 + 2")
}
```

When the error is caught, the macro returns it, so the test can examine it further:

```swift
let error = #expect(throws: SyntaxError.self) { try parse("1 +") }
#expect(error?.description.contains("unexpected") == true)
```

`#require` is the strict version of `#expect`. If its condition is false, it records the failure and then *throws*, ending the test on the spot, so it must be written with `try`. Use it for facts the rest of the test can't do without. Given an optional, it unwraps it, which removes a lot of awkward `if let` scaffolding from tests:

```swift
@Test func lookup() throws {
    let user = try #require(db.user(named: "ada"))  // stops the test if nil
    #expect(user.id == 1)  // user is a User here, not a User?
}
```

A good default is `#expect` for checks and `#require` for preconditions.

### 11.2.4. Suites, Setup, and Teardown

A *suite* is a struct, class, or actor whose methods are tests. Swift Testing creates a new instance of the suite for every test it runs, so the suite's initializer is the place for setup, and, in a class or actor, `deinit` is the place for teardown. No test ever sees state left behind by another:

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

Each test gets a fresh instance, and so a fresh directory, which is what lets tests in a suite run in parallel safely. When tests must share something that can't be duplicated, such as a single external database or a fixed network port, mark the suite `@Suite(.serialized)` to run its tests one after another.

### 11.2.5. Testing Asynchronous Code

A test function can be `async`, and then it can `await` like any other asynchronous code. That makes the concurrent programs of Chapters 8 and 9 straightforward to test:

```swift
/// A thread-safe call counter. (A Mutex is noncopyable, so it can't be
/// captured by an escaping closure directly; a class wraps it.)
final class CallCount: Sendable {
    private let n = Mutex(0)
    func increment() { n.withLock { $0 += 1 } }
    var value: Int { n.withLock { $0 } }
}

@Test func memoComputesOnce() async throws {
    let calls = CallCount()
    let memo = Memo<String, Int> { key in
        calls.increment()
        return key.count
    }
    async let a = memo.get("hello")
    async let b = memo.get("hello")
    let results = try await [a, b]
    #expect(results == [5, 5])
    #expect(calls.value == 1)  // the function ran only once
}
```

For APIs that report events through callbacks rather than return values, `confirmation` checks that the callback happens the right number of times:

```swift
/// A minimal callback-based event source.
final class Ticker {
    private let onTick: () -> Void
    init(onTick: @escaping () -> Void) { self.onTick = onTick }
    func tick() { onTick() }
}

@Test func ticksTwice() async {
    await confirmation("onTick called", expectedCount: 2) { confirm in
        let t = Ticker(onTick: { confirm() })
        t.tick()
        t.tick()
    }
}
```

### 11.2.6. Randomized and Property-Based Testing

A table checks the cases its author thought of. Bugs, unhelpfully, tend to live in the cases nobody thought of. Randomized testing explores further by generating inputs by the thousand. The difficulty is knowing what the right answer is for an input nobody wrote down.

One strategy is a second, simpler implementation (slow, perhaps, but obviously correct) to compare against. Another is to check *properties*: statements that must be true of the output for every input, whatever the exact answer is. A slug, for instance, should never begin or end with a hyphen or contain two in a row; should contain only lowercase letters, digits, and hyphens; and should be unchanged if slugified again. None of those needs a precomputed answer, and together they rule out a lot of wrong code.

Here's a property-based test for `slugify`. It builds random titles from a pool of characters chosen to include awkward ones: punctuation, accented capitals, digits, underscores, a hyphen, and some CJK text.

```swift
// swiftpl/ch11/slug3
/// Returns a random title of up to 29 characters from an awkward alphabet.
func randomTitle(using rng: inout some RandomNumberGenerator) -> String {
    let pool = Array("abcXYZ019 éÉçÇ-_,.!?'日本 ")
    let n = Int.random(in: 0..<30, using: &rng)
    return String((0..<n).map { _ in pool.randomElement(using: &rng)! })
}

@Test func slugProperties() {
    var rng = SeededGenerator(seed: 42)  // deterministic, so failures reproduce
    for _ in 0..<1000 {
        let title = randomTitle(using: &rng)
        let slug = slugify(title)
        let context: Comment = "slugify(\(title.debugDescription)) = \(slug.debugDescription)"

        #expect(!slug.hasPrefix("-") && !slug.hasSuffix("-") && !slug.contains("--"), context)
        #expect(slug.allSatisfy { $0 == "-" || (($0.isLetter || $0.isNumber) && !$0.isUppercase) }, context)
        #expect(slugify(slug) == slug, context)  // idempotent
    }
}
```

Each check carries a message that includes the generated title, so a failure tells you exactly which input broke the property.

Random tests must still be reproducible. A test that fails once and then can't be made to fail again is a frustrating thing to debug. So the generator is *seeded*: given the same seed, it produces the same sequence, and the test produces the same inputs on every run. The standard library's default generator can't be seeded, but `RandomNumberGenerator` asks for only one method, so it takes a few lines to write one. This is SplitMix64, a small, fast, well-studied algorithm:

```swift
// swiftpl/ch11/slug3 (continued)
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

A fixed seed explores the same thousand inputs every time. To explore new ground on each run while keeping failures reproducible, derive the seed from the clock and include it in every failure message.

Round trips are a particularly productive kind of property: for an encoder and decoder, `decode(encode(x)) == x` must hold for every `x`. That's a natural way to test the `Codable` types of Section 4.5 and the S-expression encoder of Chapter 12.

### 11.2.7. Testing a Command

Command-line programs are tested most easily when their logic is pulled out of `main` into functions that take their inputs as parameters and deliver their output somewhere the test can see. Printing straight to the standard output is the usual obstacle. The fix is to have the function write to a `TextOutputStream` (Section 7.1) passed by the caller: the real program passes the standard output, and a test passes a `String`.

Here's the core of `tally`, a command that prints the most frequent words in its input:

```swift
// swiftpl/ch11/tally
/// Writes the n most frequent words in text, most frequent first,
/// with ties broken alphabetically.
func tally(_ text: String, top n: Int, to out: inout some TextOutputStream) {
    var counts: [String: Int] = [:]
    for word in text.split(whereSeparator: { !$0.isLetter }) {
        counts[word.lowercased(), default: 0] += 1
    }
    let ranked = counts.sorted {
        $0.value != $1.value ? $0.value > $1.value : $0.key < $1.key
    }
    for (word, count) in ranked.prefix(n) {
        out.write("\(count) \(word)\n")
    }
}
```

and a test of it:

```swift
// swiftpl/ch11/tally (continued)
@Test(arguments: [
    (text: "", top: 3, want: ""),
    (text: "a b a", top: 1, want: "2 a\n"),
    (text: "The the THE cat", top: 2, want: "3 the\n1 cat\n"),
    (text: "pear apple pear apple fig", top: 2, want: "2 apple\n2 pear\n"),
    (text: "one, two; three", top: 5, want: "1 one\n1 three\n1 two\n"),
])
func tallyTest(text: String, top: Int, want: String) {
    var out = ""
    tally(text, top: top, to: &out)
    #expect(out == want)
}
```

This is *dependency injection* at its most modest: rather than reaching for the standard output itself, the function lets its caller decide where output goes. The same move works for anything a function would otherwise grab from its environment, such as the current time, randomness, files, or the network, and it's the most useful single technique for making code testable.

### 11.2.8. White-Box Testing

Tests can be sorted by how much they know about the code they test. A *black-box* test uses only the public interface and the documented behavior; it knows *what* the code should do but not *how*. A *white-box* test also uses knowledge of the implementation. It might check internal invariants after each operation, or steer execution into a particular branch.

Each kind has its strengths. Black-box tests survive refactoring, since they don't depend on the internals, and writing them puts you in the client's position, which tends to expose awkward APIs. White-box tests can reach corners that are hard to provoke from outside. In Swift, the distinction maps onto imports: a plain `import` gives a black-box test, and `@testable import` lets a white-box test see `internal` details.

Often, though, the best way to test hard-to-reach behavior is to make it easy to reach by injecting its dependencies. Consider a rate limiter that allows at most `limit` events in any window of `window` seconds. Its behavior depends on the passage of time, and a test that actually waited for windows to expire would be slow and flaky. So the limiter takes its clock as a parameter, defaulting to the real one:

```swift
// swiftpl/ch11/ratelimit
import Foundation

struct RateLimiter {
    let limit: Int
    let window: Double  // seconds
    private let now: () -> Double
    private var recent: [Double] = []  // times of recent events

    init(limit: Int, window: Double, now: @escaping () -> Double = { Date().timeIntervalSince1970 }) {
        self.limit = limit
        self.window = window
        self.now = now
    }

    /// Records an event and reports whether it's within the limit.
    mutating func allow() -> Bool {
        let t = now()
        recent.removeAll { $0 <= t - window }
        guard recent.count < limit else {
            return false
        }
        recent.append(t)
        return true
    }
}
```

The test supplies a clock it controls completely:

```swift
// swiftpl/ch11/ratelimit (continued)
@Test func limitsBursts() {
    var time = 0.0
    var limiter = RateLimiter(limit: 2, window: 1.0, now: { time })

    #expect(limiter.allow())
    #expect(limiter.allow())
    #expect(!limiter.allow())  // third event in the same instant: refused

    time = 0.9
    #expect(!limiter.allow())  // still inside the window

    time = 1.5
    #expect(limiter.allow())  // the first two events have aged out
}
```

The test runs in microseconds, and its timing is exact. It's still a black-box test: it uses only the limiter's public behavior. The injected clock is part of the interface, not a peek at the implementation.

### 11.2.9. Writing Good Tests

When a test fails, its message is often all that someone has to go on, perhaps someone else, months from now, looking at a CI log. Make sure the message says what was being tested, with what input, what happened, and what should have happened. `#expect` supplies the expression and its values; add the rest with its optional comment argument when the expression alone isn't self-explanatory:

```swift
#expect(got == want, "slugify(\(title.debugDescription)) = \(got.debugDescription), want \(want.debugDescription)")
```

Beyond messages, a few properties separate tests that help from tests that hinder:

- **Deterministic.** A test that fails intermittently trains everyone to ignore failures. Control time, randomness, and ordering.
- **Independent.** Tests run in parallel and in no particular order; none may rely on another's side effects.
- **Fast.** Slow suites get run less often, and a bug caught a day later costs more.
- **About behavior.** A test that pins down *how* the code works must be rewritten whenever the implementation improves. A test of *what* it does keeps paying off.

And beware tests that can't fail: those that compare a function's output with itself, and those whose expected values were pasted from the code's first run without being checked. Finally, don't aim to test everything. Tests have a maintenance cost too, and trivial code is rarely worth it. Spend the effort where bugs are likely and costly.

**Exercise 11.1:** Write a *reference* implementation of `slugify` using a regular expression (`Regex` or `NSRegularExpression`) instead of a loop, and a randomized test that checks the two agree on 10,000 random titles. When they disagree, which one is right?

**Exercise 11.2:** Write a randomized test for the `RingBuffer` of Section 6.5 that applies a long random sequence of `append` and `popFirst` operations both to a ring buffer and to a simple reference model (an array that drops its first element when it grows past the capacity), and checks that they always agree.

**Exercise 11.3:** Write tests for `runLengthEncoded` from Section 3.5, including empty strings, single characters, and multi-scalar characters such as flags. Once you've done Exercise 3.9, add a round-trip property test.

**Exercise 11.4:** Write a round-trip test, `decode(encode(x)) == x`, for randomly generated values of the `CatalogEntry` type from Section 4.5.

**Exercise 11.5:** Write a parameterized test for `buildOrder` (Section 5.6.2) that checks, for every target and each of its dependencies, that the dependency comes first.

**Exercise 11.6:** Test the `Memo` actor of Section 9.7 by starting 100 concurrent requests for one key and checking that the underlying function ran exactly once.

## 11.3. Coverage

However many tests you write, they examine only some of the program's possible executions; the rest go unchecked. It's useful to know which parts of the code the tests never reach, since bugs there will be found by users. *Coverage* measures that.

The simplest and most common measure is *statement coverage*: the proportion of the program's statements that were executed at least once during the test run. Swift's tooling, which is built on LLVM's, reports it per line and per *region* (a finer unit that distinguishes, for example, the two sides of `&&`). High coverage doesn't prove anything is correct, but low coverage proves that something is untested.

Let's measure the tests for the expression evaluator of Chapter 7. Suppose they consist of a single parameterized test:

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

`swift test` collects coverage data when asked, and `llvm-cov` turns the data into a report:

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

(On macOS, run the tool as `xcrun llvm-cov` and point it at the executable inside the `.xctest` bundle. Your figures will differ from these illustrative ones.)

Nine lines in ten is respectable, but the summary doesn't say *which* tenth was missed. The `show` subcommand prints the source annotated with how many times each line ran:

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

The zeros jump out: no test ever subtracted or divided. Two more rows close the gap:

```swift
    ("10 - 4", "6"),
    ("9 / 3", "3"),
```

`llvm-cov show -format=html -output-dir=coverage` writes the same information as a browsable, color-coded site, and Xcode can display coverage directly in the editor's gutter.

Treat coverage as a flashlight, not a score. Reaching 100% is rarely worth the effort, and it guarantees less than it seems to. A line that ran once, with one input, may still be wrong for others. Some lines are unreachable by design, such as a `fatalError` in a `default` case that can't happen. Others, such as recovery from a full disk, are reachable but expensive to test. What coverage does well is point out large, important regions of code that no test touches at all. Those are where to write the next tests.

## 11.4. Benchmarks

A *benchmark* measures how long a piece of code takes, or how much memory it uses, on a fixed workload, so that you can compare implementations or notice when a change makes things slower. Swift Testing is concerned with correctness and has no benchmark support, so Swift programmers measure performance in one of three ways.

The quickest is to time the code with a clock. `ContinuousClock` has a `measure` method that runs a closure and returns how long it took:

```swift
let clock = ContinuousClock()
let elapsed = clock.measure {
    for _ in 0..<1000 {
        _ = slugify("Crème brûlée: A Surprisingly Long Title, With Punctuation!")
    }
}
print("1000 calls took \(elapsed)")
```

This is fine for a quick look, but it's easy to fool yourself. A debug build may be an order of magnitude slower than a release build, so always measure with `-c release`. The first iterations pay for cold caches and lazy initialization. One measurement says nothing about how much results vary from run to run. And the optimizer, noticing that the result is discarded, may remove the work entirely and report an impressively short time for doing nothing. The standard defense is to pass every result to a function that the optimizer can't see into:

```swift
@inline(never) func blackHole<T>(_ x: T) {}

for _ in 0..<1000 {
    blackHole(slugify(title))
}
```

XCTest provides a more careful harness: its `measure` method runs a block several times and reports the mean and the spread, and on Apple platforms Xcode can record a baseline and flag later runs that regress:

```swift
import XCTest

final class SlugBenchmarks: XCTestCase {
    func testPerformance() {
        measure {
            for _ in 0..<1000 { _ = slugify("Crème brûlée: A Surprisingly Long Title!") }
        }
    }
}
```

For systematic work, the `package-benchmark` package is the community standard. Benchmarks live in their own targets and run with `swift package benchmark`; each is executed many times, and the results are reported as percentiles across several metrics. Besides time, it can count heap allocations and retain/release operations. In Swift, those counts are often the most informative numbers, since a surprising share of run time goes to memory management:

```swift
// Benchmarks/Slug/Slug.swift
import Benchmark
import Slug

let benchmarks = {
    Benchmark("slugify") { benchmark in
        for _ in benchmark.scaledIterations {
            blackHole(slugify("Crème brûlée: A Surprisingly Long Title, With Punctuation!"))
        }
    }
}
```

```
$ swift package benchmark
Slug:slugify
╒═════════════════════════╤═════════╤═════════╤═════════╤═════════╤═════════╤═════════╤═════════╤═════════╕
│ Metric                  │      p0 │     p25 │     p50 │     p75 │     p90 │     p99 │    p100 │ Samples │
╞═════════════════════════╪═════════╪═════════╪═════════╪═════════╪═════════╪═════════╪═════════╪═════════╡
│ Time (wall clock) (ns)  │    2140 │    2210 │    2260 │    2330 │    2410 │    2780 │    6950 │   10000 │
│ Malloc (total)          │      14 │      14 │      14 │      14 │      14 │      14 │      14 │   10000 │
...
```

(The figures are illustrative.) Fourteen allocations for one short title suggests room for improvement: every word becomes a separate `String`, then an array element, before being joined.

Whatever the tool, the method is the same. Decide what you want to know before you measure. Measure optimized builds on realistic inputs. Change one thing at a time. Compare before and after on the same machine. And be suspicious of results that look too good.

**Exercise 11.7:** Write a faster `slugify` that builds its result in a single `String`, appending a hyphen only when a new word starts, instead of collecting words in an array. Benchmark both versions on short and long titles, counting allocations as well as time. Make sure the parameterized and property-based tests still pass.

**Exercise 11.8:** Benchmark `RingBuffer` against an array used as a queue (`append` plus `removeFirst`) for keeping the last *n* of a long stream of values, at several sizes of *n*. Where does each one win?

**Exercise 11.9:** Write a merge sort that recurses on `ArraySlice`s and another that copies into new arrays at each level. Measure the difference in a release build.

## 11.5. Profiling

A benchmark tells you *how* fast something is. When a whole program is too slow, the first question is *where* the time goes, and intuition is notoriously bad at answering it. Programmers routinely spend effort speeding up code that accounts for 1% of the run time while the real cost hides somewhere they never thought to look. A *profiler* answers the question with data. It samples what the program is doing many times a second while it runs, then aggregates the samples into a report of which functions, and which call paths, account for the time.

On Apple platforms, the profiler is *Instruments*, part of Xcode. Its Time Profiler records stack samples from every thread and presents them as a call tree, with time attributed both to each function's own code and to everything it calls. Other instruments track allocations and leaks, file and network activity, thread scheduling, and energy. For Swift programs, the *Allocations* instrument and the *Swift Concurrency* instruments, which chart tasks and actors over time, are especially useful.

On Linux, the standard tool is `perf`, usually paired with a flame-graph renderer:

```
$ swift build -c release
$ perf record -g .build/release/slug-bench
$ perf report
# Overhead  Command  Shared Object   Symbol
# ........  .......  ..............  ...........................................
    31.42%  bench    libswiftCore.so [.] swift_retain
    22.87%  bench    libswiftCore.so [.] swift_release
    11.30%  bench    libswiftCore.so [.] swift_allocObject
     9.05%  bench    bench           [.] $s4Slug7slugifyyS2SF
```

The last symbol is a *mangled* Swift name, which `swift demangle` turns back into `Slug.slugify(Swift.String) -> Swift.String`. The shape of this (illustrative) profile is common in Swift: the biggest costs aren't in our code at all but in the runtime's reference-counting and allocation routines, called on our behalf. That's a strong hint to look at how the code creates and copies values. Other useful Linux tools include `valgrind --tool=callgrind`, `heaptrack` for allocation profiling, and `swift-inspect` for examining the heap and metadata of a running Swift process.

Profile an optimized build with debug information, which SwiftPM's release configuration includes by default, so that the profiler can map machine code back to source lines. And profile a realistic workload: a profile of a toy input mostly measures startup.

### 11.5.1. Common Culprits

Some causes of poor performance turn up in Swift profiles again and again. Knowing them in advance makes profiles easier to read.

*Retain/release traffic.* Copying a value that contains a reference (a class instance, a `String`, an array, a closure) usually means atomically incrementing a reference count, and later decrementing it. In hot loops, this adds up. `borrowing` and `consuming` parameter modifiers, value types without references, and avoiding needless copies all help.

*Existentials.* A call through `any SomeProtocol` goes through a table lookup, can't be inlined, and may involve boxing a large value on the heap. Generics (`some SomeProtocol`, or an explicit type parameter) give the optimizer a concrete type to specialize for.

*Unspecialized generics.* A generic function in another module can be specialized for your types only if its body is visible to the optimizer, which requires `@inlinable`. Otherwise it runs a general version that consults type metadata at run time. Profile entries such as `swift_getGenericMetadata` and `swift_conformsToProtocol` are the telltale signs.

*Character-level string processing.* Iterating over a `String`'s `Character`s requires finding grapheme-cluster boundaries, which is real work. For ASCII-heavy data, the `utf8` view can be many times faster (Section 3.5).

*Accidental copies.* A copy-on-write collection that's mutated while some other reference to its storage exists gets copied in full. If that happens inside a loop, a linear algorithm silently becomes quadratic. The usual cause is a leftover second reference: a local copy still in scope, or a capture in a closure.

*Blocking in tasks.* A blocking call inside a task holds one of the thread pool's few threads (Section 9.8). A profile in which threads sit idle in `read` or `usleep` while tasks wait to run points to this.

**Exercise 11.10:** Profile the `anagrams` program of Section 4.3 on a large file. Where does the time go? Add a fast path for ASCII input that works on the `utf8` view, and measure the gain.

**Exercise 11.11:** Write a loop that accidentally copies an array on every iteration, as described under "Accidental copies" above. Confirm with a profiler that the copying dominates, then fix it.

## 11.6. Documentation Examples

Some of the most useful documentation consists of examples: a few lines showing a call and its result communicate faster than paragraphs of description. The trouble with examples in documentation is that nothing checks them, so they drift out of date as the code changes. Swift's tools offer a few ways to keep them honest.

DocC (Section 10.7) renders code blocks in documentation comments with syntax highlighting:

```swift
/// Converts a title to a URL slug.
///
/// ```swift
/// slugify("Hello, World!")  // "hello-world"
/// slugify("Crème brûlée")  // "creme-brulee"
/// ```
public func slugify(_ title: String) -> String { ... }
```

DocC doesn't run code in comments. It can, however, include code from separate source files in tutorials and articles, and those files are compiled, so at least they can't fall out of date with the API. For behavior, the practical approach is to mirror each documented example in a test that bears its name:

```swift
@Test("Documented examples of slugify")
func documentedExamples() {
    #expect(slugify("Hello, World!") == "hello-world")
    #expect(slugify("Crème brûlée") == "creme-brulee")
}
```

If someone changes the behavior, the test fails and points at the example that needs updating.

A related technique is *snapshot testing*, offered by packages such as `swift-snapshot-testing`. A snapshot test records a function's output the first time it runs and compares future runs with that recording. It's well suited to large outputs, such as rendered HTML, pretty-printed syntax trees, or a command's help text, where writing the expected value by hand would be impractical. The first snapshot deserves careful review, because from then on the test will defend it, right or wrong.

Examples in documentation, then, serve three purposes: they explain, faster than prose can; they specify, when tests keep them accurate; and they invite experimentation. A reader can paste an example into the Swift REPL or a playground, change it, and see what happens, which is how many people learn a language best.

**Exercise 11.12:** Add documentation comments with examples to the `RingBuffer` of Section 6.5, and a test for each example.

**Exercise 11.13:** Write a snapshot-style test for the `Expr` pretty-printer of Exercise 7.13 that stores the expected output in a file next to the test. Make updating the stored file a deliberate act, for instance by requiring an environment variable to be set.
