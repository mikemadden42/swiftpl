# 8. Tasks and Asynchronous Sequences

Most interesting programs spend much of their time waiting: for a disk, for a network peer, for a user, for a timer. A program that does one thing at a time wastes that waiting. A server that handled one client at a time would make every other client queue behind the slowest. A command-line tool that fetched fifty URLs one after another would take the sum of fifty round trips instead of the longest one. And the processor itself now has many cores, which a strictly sequential program leaves idle. *Concurrency* is the discipline of structuring a program as several activities that can make progress independently, so that waiting in one place doesn't stop work everywhere else.

Swift treats concurrency as a language feature rather than a library. Three ideas fit together:

- **`async` and `await`.** A function marked `async` may *suspend* partway through, giving up its thread while it waits, and every point where that can happen is marked `await` in the source.
- **Structured concurrency.** Concurrent work is organized into *tasks*, and tasks form a tree: a child task is started inside a scope and must finish before that scope ends. Lifetimes are visible in the code, just as they are for local variables.
- **Data-race safety.** The compiler checks that whatever two tasks share is safe to share.

This chapter covers the first two ideas and adds a third tool, *asynchronous sequences*, which let one task hand a stream of values to another. Readers who know Go will recognize them as Swift's counterpart to channels. Chapter 9 takes up data-race safety, along with the classic shared-memory tools that Swift also provides: locks and actors.

Concurrent code is harder to reason about than sequential code, mostly because events in different tasks have no fixed order. The examples in this chapter were chosen to make that ordering question concrete. Run them, change them, and watch what happens.

## 8.1. Tasks and `async`/`await`

A *task* is the unit of concurrent work in Swift. Every piece of asynchronous code runs inside some task, and a program may have many tasks running, or waiting to run, at once.

It's reasonable to picture a task as a lightweight thread, and that picture will get you a long way. The difference is in the cost. Swift keeps a small pool of operating-system threads, normally one per CPU core, and runs every task on those threads. When a task reaches an `await` that can't complete yet, the task is set aside and its thread picks up some other task that is ready. When the awaited thing finishes, the suspended task is queued to resume, perhaps on a different thread than before. A suspended task occupies only the memory needed to remember its local state, so a program can comfortably keep tens of thousands of them alive.

The code at the top of `main.swift` (or the `main` method of an `@main` type) runs in the program's *main task*. The simplest way to start another task is the `Task` initializer, which takes a closure and runs it concurrently with the code that created it:

```swift
work()  // runs work; the next line waits until it returns
Task { await work() }  // starts work in a new task; the next line runs immediately
```

A task created this way is *unstructured*: it isn't tied to the scope that created it, and it keeps running until it finishes or the program exits. A variant, `Task.detached`, also refuses to inherit the *isolation* of its creator, a notion we'll define in Chapter 9. For now, the practical effect is that a detached task always runs on the shared thread pool and never on the main thread.

Here's a program that needs two things to happen at once. It counts the primes below five million by trial division, which takes long enough that a silent terminal would look like a hang, so a second task reports the elapsed time while the first one works:

```swift
// swiftpl/ch8/primes
import Foundation

let progress = Task.detached {
    await showElapsed()
}
let limit = 5_000_000
let count = countPrimes(below: limit)  // slow
progress.cancel()
print("\rThere are \(count) primes below \(limit).")

/// Rewrites a "working..." line with the elapsed time until cancelled.
func showElapsed() async {
    let start = ContinuousClock.now
    while !Task.isCancelled {
        let seconds = (ContinuousClock.now - start).components.seconds
        FileHandle.standardOutput.write(Data("\rworking... \(seconds)s".utf8))
        try? await Task.sleep(for: .milliseconds(250))
    }
}

func countPrimes(below n: Int) -> Int {
    var count = 0
    for candidate in 2..<n where isPrime(candidate) {
        count += 1
    }
    return count
}

func isPrime(_ n: Int) -> Bool {
    var d = 2
    while d * d <= n {
        if n % d == 0 {
            return false
        }
        d += 1
    }
    return n >= 2
}
```

While the count runs, the status line ticks upward; then it's overwritten by the answer:

```
There are 348513 primes below 5000000.
```

The work is split along the natural seam. The main task does the computation, which is ordinary synchronous code. The progress task does the reporting, which is asynchronous because it sleeps between updates, and `Task.sleep` suspends rather than blocking a thread. The two run side by side, and neither knows anything about the other's job.

When the count is done, the main task calls `progress.cancel()`. That doesn't stop the progress task by force. It sets a flag that the task can observe, here with `Task.isCancelled`, and it makes a pending `Task.sleep` wake up early. A task in Swift stops when *it* decides to; Section 8.9 explores why.

Two smaller details: the status line is written with `FileHandle` because `print` holds output in a buffer until it sees a newline; and the progress task was created with `Task.detached` so that it runs on the thread pool, since the main task's thread is busy computing.

When the main code returns, the process exits, taking any unfinished tasks with it. There's no way for one task to terminate another; cancellation is a request, and the request has to be honored by the code that receives it.

### 8.1.1. What `await` Means

Marking a function `async` gives it a new ability: it can suspend. In exchange, every call to it must be written with `await`, and can appear only in a context that is itself asynchronous, such as another `async` function, a task's closure, or top-level code that already uses `await`.

```swift
func loadProfile(id: Int) async throws -> Profile { ... }

let profile = try await loadProfile(id: 7)  // this line may suspend
```

Some languages, Go among them, let any function block and hide the cost from the type system. Swift makes the distinction part of each function's signature. That has a price: a synchronous function can't simply call an asynchronous one; it has to start a task to do it. But it also has a payoff. Between two `await`s, code runs without interruption from anything else sharing its actor, and when you read a function, the `await` keywords show exactly where the world might change underneath you. That property becomes important in Chapter 9.

An asynchronous call is cheap when it doesn't actually need to wait: it costs about the same as an ordinary call. Only when a function really suspends does Swift move its live local variables into a small heap-allocated frame and release the thread. There is no per-task stack to reserve or grow.

### 8.1.2. Children, Not Orphans

`Task { }` starts work that floats free of the code around it. Most of the time, though, concurrent work belongs to a particular operation, and should begin and end with it. *Structured concurrency* expresses that directly. We met one structured form, the task group, in Section 1.6. The other is `async let`, which starts a child task to compute a value and lets you collect the value later:

