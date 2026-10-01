# 9. Concurrency with Shared State

Chapter 8 kept its tasks out of each other's way. Each one owned its data, and values passed between tasks through task results and streams. That style is the easiest to get right, and it should be your first choice. But it isn't always practical. A cache that many requests consult, a counter that many handlers update, a model object that a user interface and a network layer both touch: some state really is shared.

This chapter is about that harder case. We'll look at what goes wrong when tasks share mutable state carelessly, at the three basic strategies for preventing it, and at the tools Swift provides for each: immutability, mutexes and atomics, and actors. Along the way we'll see how the Swift 6 compiler turns the most dangerous class of concurrency bug, the data race, from something you hunt for at run time into something it refuses to compile. We'll end with a closer look at how tasks map onto operating-system threads, which matters for performance and occasionally for correctness.

## 9.1. Data Races and `Sendable`

Within a single task, statements happen in the order the code says. Across tasks, there's no such order. If task A executes statement *a* and task B executes statement *b*, then, unless something links them (one task waiting for the other, say, or both taking the same lock), *a* might happen before *b*, after it, or at effectively the same moment. Most concurrency bugs come down to code that quietly assumes an order that nothing guarantees.

Call a function *concurrency-safe* if it still behaves correctly when several tasks call it at overlapping times, without the callers doing anything special to coordinate. A type is concurrency-safe if all of its operations are. A great deal can make code unsafe in this sense, including deadlocks and starvation, but the most common culprit by far is the *race condition*: a bug that shows up only for certain interleavings of the tasks' operations. Race conditions are notoriously hard to track down, because the bad interleaving may be rare. A program can pass every test for months and then fail under heavy load, on a machine with more cores, or after an unrelated change shifts the timing.

Here's a small example. A box office sells tickets for a concert with 100 seats:

```swift
// Package boxoffice sells seats for a single concert.
var seatsLeft = 100

func sell(_ n: Int) {
    seatsLeft = seatsLeft - n
}

func available() -> Int {
    seatsLeft
}
```

Called from one task, these functions are obviously correct. Now suppose there are two ticket windows, north and south, each served by its own task, and they happen to sell tickets at the same moment:

```swift
Task { sell(2) }  // north window
Task { sell(3) }  // south window
```

The statement `seatsLeft = seatsLeft - n` looks like one step but is really two: read the current value, then write a new one. Write those steps as N-read, N-write, S-read, and S-write. Here's one perfectly possible interleaving:

```
step       seatsLeft   note
N-read        100      north computes 100 - 2 = 98
S-read        100      south computes 100 - 3 = 97
S-write        97
N-write        98      south's sale has vanished
```

Five tickets were sold, but the count dropped by only two. Run this long enough and the concert will be oversold.

That is a *data race*: two tasks accessing the same memory at overlapping times, with at least one of them writing. The damage above is a lost update, which is bad enough. With more complex values it can be worse. Two tasks appending to the same array at once might both decide the buffer is uniquely referenced and both modify it in place, or both reallocate it; a dictionary can be left with a corrupt table. The outcome is *undefined behavior*: anything from a wrong answer to a crash to silent memory corruption.

In most languages, preventing data races is left to the programmer's discipline, aided at best by a run-time detector. Swift 6 takes responsibility for it. The box-office code above doesn't compile in the Swift 6 language mode:

```
error: var 'seatsLeft' is not concurrency-safe because it is nonisolated global shared mutable state
var seatsLeft = 100
    ^
note: convert 'seatsLeft' to a 'let' constant to make 'Sendable' shared state immutable
note: add '@MainActor' to make var 'seatsLeft' part of global actor 'MainActor'
```

The compiler's notes hint at the ways out. Every strategy for avoiding data races is a variation on one of three ideas.

**Don't mutate.** Data that never changes can be read by any number of tasks at once with no risk at all. In Swift, a `let` of a value type is immutable all the way down, so a global constant table is safe to share:

```swift
let statusReasons: [Int: String] = [
    200: "OK",
    301: "Moved Permanently",
    404: "Not Found",
    500: "Internal Server Error",
]

/// Concurrency-safe.
func reason(for status: Int) -> String {
    statusReasons[status] ?? "Unknown"
}
```

