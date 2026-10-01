# 9. Concurrency with Shared State

In the previous chapter, we presented several programs that use tasks and asynchronous sequences to express concurrency in a direct and natural way. However, in doing so, we glossed over a number of important and subtle issues that programmers must bear in mind when writing concurrent code.

In this chapter, we'll take a closer look at the mechanics of concurrency. In particular, we'll point out some of the problems associated with sharing mutable state among multiple tasks, the analytical techniques for recognizing those problems, and the patterns for solving them. We'll also see how Swift 6 turns many of these problems from subtle run-time bugs into compile-time errors. Finally, we'll explain some of the technical differences between tasks and operating system threads.

## 9.1. Data Races and `Sendable`

In a sequential program, that is, a program with only one thread of control, the steps of the program happen in the familiar execution order determined by the program logic. For instance, in a sequence of statements, the first one happens before the second one, and so on. In a program with two or more tasks, the steps within each task happen in the familiar order, but in general we don't know whether an event *x* in one task happens before an event *y* in another task, or happens after it, or is simultaneous with it. When we cannot confidently say that one event happens before the other, then the events *x* and *y* are *concurrent*.

Consider a function that works correctly in a sequential program. That function is *concurrency-safe* if it continues to work correctly even when called concurrently, that is, from two or more tasks with no additional synchronization. We can generalize this notion to a set of collaborating functions, such as the methods and operations of a particular type. A type is concurrency-safe if all its accessible methods and operations are concurrency-safe.

There are many reasons a function might not work when called concurrently, including deadlock, livelock, and resource starvation. We don't have space to discuss all of them, so we'll focus on the most important one, the *race condition*.

A race condition is a situation in which the program does not give the correct result for some interleavings of the operations of multiple tasks. Race conditions are pernicious because they may remain latent in a program and appear infrequently, perhaps only under heavy load or when using certain compilers, platforms, or architectures. This makes them hard to reproduce and diagnose.

It is traditional to explain the seriousness of race conditions through the metaphor of financial loss, so we'll consider a simple bank account program.

```swift
// Package bank implements a bank with only one account.
var balance = 0

func deposit(_ amount: Int) {
    balance = balance + amount
}

func currentBalance() -> Int {
    balance
}
```

For such a trivial program, we can see at a glance that any sequence of calls to `deposit` and `currentBalance` will give the right answer, that is, `currentBalance` will report the sum of all amounts previously deposited. However, if we call these functions not in sequence but concurrently, `currentBalance` is no longer guaranteed to give the right answer. Consider the following two tasks, which represent two transactions on a joint bank account:

```swift
// Alice:
Task {
    deposit(200)  // A1
    print("=", currentBalance())  // A2
}

// Bob:
Task {
    deposit(100)  // B
}
```

Alice deposits $200, then checks her balance, while Bob deposits $100. Since the steps A1 and A2 occur concurrently with B, we cannot predict the order in which they happen. Intuitively, it might seem that there are only three possible orderings. But there's a fourth possibility, in which Bob's deposit occurs in the middle of Alice's deposit, after the balance has been read (`balance + amount`) but before it has been updated (`balance = ...`), causing Bob's transaction to disappear. This is because Alice's deposit operation A1 is really a sequence of two operations, a read and a write; call them A1r and A1w. Here's the problematic interleaving:

```
Data race
      0
A1r   0         ... = balance + amount
B   100
A1w 200         balance = ...
A2  "= 200"
```

After A1r, the expression `balance + amount` evaluates to 200, so this is the value written during A1w, despite the intervening deposit. The final balance is only $200. The bank is $100 richer at Bob's expense.

This program contains a particular kind of race condition called a *data race*. A data race occurs whenever two tasks access the same variable concurrently and at least one of the accesses is a write.

Things get even messier if the data race involves a variable of a type that is larger than a single machine word, such as a string, an array, or a dictionary. If two tasks append to the same array at the same time, both may decide to reallocate the buffer, and the copy-on-write logic, which relies on an accurate reference count, may be fooled. The result is *undefined behavior*: memory corruption, a crash, or silently wrong results.