```swift
async let home = loadProfile(id: 7)  // child task 1 starts
async let work = loadProfile(id: 8)  // child task 2 starts
let both = try await [home, work]  // wait for both results
```

The two loads overlap. If the enclosing scope is left before the values are awaited, perhaps because an error was thrown, Swift cancels the children and waits for them to finish before leaving. A child task can't outlive its parent's scope, so it can't be forgotten.

Because every child has a parent, the tasks in a program form a tree, and several things flow down that tree automatically: cancellation (cancel a parent and all its descendants are cancelled), priority, and task-local values (Section 9.8). Reading structured code is also easier, for the same reason that blocks are easier to read than jumps: the region of source text where concurrent work exists is the region between the braces.

A good default is to reach for `async let` and task groups, and to use unstructured `Task { }` only when work truly has to outlive the code that starts it, as when a synchronous event handler kicks off a background job.

## 8.2. Example: Concurrent Clock Server

Network servers are the textbook case for concurrency. Each connected client is an independent conversation, and a server that talks to one client at a time is a server that most clients find unresponsive.

For network programming beyond HTTP, Swift's foundation is SwiftNIO, the event-driven networking library on which Hummingbird (Chapter 1) and most other Swift servers are built. NIO's `NIOAsyncChannel` adapts a network connection to Swift concurrency: incoming data arrives as an asynchronous sequence, and outgoing data is sent through an asynchronous writer.

Our first server implements a variation on the old Internet *daytime* service: a client connects and the server sends it the current time, once a second, until the client goes away.

```swift
// swiftpl/ch8/clock1
// Clock1 is a TCP server that periodically writes the time.
import Foundation
import NIOCore
import NIOPosix

let server = try await ServerBootstrap(group: .singletonMultiThreadedEventLoopGroup)
    .serverChannelOption(.socketOption(.so_reuseaddr), value: 1)
    .bind(host: "localhost", port: 8000) { channel in
        channel.eventLoop.makeCompletedFuture {
            try NIOAsyncChannel<ByteBuffer, ByteBuffer>(wrappingChannelSynchronously: channel)
        }
    }

try await server.executeThenClose { connections in
    for try await connection in connections {
        await handle(connection)  // handle one connection at a time
    }
}

func handle(_ connection: NIOAsyncChannel<ByteBuffer, ByteBuffer>) async {
    do {
        try await connection.executeThenClose { _, outbound in
            while true {
                try await outbound.write(ByteBuffer(string: timestamp() + "\n"))
                try await Task.sleep(for: .seconds(1))
            }
        }
    } catch {
        return  // e.g., client disconnected
    }
}

func timestamp() -> String {
    let formatter = DateFormatter()
    formatter.dateFormat = "HH:mm:ss"
    return formatter.string(from: .now)
}
```

The package depends on `https://github.com/apple/swift-nio` and its `NIOCore` and `NIOPosix` products.

Reading from the top: `ServerBootstrap` sets up a listening socket on port 8000. The closure given to `bind` runs once for every accepted connection and decides how that connection will be presented to us; here, as an `NIOAsyncChannel` of raw `ByteBuffer`s in both directions. What `bind` returns is itself an asynchronous channel, one whose incoming "messages" are connections. Calling `executeThenClose` on it hands us that stream of connections and guarantees the listener is shut down when the closure exits.

The `for try await` loop is the asynchronous form of `for`-`in`. Each trip around the loop may have to wait (hence `await`) for the next client, and accepting can fail (hence `try`).

`handle` serves a single client. It writes a line, sleeps for a second, and repeats. The `write` call is itself asynchronous and doesn't finish until NIO has accepted the bytes, so a client that reads slowly automatically slows the server down instead of making it buffer without limit. When the client disconnects, the next `write` throws, the loop ends, and `handle` returns.

Any TCP client will do for testing. The `nc` (netcat) utility, which comes with most Unix systems, connects standard input and output to a socket:

```
$ swift build
$ .build/debug/clock1 &
$ nc localhost 8000
09:41:07
09:41:08
09:41:09
^C
```

Now try connecting a second `nc` from another terminal while the first is still running. The second client sees nothing at all. Only when you stop the first client does the second start receiving times. The accept loop calls `handle` and waits for it to return, and for a well-behaved client that never happens, so this server is strictly one-at-a-time.

The fix is to stop waiting. Instead of calling `handle` directly, the accept loop hands each connection to a new child task. Since the children don't produce values anyone needs, we use a *discarding* task group, which forgets each child as soon as it completes rather than holding its result:

```swift
// swiftpl/ch8/clock2 (builds on clock1)
try await withThrowingDiscardingTaskGroup { group in
    try await server.executeThenClose { connections in
        for try await connection in connections {
            group.addTask { await handle(connection) }  // one child task per client
        }
    }
}
```

With that change, every client gets its own task, and any number of terminals can watch the time at once.

**Exercise 8.1:** Extend `clock2` so that a client may send the name of a time zone, such as `Asia/Tokyo`, as its first line, after which the server reports times in that zone. Use `TimeZone(identifier:)` and reject unknown names politely.

**Exercise 8.2:** Write a minimal TCP client in Swift, using NIO's `ClientBootstrap`, that connects to a host and port and copies everything it receives to the standard output. Use it in place of `nc`.

## 8.3. Example: Concurrent Reminder Server

In the clock server, concurrency was *between* clients: one task per connection. Sometimes a single connection needs several tasks of its own. To see why, we'll build a small reminder service. A client sends lines of the form `SECONDS MESSAGE`; the server acknowledges each one immediately and, after the given number of seconds, sends the message back.

The protocol is line-oriented, so we ask NIO to split the incoming bytes into lines for us. The `swift-nio-extras` package provides a *frame decoder* for that, which we add to the connection's processing *pipeline* when it's set up:

```swift
// swiftpl/ch8/remind1
import NIOCore
import NIOExtras
import NIOPosix

let server = try await ServerBootstrap(group: .singletonMultiThreadedEventLoopGroup)
    .serverChannelOption(.socketOption(.so_reuseaddr), value: 1)
    .bind(host: "localhost", port: 8000) { channel in
        channel.eventLoop.makeCompletedFuture {
            try channel.pipeline.syncOperations.addHandler(
                ByteToMessageHandler(LineBasedFrameDecoder()))
            return try NIOAsyncChannel<ByteBuffer, ByteBuffer>(
                wrappingChannelSynchronously: channel)
        }
    }

try await withThrowingDiscardingTaskGroup { group in
    try await server.executeThenClose { connections in
        for try await connection in connections {
            group.addTask { try? await handle(connection) }
        }
    }
}

func handle(_ connection: NIOAsyncChannel<ByteBuffer, ByteBuffer>) async throws {
    try await connection.executeThenClose { inbound, outbound in
        for try await line in inbound {
            try await remind(String(buffer: line), to: outbound)
        }
    }
}

/// Handles a request of the form "SECONDS MESSAGE": acknowledges it,
/// waits that many seconds, then sends MESSAGE back to the client.
func remind(_ request: String, to out: NIOAsyncChannelOutboundWriter<ByteBuffer>) async throws {
    let parts = request.split(separator: " ", maxSplits: 1)
    guard parts.count == 2, let seconds = Int(parts[0]), seconds >= 0 else {
        try await out.write(ByteBuffer(string: "usage: SECONDS MESSAGE\n"))
        return
    }
    try await out.write(ByteBuffer(string: "ok, in \(seconds)s\n"))
    try await Task.sleep(for: .seconds(seconds))
    try await out.write(ByteBuffer(string: "reminder: \(parts[1])\n"))
}
```

With the line decoder installed, each element of `inbound` is one line, minus its terminator, and `String(buffer:)` turns it into a string. Here's a session. The lines without a prefix are what we typed:

```
$ swift build
$ .build/debug/remind1 &
$ nc localhost 8000
5 tea is ready
ok, in 5s
2 stand up and stretch
reminder: tea is ready
ok, in 2s
reminder: stand up and stretch
```

The second request was typed right after the first, yet the server didn't even acknowledge it until the tea reminder had gone off five seconds later. Reminders within one connection are served strictly in order, each one blocking the next, because the loop in `handle` waits for `remind` to finish before reading another line. That's correct for a clock, but wrong for a reminder service, where requests are independent.

The cure is the same as before, one level down: give each request its own child task.

```swift
// swiftpl/ch8/remind2 (builds on remind1)
func handle(_ connection: NIOAsyncChannel<ByteBuffer, ByteBuffer>) async throws {
    try await connection.executeThenClose { inbound, outbound in
        try await withThrowingDiscardingTaskGroup { group in
            for try await line in inbound {
                let request = String(buffer: line)
                group.addTask { try await remind(request, to: outbound) }
            }
        }
    }
}
```

Now each request is acknowledged as soon as it arrives, and reminders come back in the order their timers expire:

```
$ nc localhost 8000
5 tea is ready
ok, in 5s
2 stand up and stretch
ok, in 2s
reminder: stand up and stretch
reminder: tea is ready
```

The change was small, but it raises a question worth pausing on. Several child tasks now write to the same `outbound` writer, possibly at the same instant. Is that safe? For most objects it wouldn't be. Here it is, because NIO's outbound writer is designed for concurrent use and declares itself `Sendable`. More importantly, we didn't have to take that on faith. The Swift 6 compiler checks everything a child task's closure captures (in this case `request`, `outbound`, and the function `remind`) and refuses to compile the program if any of it isn't safe to share. Chapter 9 explains how that check works.

**Exercise 8.3:** In `remind2`, the task group keeps the connection open until every pending reminder has fired, even after the client has sent its last line. Is that what users would want? What happens to pending reminders if the client disconnects?

**Exercise 8.4:** Add a `cancel` command that cancels all of a client's pending reminders. (Hint: one way is to keep the reminders in a nested task group that you can cancel with `cancelAll()`.)

## 8.4. Async Streams

Tasks are the *workers* of a concurrent program; something has to carry values between them. In Swift, that's usually an *asynchronous sequence*: a sequence whose elements may not exist yet, so getting the next one can involve waiting. Any type conforming to `AsyncSequence` can be consumed with `for await`. We've already used several without dwelling on them: a server's stream of connections, a connection's stream of lines, and a task group's stream of results.

When you need a channel of your own between two tasks, the standard library's `AsyncStream` is the general-purpose tool. Making one gives you two separate values: the stream, which a consumer reads, and a *continuation*, which a producer uses to feed it:

```swift
let (stream, continuation) = AsyncStream.makeStream(of: String.self)
```

The producer calls `yield(_:)` once per value and `finish()` when there will be no more. The consumer's `for await` loop delivers the values in order and exits once the stream has been finished and everything already yielded has been read:

```swift
Task {
    for word in ["alpha", "beta", "gamma"] {
        continuation.yield(word)
    }
    continuation.finish()
}
for await word in stream {
    print(word)
}
print("stream finished")
```

The split into two values matters for design. A function that should only produce values can be given only the continuation; one that should only consume can be given only the stream. Neither can misuse the other end, because it doesn't have it. (Go achieves the same discipline with send-only and receive-only channel types.)

### 8.4.1. Buffering and Back-Pressure

`yield` doesn't wait for a consumer. If nobody is currently waiting on the stream, the value goes into a buffer. By default that buffer has no limit, which is convenient but has an obvious failure mode: a producer that outpaces its consumer makes the buffer, and the process, grow without bound. A stream can be created with a policy that caps the buffer, keeping only the oldest or only the newest values:

```swift
let (stream, continuation) = AsyncStream.makeStream(
    of: Double.self, bufferingPolicy: .bufferingNewest(1))
```

A buffer of one newest value suits progress indicators and sensor readings, where a stale value is worthless once a newer one exists. The result returned by `yield` tells the producer whether its value was buffered, dropped, or rejected because the stream had already ended.

What `AsyncStream` can't do is make the producer *wait* until the consumer has caught up. That property, called *back-pressure*, is sometimes essential, for instance when each value represents real work that shouldn't pile up. The `swift-async-algorithms` package provides `AsyncChannel` for that case. Its `send` method is asynchronous and suspends until a consumer has taken the value:

```swift
import AsyncAlgorithms

let jobs = AsyncChannel<Int>()
Task {
    for id in 1...3 {
        await jobs.send(id)  // suspends until a consumer receives it
    }
    jobs.finish()
}
for await id in jobs {
    print("processing job \(id)")
}
```

With `AsyncChannel`, producer and consumer move in lockstep, which is exactly the rendezvous behavior of an unbuffered channel in Go. The same package contains a toolkit of operators for asynchronous sequences: `merge` and `zip` for combining them, `debounce` and `throttle` for rate control, `chunked` for batching, `timer` for periodic events, and more.