**Don't share.** If only one task ever touches a piece of mutable state, there's nobody to race with. We relied on that throughout Chapter 8: the crawler's `seen` set lived in the parent task, and the chat server's member list lived in the room task. Other tasks that wanted the state changed sent requests instead of touching it. Swift's *actors*, introduced in Section 9.3, turn this arrangement into a language construct.

**Take turns.** If state must be both shared and mutable, make sure only one task accesses it at a time. That's *mutual exclusion*, the subject of Section 9.2.

### 9.1.1. The `Sendable` Protocol

To check for races, the compiler needs to know which values are safe to hand from one task to another. That knowledge is captured by the protocol `Sendable`. It has no methods. It's a *marker* that the compiler checks according to a few rules:

- A struct or enum is `Sendable` when everything it stores is `Sendable`. For types that aren't `public`, the compiler works this out by itself. Numbers, strings, and the other basic types are `Sendable`, and so are arrays, dictionaries, and sets of `Sendable` elements.
- Every actor is `Sendable`, because it guards its own state.
- A `final` class is `Sendable` if all its stored properties are `let`s of `Sendable` type. A class that guards mutable state with its own lock may claim `@unchecked Sendable`, which tells the compiler to take the programmer's word for it.
- A closure is `Sendable` when its type is marked `@Sendable`, which requires that it capture only `Sendable` values and never capture a variable that could be mutated.

The compiler applies these rules wherever a value moves between concurrent contexts: when a closure is handed to `Task` or `addTask`, when an argument is passed to an actor or a result comes back from one, when a value is yielded into an `AsyncStream`. A closure that would mutate a shared variable from a child task, for instance, is rejected:

```swift
var total = 0
await withTaskGroup(of: Void.self) { group in
    group.addTask {
        total += 1  // error: mutation of captured var 'total' in concurrently-executing code
    }
}
```

These rules would be painfully strict if they applied to every value equally, so Swift 6 refines them with *region-based isolation*. A value that isn't `Sendable` may still cross into another task if the compiler can see that the original owner never uses it again; the value is transferred, not shared. A parameter declared `sending` promises the same thing in a function's signature. The question the compiler asks, in effect, is not "could this value be shared?" but "is it actually being shared?"

## 9.2. Mutual Exclusion: `Mutex`

A *mutex* (mutual-exclusion lock) is the classic tool for letting several tasks share mutable state safely, by ensuring that only one of them uses it at a time. Swift 6 added one to the standard library's `Synchronization` module.

Swift's `Mutex` differs from the locks in most languages in one important way. In C or Go, a lock and the data it protects are separate things that the programmer promises to use together. A Swift `Mutex` *holds* its data. You can't reach the data except by calling `withLock`, which acquires the lock, lends the data to a closure as an `inout` parameter, and releases the lock when the closure is done:

```swift
// swiftpl/ch9/boxoffice2
import Synchronization

let seatsLeft = Mutex(100)

func sell(_ n: Int) {
    seatsLeft.withLock { $0 -= n }
}

func available() -> Int {
    seatsLeft.withLock { $0 }
}
```

Two familiar mistakes become impossible. You can't read or write the seat count without holding the lock, because there's no way to name it outside a `withLock` closure. And you can't forget to unlock, because the lock is released automatically when the closure finishes, whether it returns normally or throws.

The body of a `withLock` closure is a *critical section*. While one task is inside it, any other task calling `withLock` on the same mutex waits its turn.

`Mutex` is `Sendable`, so a global `let` holding one may be used from any task. It's also *noncopyable* (`~Copyable`): copying a lock would produce two locks guarding what's supposed to be one piece of state, which makes no sense. In practice that means a mutex lives in a global or static `let`, or in a stored `let` of a final class:

```swift
final class BoxOffice: Sendable {
    private let seatsLeft = Mutex(100)

    func sell(_ n: Int) {
        seatsLeft.withLock { $0 -= n }
    }
}
```

Since the class stores nothing but a `let` of a `Sendable` type, the compiler can verify its `Sendable` conformance directly; no `@unchecked` is needed.

### 9.2.1. Atomicity and Critical Sections

Customers don't buy tickets blindly. They ask for a number of seats, and the sale goes through only if that many are left. A first attempt uses the two functions we already have:

```swift
// NOTE: not atomic!
func book(_ n: Int) -> Bool {
    guard available() >= n else {
        return false  // sold out
    }
    sell(n)
    return true
}
```