In most languages, avoiding data races is the programmer's responsibility. In Swift 6, it's the compiler's. If we compile the bank program in the Swift 6 language mode, the compiler rejects it:

```
error: var 'balance' is not concurrency-safe because it is nonisolated global shared mutable state
var balance = 0
    ^
note: convert 'balance' to a 'let' constant to make 'Sendable' shared state immutable
note: add '@MainActor' to make var 'balance' part of global actor 'MainActor'
```

That's the core of Swift's *data-race safety*: it's a compile-time guarantee, not a run-time check, that no two tasks can access the same mutable state concurrently unless that state is protected.

There are three ways to avoid a data race, and Swift supports all of them.

The first way is not to write the variable. If shared state is immutable, many tasks can safely read it at once. Swift makes this the default: `let` constants of value types are deeply immutable, and a global `let` is safe to read from anywhere, as long as its type is `Sendable` (which we'll define in a moment).

```swift
let icons: [String: Image] = [
    "spades.png": loadIcon("spades.png"),
    "hearts.png": loadIcon("hearts.png"),
    "diamonds.png": loadIcon("diamonds.png"),
    "clubs.png": loadIcon("clubs.png"),
]

/// Concurrency-safe.
func icon(_ name: String) -> Image? { icons[name] }
```

The second way to avoid a data race is to avoid accessing the variable from multiple tasks. This is the approach taken by many of the programs in the previous chapter. For example, the parent task in the concurrent web crawler (Section 8.6) is the sole task that accesses the `seen` set, and the broadcaster task in the chat server (Section 8.10) is the only task that accesses the `clients` dictionary. These variables are *confined* to a single task. Since other tasks cannot access the variable directly, they must send requests to the confining task, which is what the Go proverb "Do not communicate by sharing memory; instead, share memory by communicating" means. Swift elevates this pattern to a language feature, the *actor*, which we'll see in Section 9.3.

The third way to avoid a data race is to allow many tasks to access the variable, but only one at a time. This approach is known as *mutual exclusion* and is the subject of the next section.

### 9.1.1. The `Sendable` Protocol

How does the compiler know what's safe? The key is a protocol called `Sendable`. A type that conforms to `Sendable` is one whose values can be safely shared between concurrent tasks. `Sendable` has no requirements; it's a *marker protocol* whose conformance the compiler checks:

- Value types (structs and enums) are `Sendable` if all their stored properties or associated values are `Sendable`. For non-public types, the compiler infers this automatically. All the basic types are `Sendable`, and the standard collections are `Sendable` when their elements are.
- Actors are always `Sendable`, since they protect their own state.
- A final class is `Sendable` only if all its stored properties are immutable `let`s of `Sendable` types, or if it protects its state with a lock and says so with `@unchecked Sendable` (an assertion that the compiler can't verify, so you must be sure).
- Function types are `Sendable` if they're marked `@Sendable`, which means they capture only `Sendable` values, and only immutably.

Whenever a value crosses a boundary between concurrent contexts (when it's captured by a closure passed to `Task` or `addTask`, passed to an actor, returned from one, or yielded into an `AsyncStream`), the compiler checks that it's `Sendable`. So a closure that captures a mutable variable can't be run in a child task:

```swift
var count = 0
await withTaskGroup(of: Void.self) { group in
    group.addTask {
        count += 1  // error: mutation of captured var 'count' in concurrently-executing code
    }
}
```

Swift 6 also performs *region-based isolation analysis*, which lets a non-`Sendable` value cross a boundary if the compiler can prove that the sender never uses it again. For instance, a freshly created non-`Sendable` object can be passed to a task, so long as the code that created it doesn't touch it afterwards. A parameter marked `sending` expresses the same guarantee in a function's signature. This makes the rules less restrictive than they might first appear: what matters is not whether a value *could* be shared, but whether it *is*.

## 9.2. Mutual Exclusion: `Mutex`

The `Synchronization` module of the standard library, introduced in Swift 6, provides a `Mutex` type. A mutex (short for mutual exclusion lock) guarantees that at most one task at a time can access the value it protects.

Unlike the mutexes of most languages, including Go's `sync.Mutex`, Swift's `Mutex` *contains* the state it protects. The only way to get at that state is to call `withLock`, which acquires the lock, passes the state to a closure as an `inout` parameter, and releases the lock when the closure returns:

```swift
// swiftpl/ch9/bank2
import Synchronization

let balance = Mutex(0)

func deposit(_ amount: Int) {
    balance.withLock { $0 += amount }
}

func currentBalance() -> Int {
    balance.withLock { $0 }
}
```

This design rules out the most common mutex mistakes. It's impossible to access the protected state without holding the lock, since there's no other way to reach it, and impossible to forget to release the lock, since `withLock` releases it automatically when the closure returns or throws. In Go, the convention is to put the guarded variables right after the mutex declaration and to use `defer mu.Unlock()`; in Swift, the type system enforces the equivalent.

The region of code between acquiring and releasing the lock, here the closure, is called a *critical section*. Each time a task calls `deposit` or `currentBalance`, it must wait for any other task inside a critical section on the same mutex to leave it.

`Mutex` is `Sendable`, so a global `let` holding one can be accessed from any task. It is a *noncopyable* type (`~Copyable`), since copying a lock would make no sense, so it is usually stored in a `let` at global scope, as a static property, or in a property of a final class:

```swift
final class Account: Sendable {
    private let balance = Mutex(0)

    func deposit(_ amount: Int) {
        balance.withLock { $0 += amount }
    }
}
```

Because the class's only stored property is a `let` of `Sendable` type, the compiler accepts the class's declaration of `Sendable` without `@unchecked`.

### 9.2.1. Atomicity and Critical Sections

Consider the `withdraw` function below. On success, it reduces the balance by the specified amount and returns `true`. But if the account holds insufficient funds for the transaction, `withdraw` restores the balance and returns `false`.

```swift
// NOTE: not atomic!
func withdraw(_ amount: Int) -> Bool {
    deposit(-amount)
    if currentBalance() < 0 {
        deposit(amount)
        return false  // insufficient funds
    }
    return true
}
```

This function eventually gives the correct result, but it has a nasty side effect. When an excessive withdrawal is attempted, the balance transiently dips below zero. This may cause a concurrent withdrawal for a modest sum to be spuriously rejected. So if Bob tries to buy a sports car, Alice can't pay for her morning coffee. The problem is that `withdraw` is not *atomic*: it consists of a sequence of three separate operations, each of which acquires and then releases the mutex lock, but nothing locks the whole sequence.

Ideally, `withdraw` should acquire the mutex lock once around the whole operation. With `withLock`, that's natural, because the closure has direct access to the protected state:

```swift
func withdraw(_ amount: Int) -> Bool {
    balance.withLock { balance in
        guard balance >= amount else {
            return false  // insufficient funds
        }
        balance -= amount
        return true
    }
}
```

Note that we can't implement `withdraw` by calling `deposit` from inside the closure, because Swift's `Mutex` is not *re-entrant*: calling `withLock` again on the same mutex from inside its own critical section would deadlock (or trap). Go's mutexes aren't re-entrant either. A common solution is to divide a function such as `deposit` into two: an unexported function that assumes the lock is already held and does the real work, and an exported function that acquires the lock before calling the first. In Swift, the first function takes the protected state as an `inout` parameter, which makes the assumption explicit:

```swift
func deposit(_ amount: Int) {
    balance.withLock { deposit(amount, to: &$0) }
}

/// Requires the caller to hold the lock (which it must, to have &balance).
private func deposit(_ amount: Int, to balance: inout Int) {
    balance += amount
}
```

Encapsulation (Section 6.6), by reducing unexpected interactions in a program, helps us maintain data structure invariants. For the same reason, encapsulation also helps us maintain concurrency invariants. When you use a mutex, make sure that both it and the state it guards are not exported, whether they are module-level variables or the properties of a type.

There's one more important restriction on `withLock`: its closure is synchronous. You can't `await` inside a critical section. This is deliberate. A task that suspends while holding a lock would keep other tasks waiting for an unpredictable length of time, and if the task that would eventually release the lock needed one of the threads blocked waiting for it, the program would deadlock. If you need to hold exclusive access across a suspension point, you need an actor, and you need to think carefully about reentrancy (Section 9.4).

### 9.2.2. Atomics

For a single integer counter or flag, a mutex is more than you need. The `Synchronization` module also provides `Atomic`, which performs lock-free operations on integers, booleans, and a few other types, using the processor's atomic instructions:

```swift
import Synchronization

let requests = Atomic<Int>(0)

func handle() {
    requests.add(1, ordering: .relaxed)
    // ...
}

print(requests.load(ordering: .relaxed))
```

Every atomic operation takes an explicit *memory ordering*, which specifies what guarantees it makes about the visibility of other memory operations. `.relaxed` makes no guarantees beyond the atomicity of the operation itself, which is fine for statistics counters; `.acquiring`, `.releasing`, and `.sequentiallyConsistent` give stronger guarantees. Atomics are subtle, and outside of performance-critical code, a `Mutex` or an actor is easier to get right.

## 9.3. Actors

An *actor* is a reference type, like a class, that protects its mutable state by allowing only one task at a time to execute its code. You can think of an actor as an object with its own private, built-in mutex, which is acquired automatically whenever one of its methods is called from outside. Or, equally, as a task that confines its state and serves requests from other tasks, like the broadcaster in Section 8.10, but with requests written as ordinary method calls.

Here's the bank as an actor:

```swift
// swiftpl/ch9/bank3
actor Bank {
    private var balance = 0

    func deposit(_ amount: Int) {
        balance += amount
    }

    func currentBalance() -> Int {
        balance
    }

    func withdraw(_ amount: Int) -> Bool {
        guard balance >= amount else {
            return false
        }
        balance -= amount
        return true
    }
}
```

Inside the actor, code accesses `balance` freely, and methods call each other as usual; `withdraw` could even call `deposit` directly. This is because the actor's code always runs *isolated* to the actor: only one task at a time can be executing it.

From outside the actor, every access to its methods and mutable properties must be made with `await`, because the caller may have to wait its turn:

```swift
let bank = Bank()

await withTaskGroup(of: Void.self) { group in
    group.addTask { await bank.deposit(200) }  // Alice
    group.addTask { await bank.deposit(100) }  // Bob
}
print(await bank.currentBalance())  // "300"
```

The compiler enforces *actor isolation*. Code outside the actor can't touch `balance` directly at all, and code that calls an actor method must be asynchronous, so that it can suspend if the actor is busy. Meanwhile, a waiting caller doesn't block a thread; it suspends, freeing its thread to run other tasks, and the actor runs the queued calls one at a time.

An actor's methods take and return values across an *isolation boundary*, so the compiler checks that the arguments and results are `Sendable`. An actor can't return a reference to a non-`Sendable` object it owns, since that would let other tasks modify the actor's state without isolation.

Members of an actor that don't touch its mutable state can be marked `nonisolated`, which lets them be called synchronously from anywhere:

```swift
actor Bank {
    let name: String  // immutable lets of Sendable type can be read without await
    nonisolated var description: String { "Bank \(name)" }
    // ...
}
```

### 9.3.1. The Main Actor and Global Actors

Some state isn't owned by a single object but by a whole subsystem. The most important example is the user interface: on every platform, UI objects must be accessed only from the main thread. Swift represents this with a *global actor*, `MainActor`, whose executor is the main thread. Any declaration can be isolated to it with the `@MainActor` attribute:

```swift
@MainActor
final class ViewModel {
    var items: [String] = []

    func load() async {
        let fetched = await fetchItems()  // runs elsewhere; we suspend here
        items = fetched  // back on the main actor
    }
}
```

All methods and properties of `ViewModel` are now isolated to the main actor, and calls from other isolation domains need `await`. Top-level code in `main.swift` also runs on the main actor, as does an `@main` type's `main` method.

You can define your own global actors for other subsystems with `@globalActor`, though it's rarely necessary.

## 9.4. Actor Reentrancy

Actors make it impossible for two tasks to run the actor's code *at the same time*. But they don't make an actor method atomic if it contains an `await`. Whenever an actor method suspends, the actor is free to run other calls, which may change its state. This is called *actor reentrancy*, and it's the most important subtlety of actor programming.

Consider an extension to the bank that converts currency at the current exchange rate, which must be fetched from a remote server:

```swift
extension Bank {
    // NOTE: buggy!
    func withdraw(_ amount: Int, in currency: String) async throws -> Bool {
        guard balance >= amount else {
            return false
        }
        let rate = try await exchangeRate(for: currency)  // suspension point!
        balance -= Int(Double(amount) * rate)
        return true
    }
}
```

The check `balance >= amount` happens before the `await`, and the update after it. While this method is suspended waiting for the exchange rate, other calls to `withdraw` can run on the actor and reduce the balance, so by the time this call resumes, the check it made is stale, and the balance can go negative. This is the same check-then-act race we saw with `withdraw` in Section 9.2, reappearing in a new form. The compiler doesn't catch it, because there's no data race: every access to `balance` is properly isolated. It's a *logical* race.

The fix is to treat each `await` in an actor method as a point where any of the actor's state may change, and to arrange the code so that every invariant holds at each suspension point. Here, we should do the asynchronous work first, and then check and update the state in one synchronous stretch:

```swift
extension Bank {
    func withdraw(_ amount: Int, in currency: String) async throws -> Bool {
        let rate = try await exchangeRate(for: currency)
        // No suspension points from here to the end: this part is atomic.
        let converted = Int(Double(amount) * rate)
        guard balance >= converted else {
            return false
        }
        balance -= converted
        return true
    }
}
```

Why are actors reentrant, if it causes such problems? Because the alternative is worse. A non-reentrant actor would be locked for the whole duration of each call, including all its awaits, so a slow network request would block every other use of the actor. And two non-reentrant actors that called each other would deadlock. Reentrancy keeps actors deadlock-free, at the price of requiring care at every `await`.

## 9.5. Lazy Initialization

It is good practice to defer an expensive initialization step until the moment it is needed. Initializing a variable up front increases the start-up latency of a program and is unnecessary if execution doesn't always reach the part of the program that uses that variable. Let's return to the `icons` variable we saw earlier in the chapter. Suppose loading the icons is slow, and we'd like to do it only on first use.

In Go, this requires care: a naïve `if icons == nil { loadIcons() }` is a data race, and the fix is `sync.Once`. In Swift, it's free. As we saw in Section 2.6, global variables and static properties are always initialized lazily, on first access, and the language guarantees that the initialization runs exactly once, even if several threads access the variable for the first time simultaneously:

```swift
let icons: [String: Image] = loadIcons()  // in a file other than main.swift

/// Concurrency-safe.
func icon(_ name: String) -> Image? {
    icons[name]
}
```

The first call to `icon`, from whatever task, triggers `loadIcons()`; any other task that calls `icon` at the same time waits for it to finish. Every later call just reads the dictionary. Static properties behave the same way, which makes `static let shared = ...` the standard idiom for a lazily created, thread-safe singleton.

Instance properties can be declared `lazy var`, which also defers initialization until first access. But a `lazy var` is *not* thread-safe: two tasks accessing it simultaneously could both run the initializer. Since it's a mutable property, the usual concurrency rules apply, and the compiler won't let you share an object with a `lazy var` between tasks without protection.

When lazy initialization must be asynchronous, for example when the value is fetched over the network, neither of these will do, since a global's initializer can't `await`. The solution is an actor that caches a `Task`, which we'll see in Section 9.7.

## 9.6. Strict Concurrency Checking and the Thread Sanitizer

Even with the greatest of care, it's all too easy to make concurrency mistakes. That's why Swift checks for data races at compile time. In the Swift 6 language mode, which is the default for new packages with `swift-tools-version: 6.0` or later, violations of data-race safety are errors. In the Swift 5 language mode, the same checks can be enabled as warnings, which is useful for migrating existing code a module at a time:

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

The compile-time checks have escape hatches, for situations where the programmer knows something the compiler can't verify: `@unchecked Sendable` for a class that protects its state with its own lock, `nonisolated(unsafe)` for a global variable, `@preconcurrency import` for a module that hasn't adopted `Sendable` annotations yet, and `MainActor.assumeIsolated` for code that's known to run on the main thread. Each of these is a promise that, if broken, can result in a data race.

To catch races in code that uses these escape hatches, or in C and C++ code called from Swift, use the *Thread Sanitizer* (TSan). It instruments every memory access in the program, records which thread accessed which memory and under what synchronization, and reports any access to the same location from two threads without an intervening synchronization event:

```
$ swift build --sanitize=thread
$ swift test --sanitize=thread
```

Here's a program that lies to the compiler, and what TSan says about it:

```swift
nonisolated(unsafe) var counter = 0

await withTaskGroup(of: Void.self) { group in
    for _ in 0..<2 {
        group.addTask { counter += 1 }
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
  Location is global 'counter' of size 8 at 0x0001049c8030
==================
```

The report includes the identities of the threads involved and their stacks, which is usually enough to pinpoint the problem. Like Go's race detector, TSan reports only races that actually occur during a run, so it's only as good as your tests. And it adds significant overhead, so it's not suitable for production builds. But in combination with Swift 6's compile-time checking, it covers nearly all the ways a data race can arise.

## 9.7. Example: Concurrent Non-Blocking Cache

In this section, we'll build a *concurrent non-blocking cache*, an abstraction that solves a problem that arises often in real-world concurrent programs but is not well addressed by existing libraries. This is the problem of *memoizing* a function, that is, caching the result of a function so that it need be computed only once. Our solution will be concurrency-safe and will avoid the contention associated with designs based on a single lock for the whole cache.

We'll use the `httpGetBody` function below as an example of the type of function we might want to memoize. It makes an HTTP GET request and returns the response body. Calls to this function are relatively expensive, so we'd like to avoid repeating them unnecessarily.

```swift
@Sendable func httpGetBody(_ url: String) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)
    return data
}
```

Here's the first draft of the cache:

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

A `Memo` instance holds the function `f` to memoize, of type `Function`, and the cache, which is a dictionary from keys to `Result`s (Section 5.10), so that errors are cached along with values.

Here's how to use `Memo`. For each element in a stream of incoming URLs, we call `get`, logging the latency of the call and the amount of data it returns:

```swift
let m = Memo(httpGetBody)
for url in incomingURLs() {
    let start = ContinuousClock.now
    do {
        let value = try await m.get(url)
        print("\(url), \(ContinuousClock.now - start), \(value.count) bytes")
    } catch {
        print(error)
    }
}
```

We can use a test harness to investigate the effect of memoization. From the output below, we see that the URL stream contains duplicates, and that although the first call to `m.get` for each URL takes hundreds of milliseconds, the second request returns the same amount of data in under a millisecond:

```
https://www.swift.org, 0.175026 seconds, 21342 bytes
https://forums.swift.org, 0.406393 seconds, 86104 bytes
https://swiftpackageindex.com, 0.633774 seconds, 148203 bytes
https://www.swift.org, 0.000002 seconds, 21342 bytes
https://forums.swift.org, 0.000001 seconds, 86104 bytes
https://swiftpackageindex.com, 0.000001 seconds, 148203 bytes
```

The test above executes all the calls to `get` sequentially. Since HTTP requests are a great opportunity for parallelism, let's change the test so that it makes all requests concurrently, each in its own child task:

```swift
await withTaskGroup(of: Void.self) { group in
    for url in incomingURLs() {
        group.addTask {
            let start = ContinuousClock.now
            let value = try? await m.get(url)  // error: capture of non-sendable type 'Memo<String, Data>'
            print("\(url), \(ContinuousClock.now - start), \(value?.count ?? 0) bytes")
        }
    }
}
```

In Go, the equivalent program compiles and runs, and usually seems to work, but occasionally crashes or corrupts the cache, because `get` makes unsynchronized accesses to the map. Go's race detector would find the bug if a test happened to exercise it. In Swift 6, the program doesn't compile. `Memo` is a class with mutable state, so it isn't `Sendable`, and it can't be captured by a closure that runs in a child task. We have to make it concurrency-safe.

The simplest way to make the cache concurrency-safe is to use monitor-based synchronization, with a `Mutex` around the dictionary:

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

Now the compiler is satisfied, and the concurrent requests make the overall test run much faster. Notice that we take the lock twice: once to look up the key, and again to store the result. We *couldn't* hold the lock during the call to `f`, because `withLock`'s closure is synchronous and can't `await`, and even if we could, it would serialize all the calls to `f`, defeating the purpose.

Unfortunately, this means that when two tasks call `get` for the same URL at about the same time, both see that the cache has no entry, both call the slow function `f`, and both store the result, the second overwriting the first. The program is correct, but it does redundant work. Ideally, we'd like to avoid this *duplicate suppression* problem.

The solution is to cache not the *result* of the function, but the *task* that computes it. A `Task` represents work that is in progress or finished; any number of other tasks can await its `value`, and all of them receive the same result. The first task to request a key creates the task and stores it in the cache; later requesters find the task and wait for it. And to make the check and the store happen atomically, we'll use an actor:

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

Let's check this design against the reentrancy rule from Section 9.4. There are two suspension points in `get`: the two `await task.value` expressions. The check `cache[key]`, the creation of the task, and the store `cache[key] = task` all happen with no `await` in between, so they're atomic with respect to other calls on the actor. A second call to `get` for the same key, even one that arrives while the first is suspended awaiting the result, will find the task in the cache, and await the same task. Because the actor is suspended, not blocked, while waiting, calls for *other* keys proceed concurrently; the slow function `f` runs in its own task, outside the actor, so many calls to `f` can be in progress at once.

The capture list `[f]` captures the function value itself, rather than `self`, so that the new task needn't hop onto the actor to read the property.

Compare this with the Go version of the same cache, which needs a mutex, a map of entries, each with a `ready` channel that is closed when the result is available, and a careful dance of unlocking and locking to avoid holding the mutex during the call to `f`. In Swift, `Task` already provides what the entry type and the `ready` channel do, and the actor provides what the mutex does. The design is the same; the language just has the pieces built in.

```
https://www.swift.org, 0.176201 seconds, 21342 bytes
https://www.swift.org, 0.176310 seconds, 21342 bytes
https://forums.swift.org, 0.414012 seconds, 86104 bytes
https://forums.swift.org, 0.414104 seconds, 86104 bytes
https://swiftpackageindex.com, 0.640027 seconds, 148203 bytes
https://swiftpackageindex.com, 0.640190 seconds, 148203 bytes
```

Now the duplicate requests take about the same time as the original ones, because they waited for them, but only one HTTP request was made for each URL.

**Exercise 9.1:** The standard library's `Result(catching:)` initializer takes a synchronous closure. Write an `async` overload of it, and use it to shorten `memo1` and `memo2`.

**Exercise 9.2:** Extend the `Memo` actor so that the caller may provide an optional cancellation for a request: if the task that requested a value is cancelled, its wait should end. Should the shared computation be cancelled too? What if other callers are still waiting for it?

**Exercise 9.3:** Add a maximum size to the cache, evicting the least recently used entry when it is full. Take care not to evict entries whose tasks are still running.

**Exercise 9.4:** Write a version of the cache in which failed results are not cached, so that a later request retries the operation.

## 9.8. Tasks and Threads

In the previous chapter, we said that the difference between tasks and operating system (OS) threads could be ignored until later. Although the differences between them are essentially quantitative, a big enough quantitative difference becomes a qualitative one, and so it is with tasks and threads. The time has now come to distinguish them.

### 9.8.1. Stacks and Frames

Each OS thread has a fixed-size block of memory (often as large as 8 MB) for its *stack*, the work area where it saves the local variables of function calls that are in progress or temporarily suspended while another function is called. This fixed-size stack is simultaneously too much and too little. An 8 MB stack is a big waste of memory for a little task that just waits for a network response. It's not uncommon for a server to have tens of thousands of connections in progress at once, which would be impossible with a thread for each.

A Swift task doesn't have a stack of its own. While an `async` function is *running*, it uses the stack of whatever thread it's running on, like an ordinary function. When it *suspends*, the state that must survive the suspension (the local variables that are still needed, and the point at which to resume) is saved in an *async frame*, a heap-allocated block of exactly the right size. Tasks that are suspended thus cost only as much memory as their saved state, often just a few hundred bytes, so a program can easily have a hundred thousand tasks.

### 9.8.2. Scheduling

OS threads are scheduled by the OS kernel. Every few milliseconds, a hardware timer interrupts the processor, which causes the kernel to suspend the currently executing thread and save its registers in memory, look over the list of threads and decide which one should run next, restore that thread's registers from memory, then resume the execution of that thread. Because OS threads are scheduled by the kernel, passing control from one thread to another requires a full *context switch*, which is slow.

Swift's runtime contains its own scheduler. Tasks run on *executors*. Most tasks run on the *global concurrent executor*, which runs tasks on the cooperative thread pool, with one thread per CPU core. Each actor has a *serial executor*, which runs its code one task at a time (usually borrowing threads from the same pool). The main actor's executor is the main thread. When a task suspends, it simply returns to the executor, which picks the next runnable task, without any kernel involvement. Switching between tasks is about as cheap as a function call.

The fixed size of the cooperative pool has an important consequence that we noted in Section 8.5: a task that *blocks* (by waiting on a semaphore, a condition variable, a blocking read, or a long computation with no suspension points) takes a whole thread out of the pool. If all the threads block, every task in the program stalls. Go's runtime avoids this by creating more threads when goroutines block in system calls; Swift's does not. So in Swift, it's important that tasks always make *forward progress*: don't block a task waiting for another task to do something; `await` it instead. Long CPU-bound loops should occasionally call `await Task.yield()` so that other tasks get a turn.

### 9.8.3. Isolation and Where Code Runs

Whether a piece of code runs on the main thread, on an actor, or on the global executor is determined by its *isolation*, which is part of its declaration: a method of an actor runs on that actor, a `@MainActor` function on the main actor, and a `nonisolated` async function on the global executor. A closure passed to `Task { }` inherits the isolation of the code that creates it, which is why the spinner in Section 8.1 needed `Task.detached` to get off the main actor.

Swift 6.2 adjusted these defaults to make concurrency easier to adopt. With the "approachable concurrency" settings that are enabled by default in new app projects, a `nonisolated` async function runs on the *caller's* actor rather than switching to the global executor, and a function must be marked `@concurrent` to run on the global executor explicitly. Modules can also choose to make `@MainActor` the default isolation for all their code, which suits app targets that do most of their work on the main thread. Packages written for the Swift 6.0 defaults continue to compile unchanged; the new behavior is opt-in per module.

### 9.8.4. Tasks Have No Identity

In most operating systems and programming languages that support multithreading, the current thread has a distinct identity that can be easily obtained as an ordinary value, typically an integer or pointer. This makes it easy to build an abstraction called *thread-local storage*, which is essentially a global map keyed by thread identity, so that each thread can store and retrieve values independent of other threads.

Swift tasks have no identity accessible to the programmer, and a task may run on different threads at different times, so thread-local storage is meaningless for them. Instead, Swift provides *task-local values*, which are declared with the `@TaskLocal` property wrapper, bound for the duration of a scope, and inherited by child tasks:

```swift
enum Request {
    @TaskLocal static var id: String = "none"
}

await Request.$id.withValue("req-42") {
    await handle()  // handle and any child tasks it creates see Request.id == "req-42"
}
```

Task-locals are useful for cross-cutting values like request IDs for logging and tracing, which should flow through a call tree without being passed explicitly as parameters to every function. Like Go, though, Swift encourages a simpler style of programming in which parameters that affect the behavior of a function are explicit, so task-locals should be used sparingly.

We've now learned all the language features we need for writing concurrent programs in Swift. In the next two chapters, we'll step back and look at some of the techniques and tools that support programming in the large in Swift.