One rule to keep in mind: an `AsyncStream` expects a *single* consumer. Two tasks iterating the same stream at once is a bug. If you want to spread work across several consumers, use a task group; if you want every consumer to see every value, give each its own stream. Section 8.10 does the latter.

### 8.4.2. Pipelines

Chaining tasks with streams, so that each stage consumes the previous stage's output and produces input for the next, gives a *pipeline*. Here's a three-stage pipeline that finds primes, reusing `isPrime` from Section 8.1:

```
generate --numbers--> filter --primes--> report
```

The first stage emits the integers from 2 to 50, the second passes along only the primes, and the third prints what reaches it:

```swift
// swiftpl/ch8/pipeline1 (builds on primes, for isPrime)
let (numbers, numbersIn) = AsyncStream.makeStream(of: Int.self)
let (primes, primesIn) = AsyncStream.makeStream(of: Int.self)

await withDiscardingTaskGroup { group in
    // Stage 1: generate candidates.
    group.addTask {
        for n in 2...50 {
            numbersIn.yield(n)
        }
        numbersIn.finish()
    }

    // Stage 2: keep only the primes.
    group.addTask {
        for await n in numbers where isPrime(n) {
            primesIn.yield(n)
        }
        primesIn.finish()
    }

    // Stage 3: report (in the parent task).
    for await p in primes {
        print(p, terminator: " ")
    }
    print()
}
```

Shutdown flows through the pipeline from front to back. The generator finishes its stream when it runs out of numbers; that ends the filter's loop, and the filter finishes *its* stream in turn, which ends the reporting loop. A stage that forgets to call `finish` leaves everything downstream waiting forever. (A continuation that is destroyed without being finished does finish its stream, which is a useful safety net but not something to depend on.)

As pipelines grow, it's clearer to make each stage a function and pass it only the ends it needs:

```swift
// swiftpl/ch8/pipeline2 (builds on primes, for isPrime)
func generate(_ range: ClosedRange<Int>, into out: AsyncStream<Int>.Continuation) {
    for n in range {
        out.yield(n)
    }
    out.finish()
}

func keepPrimes(from input: AsyncStream<Int>, into out: AsyncStream<Int>.Continuation) async {
    for await n in input where isPrime(n) {
        out.yield(n)
    }
    out.finish()
}

func report(_ input: AsyncStream<Int>) async {
    for await p in input {
        print(p, terminator: " ")
    }
    print()
}

let (numbers, numbersIn) = AsyncStream.makeStream(of: Int.self)
let (primes, primesIn) = AsyncStream.makeStream(of: Int.self)
async let g: Void = generate(2...50, into: numbersIn)
async let k: Void = keepPrimes(from: numbers, into: primesIn)
await report(primes)
_ = await (g, k)
```

The signatures now document the data flow: `keepPrimes` reads one stream and writes another, and couldn't do anything else.

It's worth knowing that a lot of pipelines need no extra tasks at all. Asynchronous sequences have the familiar sequence operations (`map`, `filter`, `compactMap`, `prefix`, `reduce`, `contains`, and the rest), each of which wraps the sequence lazily. The filter stage could be just:

```swift
for await p in numbers.filter(isPrime) {
    print(p, terminator: " ")
}
```

In this form, the filtering happens in the consumer's task, one element at a time, so nothing runs in parallel. That's often exactly right. Introduce separate stages only when they gain something from running simultaneously, for example when one stage waits on I/O while another computes.

**Exercise 8.5:** Rewrite the generator stage as a function `numbers(_ range: ClosedRange<Int>) -> AsyncStream<Int>` that creates the stream and a task to fill it, and returns the stream. If the consumer stops iterating early, the generator keeps running. Use `AsyncStream.Continuation.onTermination` to stop it.

## 8.5. Looping in Parallel

One of the most common uses of concurrency is also one of the simplest: running the iterations of a loop at the same time. As an example, we'll compute a SHA-256 digest for each of a list of files, the job done by the `sha256sum` utility. The per-file work is a single function:

```swift
// swiftpl/ch8/digest
import Crypto
import Foundation

/// Returns the SHA256 digest of the file's contents as a hex string.
func digest(of path: String) throws -> String {
    let data = try Data(contentsOf: URL(filePath: path))
    return SHA256.hash(data: data).map { String(format: "%02x", $0) }.joined()
}
```

and the obvious loop calls it once per file:

```swift
// digestFiles computes the digests of the specified files.
func digestFiles(_ paths: [String]) throws -> [String: String] {
    var result: [String: String] = [:]
    for path in paths {
        result[path] = try digest(of: path)
    }
    return result
}
```

Nothing about one file's digest depends on any other's. When a problem divides into pieces that are entirely independent like this, it's called *embarrassingly parallel*, and it's the easiest kind of problem to speed up: with *n* cores and enough files, we can hope for something close to an *n*-fold speedup, while overlapping disk reads with hashing besides.

A task group is the right shape for it. One child per file, and the parent collects the answers:

```swift
func digestFiles(_ paths: [String]) async throws -> [String: String] {
    try await withThrowingTaskGroup(of: (String, String).self) { group in
        for path in paths {
            group.addTask {
                (path, try digest(of: path))
            }
        }
        var result: [String: String] = [:]
        for try await (path, hash) in group {
            result[path] = hash
        }
        return result
    }
}
```

Each child returns its file's name along with its digest, so that the parent can tell the results apart; they arrive in completion order, not argument order. All writes to `result` happen in the parent, so there's nothing to synchronize.

It's instructive to list what this code does *not* have to do. It doesn't count outstanding workers to know when it's finished, because `withThrowingTaskGroup` can't return while a child is still running. It doesn't need special care to avoid stranding workers when one of them fails: an error from any child surfaces in the parent's `for try await`, and as it propagates out of the group, the group cancels the remaining children and waits for them before passing the error on. And there's no question of every closure accidentally sharing one loop variable, since `path` is a fresh constant on each iteration. In languages without structured concurrency, each of those is a classic source of bugs.

A task group is itself an asynchronous sequence of child results, so aggregate questions can use sequence operations directly. Here's the total size of the files:

```swift
func totalSize(_ paths: [String]) async throws -> Int {
    try await withThrowingTaskGroup(of: Int.self) { group in
        for path in paths {
            group.addTask {
                let attrs = try FileManager.default.attributesOfItem(atPath: path)
                return (attrs[.size] as? NSNumber)?.intValue ?? 0
            }
        }
        return try await group.reduce(0, +)
    }
}
```

### 8.5.1. Limiting Parallelism

Our parallel `digestFiles` starts one child per file, all at once. For ten files that's fine. For ten thousand, it creates ten thousand tasks up front. The thread pool will only *run* a handful at a time, but each started task may already be holding a file's contents in memory. The usual remedy is to keep a fixed number of children in flight, starting a new one only when an old one finishes, like a sliding window over the work:

```swift
func digestFiles(_ paths: [String], maxConcurrent: Int = 8) async throws -> [String: String] {
    try await withThrowingTaskGroup(of: (String, String).self) { group in
        var pending = paths.makeIterator()
        func startNext() {
            if let path = pending.next() {
                group.addTask { (path, try digest(of: path)) }
            }
        }
        for _ in 0..<maxConcurrent {
            startNext()
        }
        var result: [String: String] = [:]
        for try await (path, hash) in group {
            result[path] = hash
            startNext()  // replace the finished task with a new one
        }
        return result
    }
}
```

The nested function `startNext` captures the iterator and the group and starts at most one child per call. The loop primes the window with `maxConcurrent` children and then starts one more each time a result comes back, so the group never holds more than `maxConcurrent` children.

There's a caveat hiding in `digest(of:)`. `Data(contentsOf:)` reads the file *synchronously*: the thread executing it can do nothing else until the read completes. Reading local files is quick enough that this rarely matters, but if the files lived on a slow network share, a burst of blocking reads could occupy every thread in Swift's small pool, and every other task in the program, including ones with nothing to do with files, would stall until a read finished. Swift's scheduler is built on the assumption that tasks don't block. The sliding window helps by bounding how many reads are in flight; for truly slow I/O, use asynchronous APIs or move the blocking work to a thread of its own.

**Exercise 8.6:** Write `sha256sum`, which prints the digest of each file named on the command line, in argument order, while computing the digests in parallel.

**Exercise 8.7:** Time `digestFiles` on a directory of large files with `maxConcurrent` set to 1, 2, 4, and so on up to 64. Where does the speedup stop growing, and why there?

## 8.6. Example: Concurrent Web Crawler

The link checker of Section 5.6 visited one page at a time, and so spent nearly all its time waiting on the network. Page fetches are independent of each other, which makes crawling a natural candidate for parallel work. In this section we'll build a concurrent crawler on the same `extract` function from the `Links` module. Each crawl prints a page's URL and returns its links:

```swift
// swiftpl/ch8/crawl1
import Foundation
import Links

func crawl(_ url: String) async -> [String] {
    print(url)
    do {
        return try await extract(url)
    } catch {
        printError("\(error)")
        return []
    }
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

The driver keeps a worklist of URLs to visit and a set of URLs already seen, as the link checker did, but it runs up to 20 crawls at a time as children of a task group:

```swift
// swiftpl/ch8/crawl1 (continued)
let maxConcurrent = 20

await withTaskGroup(of: [String].self) { group in
    var worklist = Array(CommandLine.arguments.dropFirst())
    var seen = Set<String>()
    var running = 0
    while true {
        // Start tasks for unseen links, up to the limit.
        while running < maxConcurrent, let url = worklist.popLast() {
            guard seen.insert(url).inserted else { continue }
            group.addTask { await crawl(url) }
            running += 1
        }
        // Wait for one task to finish, and add its links to the worklist.
        guard let links = await group.next() else {
            break  // no tasks running, and nothing left to start
        }
        running -= 1
        worklist += links
    }
}
```

The loop alternates between two steps. First it fills any free slots with crawls of URLs it hasn't seen. Then it waits for whichever crawl finishes next, using `group.next()`, and adds the links that crawl found to the worklist. When there's nothing running and nothing to start, `next()` returns `nil` and the crawl is over.

Notice where the state lives. The worklist, the seen set, and the running count are all local variables of the parent, and only the parent reads or writes them. The children share nothing; they receive a URL and hand back a list. Because of that division of labor, the program has no locks and no possibility of a data race, and the compiler can confirm it.

The task group does three jobs here. It runs the workers, it limits how many run at once (with a little help from `running`), and it tells us when everything is done. In other concurrency models, each of those might need a separate mechanism.

One behavioral detail: popping from the *end* of the array makes the exploration order closer to depth-first than breadth-first. If the order matters, a double-ended queue such as `Deque` from the `swift-collections` package, used with `popFirst()`, restores breadth-first order.

With twenty fetches in flight, the crawler is dramatically faster than its sequential ancestor. It's also unbounded: pointed at a well-linked site, it will keep going until you stop it.

**Exercise 8.8:** Add a `--depth` option, so that only pages within that many links of the starting URLs are fetched. (Hint: put `(url, depth)` pairs in the worklist.)

**Exercise 8.9:** Turn the crawler into a site mirroring tool: save each fetched page under a local directory, stay within the starting site's host, and rewrite links in saved pages so that they point to the local copies.

## 8.7. Racing Tasks

Sometimes a program has to wait for *whichever* of several things happens first. Go provides a dedicated statement, `select`, for that purpose. Swift handles the same situations with tools we already have, in two styles: race several children in a task group and keep the first answer, or funnel several sources of events into a single asynchronous sequence and process them in arrival order.

### 8.7.1. The First Answer Wins

Imagine a resource available from several mirrors, some of which may be slow on any given day. Ask all of them, use whichever answers first, and abandon the rest:

```swift
// swiftpl/ch8/fastest
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

func fastestMirror() async throws -> Data {
    try await withThrowingTaskGroup(of: Data.self) { group in
        group.addTask { try await request("https://asia.example.com") }
        group.addTask { try await request("https://europe.example.com") }
        group.addTask { try await request("https://americas.example.com") }
        defer { group.cancelAll() }
        return try await group.next()!
    }
}