Each function call takes the lock and releases it, so there's no data race. But there's still a race *condition*. Suppose three seats remain and both windows try to book two. Both call `available()`, both see three, both decide to go ahead, and both call `sell(2)`. The count ends at −1. The check and the update are each protected, but nothing protects the *pair*, so another task can slip in between them. An operation like `book` needs to be *atomic*: it must appear to happen all at once, with no other task able to observe or interfere with an intermediate state.

The fix is to hold the lock across both steps, which `withLock` makes easy because the closure has the state in hand:

```swift
func book(_ n: Int) -> Bool {
    seatsLeft.withLock { seats in
        guard seats >= n else {
            return false  // sold out
        }
        seats -= n
        return true
    }
}
```

It's tempting to write this by calling `available()` or `sell(_:)` from inside the closure. Don't. A Swift `Mutex` isn't *reentrant*, so a task that calls `withLock` on a mutex it already holds will deadlock waiting for itself (or trap, if the runtime detects it). The usual remedy is to split each operation in two: a private worker that assumes the lock is already held, and a public wrapper that takes the lock and calls the worker. In Swift, the worker can take the protected value as an `inout` parameter, which turns "the caller must hold the lock" from a comment into a fact, since only code inside `withLock` has the value to pass:

```swift
func sell(_ n: Int) {
    seatsLeft.withLock { sell(n, from: &$0) }
}

/// Requires the lock to be held; the caller proves it by passing &seats.
private func sell(_ n: Int, from seats: inout Int) {
    seats -= n
}
```

Keep the mutex and everything it protects private. If the protected state can be reached from outside the type or file, so can ways to break its invariants. Encapsulation (Section 6.6) is as much a concurrency tool as an organizational one.

One more rule: a `withLock` closure is synchronous, so you can't `await` while holding a mutex. That's a feature. A task that suspended while holding a lock could keep every other task waiting for an unbounded time, and if the work needed to wake it up were queued behind the very tasks blocked on the lock, the program would never make progress. When exclusive access really must span an `await`, the tool is an actor, used with the care described in Section 9.4.

### 9.2.2. Atomics

When the shared state is a single number or flag, even a mutex is heavier than necessary. `Synchronization` also provides `Atomic`, which updates integers, Booleans, and a few other simple types using the processor's lock-free atomic instructions:

```swift
import Synchronization

let requestsServed = Atomic<Int>(0)

func handle() {
    requestsServed.add(1, ordering: .relaxed)
    // ...
}

print(requestsServed.load(ordering: .relaxed))
```

Every atomic operation names a *memory ordering*, which controls what the operation implies about the visibility of *other* reads and writes around it. `.relaxed` promises only that this one operation is indivisible, which is all a statistics counter needs. The stronger orderings (`.acquiring`, `.releasing`, `.acquiringAndReleasing`, `.sequentiallyConsistent`) are for building synchronization of your own, and they are difficult to use correctly. Unless profiling shows that a lock is a bottleneck, prefer a `Mutex` or an actor.

## 9.3. Actors

An *actor* is a reference type, declared much like a class, whose mutable state can be touched by only one task at a time. You can think of it in two ways, and both are useful. It's an object with a built-in lock that every outside call acquires automatically. It's also a task that owns some state and serves requests from others, like the room task in Section 8.10, except that the requests are ordinary method calls instead of hand-built event enums.

Here's the box office as an actor:

```swift
// swiftpl/ch9/boxoffice3
actor BoxOffice {
    private var seatsLeft = 100

    func sell(_ n: Int) {
        seatsLeft -= n
    }

    func available() -> Int {
        seatsLeft
    }

    func book(_ n: Int) -> Bool {
        guard seatsLeft >= n else {
            return false
        }
        seatsLeft -= n
        return true
    }
}
```

Inside the actor, nothing special is required. Methods read and write `seatsLeft` directly and call one another freely; `book` could call `available()` and `sell(_:)` without any risk of the deadlock that a mutex would cause. All of this code is *isolated* to the actor, meaning it only ever runs while the actor is serving one particular caller.

Outside the actor, things are different. Every call into it is written with `await`, because the actor may be busy serving someone else:

```swift
// swiftpl/ch9/boxoffice3 (continued)
let office = BoxOffice()

await withTaskGroup(of: Bool.self) { group in
    group.addTask { await office.book(2) }  // north window
    group.addTask { await office.book(3) }  // south window
}
print(await office.available())  // "95"
```

A caller waiting for an actor doesn't block a thread. It suspends, and its thread goes off to run other tasks. The actor works through its queued calls one at a time, and each caller resumes when its call completes.

The compiler polices the boundary. Outside code can't reach `seatsLeft` at all, and anything passed into or returned out of the actor must be `Sendable`. That last rule stops a subtle leak: if an actor could return a reference to a mutable, non-`Sendable` object it owned, other tasks could mutate the actor's state behind its back.

Parts of an actor that don't depend on its mutable state can be declared `nonisolated` and used without `await`. Immutable `let` properties of `Sendable` type are already accessible that way:

```swift
actor BoxOffice {
    let venue: String  // a Sendable let: readable from anywhere
    nonisolated var description: String { "Box office at \(venue)" }
    // ...
}
```

### 9.3.1. The Main Actor and Global Actors

Some state belongs not to one object but to an entire subsystem. The standard example is the user interface. Every major UI toolkit insists that its objects be used only from the main thread. Swift models that rule with a *global actor*, `MainActor`, which runs everything isolated to it on the main thread. Marking a declaration `@MainActor` places it under that rule:

```swift
@MainActor
final class ViewModel {
    var items: [String] = []

    func load() async {
        let fetched = await fetchItems()  // the fetch runs elsewhere; we suspend here
        items = fetched  // back on the main actor when we resume
    }
}
```

Every method and property of `ViewModel` is now main-actor-isolated, so code running elsewhere must use `await` to reach it. Top-level code in `main.swift` and the `main` method of an `@main` type run on the main actor too.

The `@globalActor` attribute lets you declare global actors of your own, for other subsystems that need a single serial context. It's seldom needed.

## 9.4. Actor Reentrancy

An actor guarantees that two tasks never execute its code *simultaneously*. It does *not* guarantee that a method runs from start to finish without interruption. Whenever an actor method reaches an `await`, it suspends, and while it's suspended the actor is free to serve other callers, who may change its state. When the first method resumes, the world may have moved on. This property is called *actor reentrancy*, and understanding it is the key to writing correct actors.

Let's add payment to the box office. Before seats are sold, the customer's card must be authorized by a remote payment service, which takes a network round trip:

```swift
extension BoxOffice {
    // NOTE: buggy!
    func book(_ n: Int, paidBy card: Card) async throws -> Bool {
        guard seatsLeft >= n else {
            return false
        }
        try await authorize(card, amount: n * ticketPrice)  // suspension point!
        seatsLeft -= n
        return true
    }
}
```

This is the not-atomic `book` of Section 9.2.1 all over again. The method checks the seat count, then suspends while the payment service thinks. During that pause, other `book` calls run, pass the same check against the same count, and suspend in turn. When the authorizations come back, every one of them subtracts its seats, and the show is oversold. The compiler can't help here, because nothing is a data race: every access to `seatsLeft` happens on the actor. The bug is in the logic, which assumed the check still held after an `await`.

The discipline that avoids it is simple to state: *at every `await`, the actor's invariants must hold, and anything you learned before the `await` may be out of date after it.* For a booking, the natural fix is the one real ticket systems use. Hold the seats *before* suspending, then release them if the payment fails:

```swift
extension BoxOffice {
    func book(_ n: Int, paidBy card: Card) async throws -> Bool {
        guard seatsLeft >= n else {
            return false
        }
        seatsLeft -= n  // hold the seats before we suspend
        do {
            try await authorize(card, amount: n * ticketPrice)
        } catch {
            seatsLeft += n  // payment failed: release the hold
            throw error
        }
        return true
    }
}
```

Now the check and the update happen together, with no `await` between them, so no other booking can see the old count. While a payment is pending, the held seats are unavailable to anyone else, which is exactly the behavior customers expect.

Another common fix is to do the asynchronous work first and the check-and-update afterward, so that all the state changes happen in one uninterrupted stretch. Which one fits depends on the problem. What doesn't change is that each `await` in an actor method is a place where you must stop and ask what could have changed.