func request(_ host: String) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(from: URL(string: host)!)
    return data
}
```

`group.next()` returns the first result to arrive. The `defer` cancels the remaining children on the way out, and, because the group is structured, `withThrowingTaskGroup` doesn't actually return until those losers have wound down. `URLSession` notices cancellation and abandons its request, so that happens almost immediately. No losing request can be left running in the background.

Note that "first result" includes failures. If one mirror fails quickly, perhaps because its host name doesn't resolve, `next()` rethrows that error and `fastestMirror` gives up, even though the other mirrors might have succeeded. Whether that's right depends on the application; Exercise 8.11 asks for the alternative.

Racing an operation against a timer gives a timeout:

```swift
// swiftpl/ch8/fastest (continued)
struct TimeoutError: Error {}

func withTimeout<T: Sendable>(
    _ timeout: Duration, _ operation: @escaping @Sendable () async throws -> T
) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask { try await operation() }
        group.addTask {
            try await Task.sleep(for: timeout)
            throw TimeoutError()
        }
        defer { group.cancelAll() }
        return try await group.next()!
    }
}

let data = try await withTimeout(.seconds(5)) { try await request("https://example.com") }
```

If the operation wins, its value is returned and the sleeper is cancelled. If the sleeper wins, it throws `TimeoutError`, and the operation is cancelled. The `@Sendable` annotation on `operation` is required because the closure runs in a child task; the compiler will check that whatever the caller's closure captures can safely cross into that task.

### 8.7.2. One Loop, Many Sources

The second style suits a long-running task that must react to events from several independent sources. The trick is to give every source the same continuation, so that their events interleave into one stream, and then handle that stream with a single `for await` loop.

The program below is a timed quiz. It asks a question and gives the user ten seconds to answer, counting down the last few seconds aloud. Two event sources are involved: a clock that ticks every second, and the keyboard.

```swift
// swiftpl/ch8/quiz
import Foundation

enum Event: Sendable {
    case tick
    case answer(String)
}

let (events, eventsIn) = AsyncStream.makeStream(of: Event.self)

// readLine blocks its thread, so it gets a thread of its own.
Thread.detachNewThread {
    eventsIn.yield(.answer(readLine() ?? ""))
}

// One tick per second.
let clock = Task {
    while !Task.isCancelled {
        try? await Task.sleep(for: .seconds(1))
        eventsIn.yield(.tick)
    }
}

print("What is 7 × 8? You have 10 seconds.")
var remaining = 10
loop: for await event in events {
    switch event {
    case .tick:
        remaining -= 1
        if remaining == 0 {
            print("Time's up! The answer is 56.")
            break loop
        }
        if remaining <= 3 {
            print("\(remaining)...")
        }
    case .answer(let text):
        let correct = text.trimmingCharacters(in: .whitespaces) == "56"
        print(correct ? "Correct!" : "Sorry, the answer is 56.")
        break loop
    }
}
clock.cancel()
```

Whatever happens first, a final tick or an answer, ends the loop. `break loop` names the loop explicitly, since a bare `break` inside a `switch` would only leave the `switch`. A continuation may be shared by any number of producers, which is what makes this pattern work. When the program ends, the thread still blocked in `readLine` is simply discarded with the process.

When the sources are already asynchronous sequences, the `merge` function from `swift-async-algorithms` combines them without a hand-built stream, and its `AsyncTimerSequence` provides ticks: `for await event in merge(ticks, answers) { ... }`.

**Exercise 8.10:** Rewrite `quiz` using `merge` and `AsyncTimerSequence` from `swift-async-algorithms`.

**Exercise 8.11:** Write a version of `download` (Section 1.5) that takes several URLs for the same resource, requests them all concurrently, prints whichever successful response arrives first, and cancels the others. A request that fails shouldn't end the race unless every request has failed.

## 8.8. Example: Concurrent Directory Traversal

Our next program, `dirsize`, reports how many files a directory tree contains and how much space they use, in the spirit of the Unix `du` command. Walking a directory tree is a good test of concurrency techniques because the work is irregular: some directories are tiny, some enormous, and you can't know which until you look.

The building blocks are a small value type for the totals and a function that lists a directory:

```swift
// swiftpl/ch8/dirsize1
import Foundation

struct Usage: Sendable {
    var files = 0
    var bytes = 0

    static func + (a: Usage, b: Usage) -> Usage {
        Usage(files: a.files + b.files, bytes: a.bytes + b.bytes)
    }
}

/// Returns the entries of directory dir.
func entries(of dir: URL) -> [URL] {
    do {
        return try FileManager.default.contentsOfDirectory(
            at: dir, includingPropertiesForKeys: [.isDirectoryKey, .fileSizeKey])
    } catch {
        printError("dirsize: \(error.localizedDescription)")
        return []
    }
}

/// Recursively walks the file tree rooted at dir and returns the
/// number of files and their total size.
func walkDir(_ dir: URL) -> Usage {
    var usage = Usage()
    for entry in entries(of: dir) {
        let values = try? entry.resourceValues(forKeys: [.isDirectoryKey, .fileSizeKey])
        if values?.isDirectory == true {
            usage = usage + walkDir(entry)
        } else {
            usage.files += 1
            usage.bytes += values?.fileSize ?? 0
        }
    }
    return usage
}

func printError(_ message: String) {
    FileHandle.standardError.write(Data((message + "\n").utf8))
}
```

The main code walks each directory named on the command line, or the current directory if there are none, and prints the combined result:

```swift
// swiftpl/ch8/dirsize1 (continued)
// Determine the initial directories.
var roots = CommandLine.arguments.dropFirst().map { URL(filePath: $0) }
if roots.isEmpty {
    roots = [URL(filePath: ".")]
}
let usage = roots.map(walkDir).reduce(Usage(), +)
printDiskUsage(usage)