If reentrancy causes this much trouble, why allow it? Because a non-reentrant actor would be worse. It would stay locked for the entire duration of every call, including every network wait inside it, so one slow request would stall all other users of the actor. And two non-reentrant actors that called each other would deadlock immediately. Reentrancy keeps actors responsive and deadlock-free; the price is the discipline above.

**Exercise 9.1:** Start 1,000 tasks in a task group, each trying to book one seat from a `Mutex`-protected count of 100, with the check and the update inside a single `withLock`. Confirm that exactly 100 bookings succeed. Then split the check and the update into two `withLock` calls, and see whether you can make the program oversell.

**Exercise 9.2:** Implement the other fix described above for the reentrant `book`: authorize the payment first, then check and update the seat count with no `await` in between. What must the method do when the seats are gone by the time the payment succeeds? Which fix would a customer prefer?

**Exercise 9.3:** Rewrite the actor `BoxOffice` of Section 9.3 as a `final class` that keeps its state in a `Mutex`. How do its callers change? Why can't the payment-taking `book` of this section be written the same way, with the `authorize` call inside `withLock`?

## 9.5. Lazy Initialization

Some values are expensive to compute and not always needed: a large lookup table, a parsed configuration file, a compiled regular expression. It's better to create such a value the first time it's used than to pay for it at startup.

Lazy initialization is a classic source of races in other languages. The obvious code, "if the table is nil, build it," lets two tasks both see nil and both build it, or worse, lets one task see a half-built table. Swift sidesteps the problem. As noted in Section 2.6, every global variable and every static property is initialized lazily, on first access, and the runtime guarantees that initialization happens exactly once, even when several threads arrive at the same moment:

```swift
// in a file other than main.swift
let statusReasons: [Int: String] = loadStatusTable()  // reads a large data file

/// Concurrency-safe.
func reason(for status: Int) -> String {
    statusReasons[status] ?? "Unknown"
}
```

The first call to `reason(for:)` runs `loadStatusTable()`; any task that arrives while that's in progress waits for it; every later call simply reads the finished dictionary. Static properties work the same way, which is why `static let shared = ...` is the conventional way to create a lazily initialized singleton.

An instance property declared `lazy var` is also initialized on first use, but it offers no such guarantee. Two tasks reading it at once could both run its initializer. Since a `lazy var` is mutable state, the usual rules apply, and the compiler won't let an object that has one be shared between tasks unless something protects it.

Neither mechanism works when building the value requires `await`, such as fetching it from a server, since a global's initializer must be synchronous. For that, keep a `Task` that produces the value inside an actor. Section 9.7 shows how.

## 9.6. Strict Concurrency Checking and the Thread Sanitizer

Concurrency bugs are hard to find by testing, which is why Swift tries to stop them at compile time. In the Swift 6 language mode, the default for packages whose manifest declares `swift-tools-version: 6.0` or later, any code that could race is a compile error. Code built in the Swift 5 language mode can opt in to the same checks as warnings, which makes it possible to migrate a large codebase one module at a time:

```swift
// In Package.swift
.target(
    name: "Legacy",
    swiftSettings: [
        .swiftLanguageMode(.v5),
        .enableUpcomingFeature("StrictConcurrency"),
    ]
)
```

The checks can be overridden where the programmer knows something the compiler can't prove. `@unchecked Sendable` vouches for a class that synchronizes itself. `nonisolated(unsafe)` exempts a global variable. `@preconcurrency import` silences diagnostics about a library that predates `Sendable`, and `MainActor.assumeIsolated` asserts that some code already runs on the main thread. Each of these is a promise. If the promise is false, the race the compiler would have caught is back.

For code that uses these escape hatches, and for C and C++ code that the Swift compiler can't check at all, there's a dynamic tool: the *Thread Sanitizer* (TSan). A program built with TSan records, for every memory access, which thread made it and which synchronization operations preceded it, and reports any pair of conflicting accesses that weren't ordered by synchronization:

```
$ swift build --sanitize=thread
$ swift test --sanitize=thread
```

Here's a deliberate lie to the compiler, and the result of running it under TSan:

```swift
nonisolated(unsafe) var hits = 0

await withTaskGroup(of: Void.self) { group in
    for _ in 0..<2 {
        group.addTask { hits += 1 }
    }
}
```

```
==================
WARNING: ThreadSanitizer: Swift access race (pid=48233)
  Modifying access of Swift variable at 0x0001049c8030 by thread T2:
    #0 closure #1 in closure #1 in race main.swift:6
  Previous modifying access of Swift variable at 0x0001049c8030 by thread T1:
    #0 closure #1 in closure #1 in race main.swift:6
  Location is global 'hits' of size 8 at 0x0001049c8030
==================
```

(The addresses, thread numbers, and process ID will differ on your machine.) The report pinpoints both accesses and the variable involved. TSan has limits: it finds only races that actually happen during the run being observed, so it's no better than the tests that drive it, and it slows the program considerably. It's a debugging and testing tool, not something to ship. Used together with the compiler's static checks, though, it closes nearly every gap.

## 9.7. Example: Concurrent Non-Blocking Cache

We'll finish the chapter's examples with a problem that shows up in nearly every server: *memoization*, caching the result of an expensive function so that repeated calls with the same argument don't redo the work. Our target is a cache that many tasks can use at once, that never makes one key wait on another key's computation, and that never computes the same key twice, even when several tasks ask for it at the same moment.

As the expensive function, we'll use an HTTP fetch:

```swift
@Sendable func fetchBody(_ url: String) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)
    return data
}
```

The simplest cache is a class with a dictionary. It stores a `Result` (Section 5.10) for each key, so that a failure is remembered as well as a success:

```swift
// swiftpl/ch9/memo1
/// A Memo caches the results of calling a function.
final class Memo<Key: Hashable, Value> {
    typealias Function = (Key) async throws -> Value

    private let f: Function
    private var cache: [Key: Result<Value, any Error>] = [:]

    init(_ f: @escaping Function) {
        self.f = f
    }

    // NOTE: not concurrency-safe!
    func get(_ key: Key) async throws -> Value {
        if let result = cache[key] {
            return try result.get()
        }
        let result: Result<Value, any Error>
        do {
            result = .success(try await f(key))
        } catch {
            result = .failure(error)
        }
        cache[key] = result
        return try result.get()
    }
}
```

Used from a single task, it does its job. Here's a driver that fetches a list of URLs containing repeats and reports how long each `get` took:

```swift
let m = Memo(fetchBody)
for url in urlsToFetch() {
    let start = ContinuousClock.now
    do {
        let value = try await m.get(url)
        print("\(url), \(ContinuousClock.now - start), \(value.count) bytes")
    } catch {
        print(error)
    }
}
```

A typical run shows the cache working: repeats come back almost instantly. (Your timings and sizes will differ.)

```
https://www.swift.org, 0.175026 seconds, 21342 bytes
https://forums.swift.org, 0.406393 seconds, 86104 bytes
https://swiftpackageindex.com, 0.633774 seconds, 148203 bytes
https://www.swift.org, 0.000002 seconds, 21342 bytes
https://forums.swift.org, 0.000001 seconds, 86104 bytes
https://swiftpackageindex.com, 0.000001 seconds, 148203 bytes
```

But the fetches happen one after another, which wastes the opportunity to overlap them. So let's issue them all at once, one child task per URL:

```swift
await withTaskGroup(of: Void.self) { group in
    for url in urlsToFetch() {
        group.addTask {
            let start = ContinuousClock.now
            let value = try? await m.get(url)  // error: capture of non-sendable type 'Memo<String, Data>'
            print("\(url), \(ContinuousClock.now - start), \(value?.count ?? 0) bytes")
        }
    }
}
```

This doesn't compile, and that's the compiler doing its job. `Memo` is a class with an unprotected mutable dictionary, so concurrent calls to `get` would race on it. In a language without static race checking, this program would build, pass a casual test, and then corrupt its cache now and then in production. Here, we're forced to decide how the cache will be shared before the program can run at all.

### 9.7.1. A Cache with a Lock

The first fix that comes to mind is a mutex around the dictionary:

```swift
// swiftpl/ch9/memo2
final class Memo<Key: Hashable & Sendable, Value: Sendable>: Sendable {
    typealias Function = @Sendable (Key) async throws -> Value

    private let f: Function
    private let cache = Mutex<[Key: Result<Value, any Error>]>([:])

    init(_ f: @escaping Function) {
        self.f = f
    }

    func get(_ key: Key) async throws -> Value {
        if let result = cache.withLock({ $0[key] }) {
            return try result.get()
        }
        let result: Result<Value, any Error>
        do {
            result = .success(try await f(key))
        } catch {
            result = .failure(error)
        }
        cache.withLock { $0[key] = result }
        return try result.get()
    }
}
```