func printDiskUsage(_ u: Usage) {
    print(String(format: "%d files  %.1f GB", u.files, Double(u.bytes) / 1e9))
}
```

On a large home directory, this sequential version can run for many seconds before printing its one line.

### 8.8.1. Walking in Parallel

The subdirectories of any directory can be measured independently, so the recursion itself can fan out. Each call to `walkDir` becomes the root of a small task group, with one child per subdirectory:

```swift
// swiftpl/ch8/dirsize2 (builds on dirsize1)
func walkDir(_ dir: URL) async -> Usage {
    await withTaskGroup(of: Usage.self) { group in
        var usage = Usage()
        for entry in entries(of: dir) {
            let values = try? entry.resourceValues(forKeys: [.isDirectoryKey, .fileSizeKey])
            if values?.isDirectory == true {
                group.addTask { await walkDir(entry) }
            } else {
                usage.files += 1
                usage.bytes += values?.fileSize ?? 0
            }
        }
        return await group.reduce(usage, +)
    }
}
```

Because `walkDir` is now `async`, it can no longer be passed to `map`, which expects a synchronous function. The main code awaits each root's total instead:

```swift
// swiftpl/ch8/dirsize2 (continued)
// Replaces dirsize1's `let usage = roots.map(walkDir).reduce(Usage(), +)`.
var usage = Usage()
for root in roots {
    usage = usage + (await walkDir(root))
}
```

The shape of the code mirrors the shape of the data. Files are counted on the spot; each subdirectory gets a child task, which may spawn children of its own; and each level adds its own files to the totals reported by its children. The result is a tree of tasks with the same structure as the tree of directories, and structured concurrency guarantees that when the top-level call returns, every task in that tree has finished.

`entries(of:)` makes blocking system calls, which, as Section 8.5 warned, can tie up the thread pool. Here the pool's small size works in our favor: at most one directory listing per core can be in progress, which keeps the program from flooding the file system with requests. On a machine with a fast SSD, this version finishes several times sooner than the sequential one.

### 8.8.2. Reporting Progress

A long-running tool should show signs of life. Let's add an optional `-v` flag that prints running totals twice a second. The design is the event loop of Section 8.7 again: the walker produces an event per file, a clock produces ticks, and a single loop consumes both.

```swift
// swiftpl/ch8/dirsize3 (builds on dirsize1)
enum Event: Sendable {
    case file(bytes: Int)
    case tick
    case done
}

let verbose = CommandLine.arguments.contains("-v")
let paths = CommandLine.arguments.dropFirst().filter { $0 != "-v" }
let roots = (paths.isEmpty ? ["."] : paths).map { URL(filePath: $0) }
let (events, eventsIn) = AsyncStream.makeStream(of: Event.self)

Task.detached {
    for root in roots {
        walkDir(root) { bytes in eventsIn.yield(.file(bytes: bytes)) }
    }
    eventsIn.yield(.done)
}
let clock = Task {
    while verbose && !Task.isCancelled {
        try? await Task.sleep(for: .milliseconds(500))
        eventsIn.yield(.tick)
    }
}

var usage = Usage()
loop: for await event in events {
    switch event {
    case .file(let bytes):
        usage.files += 1
        usage.bytes += bytes
    case .tick:
        printDiskUsage(usage)
    case .done:
        break loop
    }
}
clock.cancel()
printDiskUsage(usage)  // final totals
```

For this version, `walkDir` reports each file to a closure as it finds it, instead of returning a total:

```swift
// swiftpl/ch8/dirsize3 (continued)
func walkDir(_ dir: URL, _ found: (Int) -> Void) {
    for entry in entries(of: dir) {
        let values = try? entry.resourceValues(forKeys: [.isDirectoryKey, .fileSizeKey])
        if values?.isDirectory == true {
            walkDir(entry, found)
        } else {
            found(values?.fileSize ?? 0)
        }
    }
}
```

The walk runs in a detached task, so that it doesn't occupy the main actor, and the main code is free to print. Every half second, the user sees the totals so far:

```
$ .build/release/dirsize3 -v ~
31206 files  4.7 GB
70318 files  9.2 GB
...
```

**Exercise 8.12:** `dirsize3` walks sequentially. Combine it with `dirsize2` so that the walk is parallel *and* reports progress. Where should the events be produced?

**Exercise 8.13:** Print a separate line of totals for each root directory, updated periodically, instead of one combined line.

## 8.9. Cancellation

Plenty of work turns out to be unnecessary partway through. The user changes their mind, the client hangs up, a search finds its answer in the first place it looks. A good concurrent program stops doing work that no longer matters, and in Swift the mechanism for that is *cancellation*.

Cancellation is a request, not a command. Calling `cancel()` on a task, or `cancelAll()` on a task group, marks the task as cancelled, and the mark spreads to every child task beneath it. Nothing is interrupted. Instead, code checks for the mark at points where stopping makes sense, using one of these mechanisms:

- `Task.isCancelled` is a Boolean to poll. Long loops should consult it now and then.
- `try Task.checkCancellation()` throws `CancellationError` if the task is cancelled, which suits throwing code.
- Many asynchronous APIs check for you. `Task.sleep` throws `CancellationError` as soon as the task is cancelled, `URLSession` gives up on its request and throws `URLError(.cancelled)`, and iterating an `AsyncStream` ends early.
- `withTaskCancellationHandler(operation:onCancel:)` runs a handler the moment cancellation happens, so that code built on another mechanism, such as a callback API or a socket, can be told to stop.

Why make it cooperative? Because stopping a task at an arbitrary instruction is dangerous. It might be halfway through updating a data structure, holding a lock, or writing a file. Letting the task choose its own stopping points means it can always stop in a consistent state.

Let's make `dirsize` stoppable from the keyboard: pressing Return should end the walk promptly and print whatever has been counted so far. The walk runs in a task, and a dedicated thread waits for input and cancels it:

```swift
// swiftpl/ch8/dirsize4 (builds on dirsize2)
let walk = Task {
    var total = Usage()
    for root in roots {
        total = total + (await walkDir(root))
    }
    return total
}

Thread.detachNewThread {
    if readLine() != nil {  // a line was entered, not the end of input
        walk.cancel()
    }
}

let usage = await walk.value
printDiskUsage(usage)
if walk.isCancelled {
    print("(cancelled: totals are incomplete)")
}
```

`walk.value` suspends until the task produces its result.

The parallel `walkDir` from Section 8.8 needs only one change to honor the request. Cancelling `walk` cancels every task group it contains, and every child in those groups, so each call just has to look before doing any work:

```swift
// swiftpl/ch8/dirsize4 (continued)
func walkDir(_ dir: URL) async -> Usage {
    if Task.isCancelled {
        return Usage()
    }
    return await withTaskGroup(of: Usage.self) { group in
        // ...as before...
    }
}
```

One check per directory is enough to make the walk stop within a directory or two of the keypress. Children added to a group after cancellation start out cancelled themselves, so they hit the check and return immediately.

No further cleanup is required, and that's the payoff of structure. Each task group waits for its children, so by the time `walk.value` delivers a result, every task the walk ever started has completed. There are no stragglers to track down.

Because cancellation interrupts nothing, the usual tools for cleanup still apply. `defer` blocks run, `catch` clauses catch, and a `CancellationError` propagates like any other error.

**Exercise 8.14:** `dirsize4` measures its roots one at a time. Measure them concurrently in a task group instead. Does cancellation still reach every task? How can you tell?

**Exercise 8.15:** Make the crawler of Section 8.6 cancellable from the keyboard, and confirm that fetches in progress are abandoned rather than allowed to finish.

**Exercise 8.16:** Write `search(_ paths: [String], for needle: String) async -> String?`, which looks through many files concurrently and returns the first one containing `needle`, cancelling all remaining work once it has an answer.

## 8.10. Example: Chat Server

We'll end the chapter with a program that pulls these pieces together: a chat room. Clients connect with any TCP client, give a name, and from then on, every line any of them types is shown to all of them.

The interesting problem is the *membership list*. Each client's connection is handled by its own task, yet sending a message means reaching every other client, so all those tasks need access to the current set of members. Rather than share that set and protect it, we'll give it to one task, the *room*, which is its sole owner. Every other task talks to the room by sending it events through a stream, much as a customer talks to a shop through its counter rather than walking into the stockroom. (Chapter 9 introduces the *actor*, a language feature that packages up exactly this pattern.)

Three kinds of event reach the room: someone joins, someone leaves, and someone says something. Here are the event type and the server's main loop, which reuses the line-oriented setup from Section 8.3:

```swift
// swiftpl/ch8/chat (builds on remind1)
enum RoomEvent: Sendable {
    case join(id: Int, name: String, inbox: AsyncStream<String>.Continuation)
    case leave(id: Int)
    case say(String)
}

let (roomEvents, toRoom) = AsyncStream.makeStream(of: RoomEvent.self)

try await withThrowingDiscardingTaskGroup { group in
    group.addTask { await room() }
    var nextID = 0
    try await server.executeThenClose { connections in
        for try await connection in connections {
            let id = nextID
            nextID += 1
            group.addTask { try? await serve(connection, id: id) }
        }
    }
}
```

Every member has an *inbox*: a stream of lines waiting to be sent to that member. When someone joins, they hand the room the sending end of their inbox. The room keeps those, and nothing else:

```swift
// swiftpl/ch8/chat (continued)
func room() async {
    var members: [Int: (name: String, inbox: AsyncStream<String>.Continuation)] = [:]
    for await event in roomEvents {
        switch event {
        case .join(let id, let name, let inbox):
            for member in members.values {
                member.inbox.yield("* \(name) joined")
            }
            inbox.yield("* \(members.count) other(s) here")
            members[id] = (name, inbox)
        case .leave(let id):
            guard let gone = members.removeValue(forKey: id) else { break }
            gone.inbox.finish()
            for member in members.values {
                member.inbox.yield("* \(gone.name) left")
            }
        case .say(let line):
            for member in members.values {
                member.inbox.yield(line)
            }
        }
    }
}
```

`members` is a local variable of the room task. No other task can reach it, however many of them ask the room to change it, so it needs no lock. The rule of thumb is: don't share state between tasks; give it to one task and let the others make requests.

Each connection is served by a function that runs in its own task and starts one helper task of its own:

```swift
// swiftpl/ch8/chat (continued)
func serve(_ connection: NIOAsyncChannel<ByteBuffer, ByteBuffer>, id: Int) async throws {
    let (inbox, inboxIn) = AsyncStream.makeStream(of: String.self)

    try await connection.executeThenClose { inbound, outbound in
        try await withThrowingDiscardingTaskGroup { group in
            // Deliver everything that arrives in the inbox to the client.
            group.addTask {
                for await line in inbox {
                    try await outbound.write(ByteBuffer(string: line + "\n"))
                }
            }

            var lines = inbound.makeAsyncIterator()
            inboxIn.yield("Welcome! What's your name?")
            guard let first = try await lines.next() else {
                inboxIn.finish()
                return
            }
            let name = String(buffer: first)
            toRoom.yield(.join(id: id, name: name, inbox: inboxIn))
            defer { toRoom.yield(.leave(id: id)) }

            while let line = try await lines.next() {
                toRoom.yield(.say("[\(name)] \(String(buffer: line))"))
            }
        }
    }
}
```

The helper task drains the member's inbox onto the network. Keeping that job separate means a member who is slow to receive doesn't stall the task reading their input, nor the room.

The rest of `serve` handles the conversation. It asks for a name, announces the arrival to the room, and then forwards each line the client types. If the client sends nothing at all before disconnecting, `serve` finishes the inbox itself so that the delivery task can end. Otherwise, the `defer` statement guarantees that the room hears about the departure however the reading loop ends, whether the client hung up politely or the connection failed. On receiving `leave`, the room finishes that member's inbox, which ends the delivery task, which lets the task group, and with it `serve`, complete. If the connection fails with an error instead, the group cancels the delivery task directly, and the inbox's `for await` loop ends because its task was cancelled.

Here's a short conversation. Ada connects first, then Grace. This is Ada's terminal:

```
$ nc localhost 8000
Welcome! What's your name?
ada
* 0 other(s) here
* grace joined
hello, grace
[ada] hello, grace
[grace] hi! compiling anything fun?
```

and this is Grace's:

```
$ nc localhost 8000
Welcome! What's your name?
grace
* 1 other(s) here
[ada] hello, grace
hi! compiling anything fun?
[grace] hi! compiling anything fun?
```

(Each client sees its own lines echoed back by the room, since `say` goes to every member including the sender.)

With *n* people in the room, the server runs 2*n* + 2 tasks: the main task, the room, and two per member. Not one of them uses a lock. The only values touched by more than one task are the streams and their continuations, which are designed for exactly that. The next chapter is about what to do when that kind of confinement isn't practical, and state really must be shared.

**Exercise 8.17:** Add a `/who` command that lists the members currently in the room. The room handles it, but only the member who asked should see the answer.

**Exercise 8.18:** Disconnect members who have said nothing for five minutes. (Hint: wrap each read in `withTimeout` from Section 8.7.)

**Exercise 8.19:** Every inbox is unbounded, so a member who never reads makes the server's memory grow without limit. Make inboxes keep only the newest 100 lines, and tell a member when their lines have been dropped.

**Exercise 8.20:** Reimplement the room as an actor (Section 9.3) with methods `join`, `leave`, and `say`, and compare the two versions for clarity.