This compiles, and the concurrent version of the driver runs much faster than the sequential one. Look closely at how the lock is used, though. It's taken once to check the cache and again to store the result, and released in between, while `f` runs. It has to be: `withLock` won't let us `await` inside it, and holding the lock during a slow fetch would make every other key wait behind it anyway.

The gap between the check and the store has a cost. If two tasks ask for the same URL at nearly the same moment, both find nothing in the cache, both fetch the page, and both store the result. Nothing is corrupted, since the second store merely overwrites the first with an equal value, but the work is done twice. For an expensive function, that's exactly what a cache is supposed to prevent.

### 9.7.2. Caching the Work, Not the Answer

The way out is to cache something that exists *before* the answer does: the work in progress. A `Task` is a handle to a computation that may still be running. Any number of tasks can await its `value`, and all of them receive the same result when it's ready. So instead of storing results, we store tasks. The first caller for a key starts the computation and records the task; anyone who arrives later, whether the computation has finished or not, finds the task and waits on it.

For this to work, checking for an existing task and recording a new one must happen as a single atomic step. An actor gives us that:

```swift
// swiftpl/ch9/memo3
/// A Memo caches the results of calling a function,
/// computing each result at most once.
actor Memo<Key: Hashable & Sendable, Value: Sendable> {
    typealias Function = @Sendable (Key) async throws -> Value

    private let f: Function
    private var cache: [Key: Task<Value, any Error>] = [:]

    init(_ f: @escaping Function) {
        self.f = f
    }

    func get(_ key: Key) async throws -> Value {
        if let task = cache[key] {
            return try await task.value  // wait for the result
        }
        // This is the first request for this key.
        // Start computing it and record the task in the cache.
        let task = Task { [f] in try await f(key) }
        cache[key] = task
        return try await task.value
    }
}
```

Apply the reentrancy test from Section 9.4. The method's only suspension points are the two `await task.value` lines. The lookup, the creation of the task, and the store into `cache` all happen between suspension points, so no other call on the actor can run in the middle of them. A second request for the same key, even one that arrives while the first caller is still waiting, will find the stored task and wait on it rather than starting another fetch.

The actor doesn't become a bottleneck either. The expensive function runs in its own task, not on the actor; callers waiting for results are suspended, not holding the actor. So the actor is busy only for the brief bookkeeping in `get`, and fetches for different keys proceed fully in parallel. The capture list `[f]` copies the function into the new task, so that the task never needs to visit the actor to read the property.

Run the concurrent driver against this version and the pattern of the output changes:

```
https://www.swift.org, 0.176201 seconds, 21342 bytes
https://www.swift.org, 0.176310 seconds, 21342 bytes
https://forums.swift.org, 0.414012 seconds, 86104 bytes
https://forums.swift.org, 0.414104 seconds, 86104 bytes
https://swiftpackageindex.com, 0.640027 seconds, 148203 bytes
https://swiftpackageindex.com, 0.640190 seconds, 148203 bytes
```

Each duplicate request now takes as long as the original, because it waited for the original, but each URL was fetched exactly once.

This pattern of memoizing *tasks* rather than values is worth remembering. It's the standard Swift answer to many problems that look like "lazy initialization, but asynchronous": loading a configuration once, establishing a single shared connection, refreshing an authentication token without a stampede of concurrent refreshes.

**Exercise 9.4:** The standard library's `Result(catching:)` initializer takes a synchronous closure. Write an `async` overload, and use it to simplify `memo1` and `memo2`.

**Exercise 9.5:** If a caller of `get` is cancelled, it should stop waiting. Should cancelling one caller cancel the shared computation? What if other callers are still waiting for it? Implement the policy you choose.

**Exercise 9.6:** Give the cache a maximum size, evicting the least recently used entry when it's full. Be careful not to evict an entry whose task is still running.

**Exercise 9.7:** Change the cache so that failures aren't remembered, and a later `get` for the same key tries again.

## 9.8. Tasks and Threads

In Chapter 8 we said that a task could be thought of as a cheap thread. That approximation is good enough for writing correct programs. To write efficient ones, and to understand a few rules that would otherwise seem arbitrary, it helps to know how tasks actually run.

### 9.8.1. Stacks and Frames

An operating-system thread is reserved a fixed region of memory for its call stack when it's created: commonly 512 KB to 8 MB, mostly unused. A server that dedicated a thread to each of 50,000 idle connections would reserve gigabytes of address space for stacks that hold almost nothing. And the fixed size cuts the other way as well: a thread that recurses too deeply overflows its stack and crashes.

Tasks avoid the first problem by not owning stacks at all. A running `async` function borrows the stack of whatever thread is executing it, exactly as a synchronous function does. Only when it suspends does Swift copy the state it will need afterward (the live local variables and where to resume) into a heap-allocated *async frame* sized for that function. A suspended task therefore costs roughly what it has to remember, often a few hundred bytes. That's why programs can have so many of them.

### 9.8.2. Scheduling

Operating-system threads are scheduled by the kernel. Switching from one thread to another means entering the kernel, saving one set of registers, choosing a successor, and restoring another set, a *context switch* that costs on the order of microseconds and disturbs the processor's caches.

Swift tasks are scheduled by the Swift runtime, onto *executors*. Ordinary tasks use the *global concurrent executor*, which runs them on a pool of threads sized to the number of CPU cores. Each actor has a *serial executor* that runs one of its jobs at a time, usually on a thread borrowed from that same pool, and the main actor's executor is the main thread. When a task suspends, it simply returns control to its executor, which picks up the next ready job, all without involving the kernel. Switching tasks costs about as much as a function call.

The pool's fixed size is a deliberate choice, and it explains a rule we've met twice already. A task that *blocks* its thread (by sleeping with a blocking call, waiting on a semaphore, performing slow synchronous I/O, or running a long computation without ever suspending) takes that thread out of the pool for as long as it blocks. Block enough threads and every task in the program stalls, even ones with work ready to do. Some runtimes compensate by quietly adding threads when others block; Swift's doesn't, in exchange for predictable, low overhead. So Swift code should keep tasks moving: `await` rather than block, use asynchronous I/O where it matters, and have long computations call `await Task.yield()` now and then to give other tasks a turn.

### 9.8.3. Isolation and Where Code Runs

Which executor a piece of code runs on follows from its *isolation*, which is part of its declaration. A method of an actor runs on that actor; a `@MainActor` declaration runs on the main actor; a `nonisolated` asynchronous function runs on the global executor. A closure passed to `Task { }` takes on the isolation of the code that created it. That's why the progress reporter in Section 8.1 used `Task.detached`: top-level code runs on the main actor, and a plain `Task` would have queued the reporter behind the computation that was occupying the main thread.

Swift 6.2 revised some of these defaults to make concurrency easier to adopt. Under the "approachable concurrency" settings, which new app projects enable, a `nonisolated` asynchronous function runs on its *caller's* executor instead of hopping to the global one, and a function that should always run on the global executor is marked `@concurrent`. A module can also make `@MainActor` its default isolation, a good fit for app code that mostly lives on the main thread. These are per-module settings; packages written against the Swift 6.0 behavior keep compiling as before.

### 9.8.4. No Task Identity, but Task-Local Values

Threads usually have identities: a thread ID you can ask for and use as a key. Many systems build *thread-local storage* on top of that, giving each thread its own copy of certain variables.

Swift tasks expose no identity, and since a task can resume on a different thread after every suspension, thread-local storage doesn't mean anything to it. What Swift offers instead is *task-local values*. A task-local is declared with the `@TaskLocal` property wrapper, given a value for the duration of a closure with `withValue`, and visible to everything that runs within that scope, including child tasks:

```swift
enum Trace {
    @TaskLocal static var requestID: String = "none"
}

await Trace.$requestID.withValue("req-42") {
    await handle()  // handle, and any child tasks it starts, see Trace.requestID == "req-42"
}
```

Task-locals are well suited to context that cuts across many layers, such as request IDs for logging and tracing spans, where threading an extra parameter through every function would be noise. For anything that changes what a function computes, explicit parameters are clearer, and task-locals should be the exception.

That completes the language features for concurrency. The next two chapters turn to the tools around the language: organizing code into packages, and testing it.
