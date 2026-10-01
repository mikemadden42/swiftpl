# 8. Tasks and Asynchronous Sequences

Concurrent programming, the expression of a program as a composition of several autonomous activities, has never been more important than it is today. Web servers handle requests for thousands of clients at once. Phone and desktop apps render animations in the user interface while simultaneously performing computation and network requests in the background. Even traditional batch problems (read some data, compute, write some output) use concurrency to hide the latency of I/O operations and to exploit a modern computer's many processors, which every year grow in number but not in speed.

Swift's concurrency model is built into the language. It has three pillars: *`async`/`await`*, which lets functions suspend while waiting for something without blocking a thread; *structured concurrency*, in which concurrent work is organized into a tree of *tasks* whose lifetimes are bounded by the scopes that create them; and *data-race safety*, in which the compiler checks that values shared between concurrent tasks are safe to share. This chapter covers the first two, and introduces *asynchronous sequences*, which play the role that channels play in Go: they let one task deliver a stream of values to another. Chapter 9 covers the third pillar, along with the more traditional tools of shared-memory concurrency, mutexes and actors.

Even though Swift's support for concurrency is one of its great strengths, reasoning about concurrent programs is inherently harder than about sequential ones, and intuitions acquired from sequential programming may at times lead us astray. If this is your first encounter with concurrency, we recommend spending a little extra time thinking about the examples in these two chapters.

## 8.1. Tasks and `async`/`await`

In Swift, each concurrently executing activity is called a *task*. Consider a program that has two functions, one that does some computation and one that writes some output, and assume that neither function calls the other. A sequential program may call one function and then call the other, but in a *concurrent* program with two or more tasks, calls to both functions can be active at the same time. As we'll see in a moment, such a program is like one that has been divided among several workers.

If you have used operating system threads or threads in other languages, then you can assume for now that a task is similar to a thread, and you'll be able to write correct programs. The differences are important for performance, though: a task is much cheaper than a thread. Swift runs all tasks on a small, fixed pool of threads, usually one per processor core, called the *cooperative thread pool*. When a task reaches an `await` and has to wait (for a network response, a timer, or another task) it *suspends*, releasing its thread to run other tasks, and *resumes* later, possibly on a different thread. So a program can have tens of thousands of tasks, most of them suspended, while using only a handful of threads.

When a program starts, its top-level code (or its `@main` type's `main` method) runs in the *main task*. New tasks are created in several ways, which we'll meet throughout this chapter. The most direct is the `Task` initializer, which starts a new, *unstructured* task to run a closure:

```swift
f()  // call f(); wait for it to return
Task { await f() }  // create a new task that calls f(); don't wait
```

`Task.detached` does the same, except that the new task does not inherit the *actor isolation* of the code that created it; we'll explain what that means in Chapter 9. For now, it means that a detached task always runs on the cooperative thread pool, never on the main thread.

In the example below, the main task computes the 45th Fibonacci number. Since it uses the terribly inefficient recursive algorithm, it runs for an appreciable time, during which we'd like to provide the user with a visual indication that the program is still running, by displaying an animated textual "spinner."

```swift
// swiftpl/ch8/spinner
import Foundation

let spinner = Task.detached {
    await spin(delay: .milliseconds(100))
}
let n = 45
let fibN = fib(n)  // slow
spinner.cancel()
print("\rFibonacci(\(n)) = \(fibN)")

func spin(delay: Duration) async {
    while !Task.isCancelled {
        for r in #"-\|/"# {
            FileHandle.standardOutput.write(Data("\r\(r)".utf8))
            try? await Task.sleep(for: delay)
        }
    }
}

func fib(_ x: Int) -> Int {
    x < 2 ? x : fib(x - 1) + fib(x - 2)
}
```

After several seconds of animation, the `fib(45)` call returns and the main task prints its result:

```
Fibonacci(45) = 1134903170
```

The `spin` function is *asynchronous*: it's marked `async` because it calls `Task.sleep`, which suspends the task for a while. The spinner task loops until it notices that it has been *cancelled*, which the main task requests by calling `spinner.cancel()` after the computation is done. Cancellation in Swift is *cooperative*: cancelling a task just sets a flag, which the task checks with `Task.isCancelled`, and which makes operations like `Task.sleep` return early. We'll discuss cancellation in Section 8.9.

We use `FileHandle` rather than `print` to write the spinner characters because `print` buffers its output until it sees a newline.

When the top-level code finishes, the program exits, and any tasks still running are abruptly terminated. Other than by returning from the main code or exiting the program, there is no programmatic way for one task to stop another, but as we'll see later, there are ways to communicate with a task to request that it stop itself.

Notice how the program is expressed as the composition of two autonomous activities, spinning and Fibonacci computation. Each is written as a separate function, but both make progress concurrently.

### 8.1.1. `async` and `await`

An `async` function can do something no ordinary function can: it can suspend. Every possible suspension point inside it is marked with `await`, so readers can see where the function might pause and other code might run. An `async` function can be called only from another `async` context, with `await`: from another `async` function, from a task's closure, or from top-level code that uses `await`.

```swift
func fetchUser(id: Int) async throws -> User { ... }

let user = try await fetchUser(id: 42)  // suspends until the result is ready
```

In Go, every function can block, and the runtime arranges for a blocked goroutine to give up its thread. In Swift, blocking is explicit in the type system: `async` functions may suspend, and synchronous functions may not. This is sometimes called *function coloring*, and it has a cost. You can't call an `async` function from a synchronous one without creating a task. But it also has benefits. A synchronous function is guaranteed to run to completion without interruption from other code on the same actor, and the `await` keyword makes every point where the world might change visible in the code.

Swift's asynchronous functions are not threads. When an `async` function calls another, and the callee doesn't need to suspend, the call is about as cheap as an ordinary function call. When it does suspend, its local variables are saved in a heap-allocated frame, and its thread is released. There is no fixed-size stack per task, so there's no stack to grow, and tasks are cheap: creating one costs a small allocation.

### 8.1.2. Structured Concurrency

Unstructured tasks created with `Task { }` are useful, but in most code you'll use *structured* concurrency, in which child tasks are created within a scope and can't outlive it. We saw one form in Section 1.6, the task group. The other is `async let`, which starts a child task to compute a value, and lets you `await` the value later:

```swift
async let a = fetchUser(id: 1)  // starts a child task
async let b = fetchUser(id: 2)  // starts another child task
let users = try await [a, b]  // waits for both
```

The two fetches run concurrently. If the scope exits before the values are awaited (say, because of a thrown error), the child tasks are cancelled and awaited automatically. A child task can never be "leaked."

Structure gives the program a tree of tasks, and the tree is what makes cancellation, priority, and task-local values propagate: when a parent is cancelled, so are all its children, recursively. It also makes programs easier to reason about, for the same reason that structured programming made `goto` unnecessary: concurrent work starts and ends within a visible region of the code.

Prefer `async let` and task groups. Reach for unstructured `Task { }` only when the new work genuinely needs to outlive the current scope, for example when starting work from synchronous code like an event handler.

## 8.2. Example: Concurrent Clock Server

Networking is a natural domain in which to use concurrency, since servers typically handle many connections from their clients at once, each client being essentially independent of the others. In this section, we'll use SwiftNIO, the high-performance networking library that underlies most server-side Swift, including the Hummingbird server we used in Chapter 1. NIO's `NIOAsyncChannel` type presents a network connection as an asynchronous sequence of incoming messages and an asynchronous writer for outgoing ones, which fits nicely with Swift's concurrency model.

Our first example is a sequential clock server that writes the current time to the client once per second:

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

The package depends on `https://github.com/apple/swift-nio` and uses its `NIOCore` and `NIOPosix` products.

`ServerBootstrap` creates a *listener*, an object that listens for incoming connections on a network port, in this case TCP port 8000 on `localhost`. The closure passed to `bind` is called for each new connection to configure it; ours wraps the connection's NIO channel in an `NIOAsyncChannel` that reads and writes `ByteBuffer`s.

The result of `bind` is itself an `NIOAsyncChannel` whose inbound messages are connections. `executeThenClose` gives us access to its inbound stream, and ensures that the listener is closed when we're done. The `for try await` loop waits for each new connection and handles it. A `for`-`in` loop over an asynchronous sequence uses `await` because each call to fetch the next element may suspend, and `try` because it may throw.

The `handle` function handles one complete client connection. In a loop, it writes the current time to the client. Since the output writer's `write` is asynchronous, and won't complete until the data has been handed off to the network, the server naturally slows down if the client can't keep up. When the client disconnects, `write` throws an error, which ends the loop and closes the connection, and the server goes back to waiting for another connection request.

To connect to the server, we'll use the standard `nc` ("netcat") program, a utility for manipulating network connections:

```
$ swift build
$ .build/debug/clock1 &
[1] 2100
$ nc localhost 8000
13:58:54
13:58:55
13:58:56
13:58:57
^C
```

The client displays the time sent by the server each second until we interrupt the client with Control-C, which on Unix systems is echoed as `^C` by the shell.

If we run two clients at the same time on different terminals, one shown to the left and one to the right, the second client must wait until the first client is finished, because the server is *sequential*: it deals with only one client at a time.

```
$ nc localhost 8000              |
13:58:54                         | $ nc localhost 8000
13:58:55                         |
13:58:56                         |
^C                               |
                                 | 13:58:57
                                 | 13:58:58
                                 | 13:58:59
                                 | ^C
$ killall clock1
```

Just one small change is needed to make the server concurrent: handling each connection in a new child task, rather than in the main task. We'll create the child tasks in a *discarding task group*, a kind of task group that doesn't collect its children's results, which is just what a server needs, since each child task produces no result and the group frees each child's resources as soon as it finishes:

```swift
// swiftpl/ch8/clock2
try await withThrowingDiscardingTaskGroup { group in
    try await server.executeThenClose { connections in
        for try await connection in connections {
            group.addTask { await handle(connection) }  // handle connections concurrently
        }
    }
}
```

Now, multiple clients can receive the time at once:

```
$ nc localhost 8000              |
14:02:54                         | $ nc localhost 8000
14:02:55                         | 14:02:55
14:02:56                         | 14:02:56
14:02:57                         | ^C
14:02:58                         |
^C                               |
```

**Exercise 8.1:** Modify `clock2` to accept a port number, and write a program, `clockwall`, that acts as a client of several clock servers at once, reading the times from each one and displaying the results in a table, akin to the wall of clocks seen in some business offices. If you have access to geographically distributed computers, run instances remotely; otherwise run local instances on different ports with fake time zones (using `TimeZone(identifier:)`).

```
$ TZ=US/Eastern    ./clock2 -port 8010 &
$ TZ=Asia/Tokyo    ./clock2 -port 8020 &
$ TZ=Europe/London ./clock2 -port 8030 &
$ clockwall NewYork=localhost:8010 Tokyo=localhost:8020 London=localhost:8030
```

**Exercise 8.2:** Write a simple `netcat` replacement, a client that connects to a server and copies its output to the standard output, using NIO's `ClientBootstrap`.

## 8.3. Example: Concurrent Echo Server

The clock server used one task per connection. In this section, we'll build an echo server that uses multiple tasks per connection. Most echo servers merely write whatever they read, which can be done with this trivial loop:

```swift
for try await buffer in inbound {
    try await outbound.write(buffer)
}
```

A more interesting echo server might simulate the reverberations of a real echo, with the response loud at first ("HELLO!"), then moderate ("Hello!") after a delay, then quiet ("hello!") before fading to nothing. To do that, the server needs to receive its input a *line* at a time. NIO's companion package `swift-nio-extras` provides a *frame decoder* that splits a byte stream into lines, which we install in the connection's *pipeline* when we configure it:

```swift
// swiftpl/ch8/reverb1
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
            try await echo(String(buffer: line), delay: .seconds(1), to: outbound)
        }
    }
}

func echo(_ shout: String, delay: Duration,
          to out: NIOAsyncChannelOutboundWriter<ByteBuffer>) async throws {
    try await out.write(ByteBuffer(string: "\t\(shout.uppercased())\n"))
    try await Task.sleep(for: delay)
    try await out.write(ByteBuffer(string: "\t\(shout)\n"))
    try await Task.sleep(for: delay)
    try await out.write(ByteBuffer(string: "\t\(shout.lowercased())\n"))
}
```

Each element of `inbound` is now a single line, without its newline, and `String(buffer:)` decodes it as UTF-8. Let's run a session, typing lines into `nc` (shown without indentation) and seeing the server's responses (indented):

```
$ swift build
$ .build/debug/reverb1 &
$ nc localhost 8000
Hello?
	HELLO?
	Hello?
	hello?
Is there anybody there?
	IS THERE ANYBODY THERE?
Yooo-hooo!
	Is there anybody there?
	is there anybody there?
	YOOO-HOOO!
	Yooo-hooo!
	yooo-hooo!
^C
```

Notice that the third shout from the client is not dealt with until the second shout has petered out, which is not very realistic. A real echo would consist of the *composition* of the three independent shouts. To simulate it, we'll need more tasks. Again, all we need to do is to echo each line in a child task:

```swift
// swiftpl/ch8/reverb2
func handle(_ connection: NIOAsyncChannel<ByteBuffer, ByteBuffer>) async throws {
    try await connection.executeThenClose { inbound, outbound in
        try await withThrowingDiscardingTaskGroup { group in
            for try await line in inbound {
                let shout = String(buffer: line)
                group.addTask { try await echo(shout, delay: .seconds(1), to: outbound) }
            }
        }
    }
}
```

The outbound writer is `Sendable`, safe to use from several tasks at once, so the concurrent echoes can share it. Now the echoes overlap:

```
$ nc localhost 8000
Is there anybody there?
	IS THERE ANYBODY THERE?
Yooo-hooo!
	Is there anybody there?
	YOOO-HOOO!
	is there anybody there?
	Yooo-hooo!
	yooo-hooo!
^C
```

All that was required to make the server use concurrency, not just to handle connections from multiple clients but even within a single connection, was to add a task group and an `addTask` call.

However, in adding these tasks, we had to consider carefully that it's safe to call methods of `outbound` concurrently, which is not true for most types. Fortunately, in Swift 6, we don't have to rely on care alone. The compiler checks that everything captured by the `addTask` closure (here `shout`, `outbound`, and the `echo` function) is `Sendable`, and rejects the program if it isn't. We'll discuss this in detail in Chapter 9.

**Exercise 8.3:** In `reverb2`, the group waits for all echoes to finish before the connection is closed. Is that desirable? What happens if the client disconnects while echoes are pending?

**Exercise 8.4:** Add a timeout to the echo server so that it disconnects any client that shouts nothing within 10 seconds.

## 8.4. Async Streams

If tasks are the activities of a concurrent Swift program, *asynchronous sequences* are the connections between them. An asynchronous sequence is like an ordinary sequence, except that producing each element may require waiting. It's described by the protocol `AsyncSequence`, and consumed with `for await`. We've already seen several: the stream of connections accepted by a server, the stream of lines from a client, and the results of a task group.

The standard library's `AsyncStream` is a general-purpose asynchronous sequence that one task can use to send values to another, much as a Go channel does. Each stream is created together with a *continuation*, the sending end:

```swift
let (stream, continuation) = AsyncStream.makeStream(of: Int.self)
```

The producer calls `continuation.yield(x)` to send a value and `continuation.finish()` to indicate that no more values will be sent. The consumer iterates over the stream with `for await`, which receives values in order and ends when the stream is finished and its buffer drained:

```swift
Task {
    for i in 1...3 {
        continuation.yield(i)
    }
    continuation.finish()
}
for await x in stream {
    print(x)  // "1", "2", "3"
}
print("done")
```

Unlike a Go channel, the two ends of an `AsyncStream` are separate values of different types, so a function that should only send can be given only the continuation, and one that should only receive can be given only the stream. That's the same restriction that Go expresses with unidirectional channel types like `chan<- int` and `<-chan int`.

### 8.4.1. Buffering

When the producer calls `yield` and the consumer isn't waiting, the value is buffered. By default, the buffer is unbounded: `yield` never waits, so a fast producer and a slow consumer can make the buffer grow without limit. An `AsyncStream` can instead be created with a *buffering policy* that keeps only the oldest or newest *n* values, discarding the rest:

```swift
let (stream, continuation) = AsyncStream.makeStream(of: Int.self, bufferingPolicy: .bufferingNewest(1))
```

This is useful for "latest value wins" situations such as progress updates. `yield` returns a result that tells the producer whether the value was enqueued, dropped, or the stream has been terminated.

What `AsyncStream` doesn't provide is *back-pressure*: a way for a slow consumer to make the producer wait, as an unbuffered Go channel does. For that, use `AsyncChannel` from the `swift-async-algorithms` package, whose `send` method is `async` and doesn't return until a consumer has received the value:

```swift
import AsyncAlgorithms

let channel = AsyncChannel<Int>()
Task {
    for i in 1...3 {
        await channel.send(i)  // waits for a receiver
    }
    channel.finish()
}
for await x in channel {
    print(x)
}
```

This is the closest analogue in Swift to Go's unbuffered channel, and it creates the same kind of synchronization between sender and receiver. `swift-async-algorithms` also provides `AsyncThrowingChannel` and a library of operators for asynchronous sequences, including `merge`, `zip`, `combineLatest`, `debounce`, `throttle`, `chunked`, and `timer`.

An important difference from Go: an `AsyncStream` is designed for a *single consumer*. Iterating over the same stream from two tasks at once is a programming error. To distribute work to several consumers, use a task group; to broadcast values to several consumers, give each one its own stream, as we'll do in Section 8.10.

### 8.4.2. Pipelines

Streams can be used to connect tasks together so that the output of one is the input to another. This is called a *pipeline*. The program below consists of three tasks connected by two streams.

```
Counter --naturals--> Squarer --squares--> Printer
```

The first task, `counter`, generates the integers 0, 1, 2, ..., and sends them over a stream to the second task, `squarer`, which receives each value, squares it, then sends the result over another stream to the third task, `printer`, which receives the squared values and prints them.

```swift
// swiftpl/ch8/pipeline1
let (naturals, naturalsOut) = AsyncStream.makeStream(of: Int.self)
let (squares, squaresOut) = AsyncStream.makeStream(of: Int.self)

await withDiscardingTaskGroup { group in
    // Counter
    group.addTask {
        for x in 0..<100 {
            naturalsOut.yield(x)
        }
        naturalsOut.finish()
    }

    // Squarer
    group.addTask {
        for await x in naturals {
            squaresOut.yield(x * x)
        }
        squaresOut.finish()
    }

    // Printer (in the parent task)
    for await x in squares {
        print(x)
    }
}
```

As we have seen, the producer of a stream signals that no further values will be sent by calling `finish()`. When the counter finishes after 100 elements, the squarer's loop ends once it has received them all, and it in turn finishes its own output stream, which ends the printer's loop. If a producer forgets to call `finish`, the consumer waits forever. (If the continuation is deallocated without `finish` having been called, the stream is finished automatically, which is a helpful safety net.)

Let's refactor the pipeline into separate functions. The functions are given the ends of the streams they use, making the direction of each stream clear:

```swift
// swiftpl/ch8/pipeline2
func counter(_ out: AsyncStream<Int>.Continuation) {
    for x in 0..<100 {
        out.yield(x)
    }
    out.finish()
}

func squarer(_ out: AsyncStream<Int>.Continuation, _ input: AsyncStream<Int>) async {
    for await v in input {
        out.yield(v * v)
    }
    out.finish()
}

func printer(_ input: AsyncStream<Int>) async {
    for await v in input {
        print(v)
    }
}

let (naturals, naturalsOut) = AsyncStream.makeStream(of: Int.self)
let (squares, squaresOut) = AsyncStream.makeStream(of: Int.self)
async let c: Void = counter(naturalsOut)
async let s: Void = squarer(squaresOut, naturals)
await printer(squares)
_ = await (c, s)
```

In practice, many pipelines don't need explicit streams at all, because asynchronous sequences support the same transformations as ordinary sequences: `map`, `filter`, `compactMap`, `prefix`, `reduce`, `contains`, and so on, each producing a new asynchronous sequence lazily. So the same pipeline can be expressed as:

```swift
let squares = naturals.map { $0 * $0 }
for await x in squares {
    print(x)
}
```

Each stage runs in the consumer's task, one element at a time, so this version isn't concurrent. That's often what you want; use separate tasks only when the stages genuinely benefit from running at the same time.

**Exercise 8.5:** Write a function `generate(_:)` that returns an `AsyncStream<Int>` producing the elements of an array, by creating a task that yields them. Then write the pipeline with it. What happens if the consumer stops iterating early? (Look up `AsyncStream.Continuation.onTermination`.)

## 8.5. Looping in Parallel

In this section, we'll explore some common concurrency patterns for executing all the iterations of a loop in parallel. We'll consider the problem of computing a cryptographic digest of each of a set of files, as a tool like `sha256sum` does. The `digest(of:)` function reads a file and computes its SHA256 hash:

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

Here's a sequential loop that computes the digests of a list of files:

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

Obviously the order in which we process the files doesn't matter, since each operation is independent of all the others. Problems like this that consist entirely of subproblems that are completely independent of each other are described as *embarrassingly parallel*. Embarrassingly parallel problems are the easiest kind to implement concurrently and enjoy performance that scales linearly with the amount of parallelism.

Let's execute all these operations in parallel, thereby hiding the latency of the file I/O and using multiple CPUs for the hash computations. A task group is the natural tool:

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

Each child task returns a tuple pairing the file name with its digest. The parent collects the results in whatever order the children finish, and stores them in a dictionary. Since only the parent touches the dictionary, there's no question of a data race.

Go programmers implementing this pattern must solve several problems by hand: waiting for all the goroutines to finish (with a `sync.WaitGroup` or by counting channel receives), avoiding *goroutine leaks* when returning early after an error (by using buffered channels or cancellation), and making sure the loop variable is captured correctly. Structured concurrency handles all of them. `withThrowingTaskGroup` doesn't return until every child has finished. If a child throws, the `for try await` loop rethrows the error in the parent; as the error propagates out of the group's scope, the group *cancels* the remaining children and waits for them to finish before rethrowing. No task can be leaked. And the `path` captured by each closure is a fresh constant in each iteration.

What if we want to compute a total, such as the number of bytes in all the files? A task group is an asynchronous sequence of its children's results, so all the usual sequence operations work on it, including `reduce`:

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

With a thousand files, our `digestFiles` creates a thousand child tasks at once. They don't all run at once, since there are only as many threads as cores, but they're all *started*, and each may hold memory and an open file. Often we want to limit the number of operations in progress. The idiomatic technique is a *sliding window*: start a fixed number of children, and each time one finishes, start another:

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

The nested function `startNext` captures both the iterator and the group. At any time, there are at most `maxConcurrent` children in the group.

There's one more subtlety in this example, which deserves a warning. `digest(of:)` does synchronous, *blocking* I/O: while `Data(contentsOf:)` reads the file, the thread can't run anything else. That's acceptable for local files, which are read quickly, but if the I/O might take a long time (a network file system, a slow device), blocking calls can tie up the entire cooperative thread pool and stall every other task in the program. Swift's thread pool assumes that tasks make *forward progress*. The rule of thumb is to avoid blocking calls in `async` code, and if you must make them, to run them on a dedicated thread or limit how many happen at once, as the sliding window does.

**Exercise 8.6:** Write a program `sha256sum` that prints the digest of each file named on its command line, in the same order as the arguments, but computes the digests in parallel.

**Exercise 8.7:** Measure the speedup of `digestFiles` as you vary `maxConcurrent` from 1 to 64 on a directory of large files. How does it relate to the number of cores on your machine?

## 8.6. Example: Concurrent Web Crawler

In Section 5.6, we made a simple web crawler that explored the link graph of the web in breadth-first order. In this section, we'll make it concurrent so that independent calls to `crawl` can exploit the I/O parallelism available in the web. The `crawl` function remains exactly as it was:

```swift
// swiftpl/ch8/crawl1
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
```

The main function resembles `breadthFirst` from Section 5.6. As before, a worklist records the queue of items that need processing, and a set records which items have been seen. Now, though, we'll start a child task for each URL, and keep at most 20 of them running at once:

```swift
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

The parent task is the only one that touches `worklist` and `seen`, so they need no protection. Each child performs a single `crawl`, and returns its links to the parent, which adds them to the worklist. `group.next()` waits for the next child to finish and returns its result, or `nil` when there are no more children. The loop terminates when the worklist is empty and no children are running.

Compared with the Go version of this program, there are fewer moving parts: no channel, no counting semaphore, and no need to count outstanding sends to know when to stop. The task group is all three.

We should note one difference in behavior. Because we pop from the end of the worklist, this crawler explores in roughly depth-first order. Using a `Deque` from the `swift-collections` package, with `popFirst()`, would restore breadth-first order.

The crawler now runs much faster, discovering hundreds of pages per second, until it is stopped (by Control-C) or runs out of memory. Unfortunately, it's also unbounded, so unless you aim it at a small site it will run, essentially, forever.

**Exercise 8.8:** Add depth-limiting to the concurrent crawler. That is, if the user sets `--depth 3`, then only URLs reachable by at most three links will be fetched. (Hint: put `(url, depth)` pairs in the worklist.)

**Exercise 8.9:** Write a program that makes a local mirror of a web site, fetching each reachable page and writing it to a directory on the local disk. Only pages within the original domain should be fetched. URLs within mirrored pages should be altered as needed so that they refer to the mirrored page, not the original.

## 8.7. Racing Tasks

Go has a `select` statement that waits on several channel operations at once and proceeds with whichever is ready first. Swift has no `select`, but the same effects are achieved in two ways: by *racing* child tasks in a task group, and by *merging* several event sources into one asynchronous sequence.

### 8.7.1. The First Result Wins

Suppose we have several mirrors of the same server and want the response of whichever answers first. We start one child task per mirror, take the first result, and cancel the rest:

```swift
// swiftpl/ch8/mirrored
func mirroredQuery() async throws -> Data {
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

The `defer` cancels the losers as the closure returns, and the group waits for them to finish cancelling before `withThrowingTaskGroup` returns. Because `URLSession` responds to cancellation by abandoning the request, the losers finish promptly. (A Go programmer must remember to use a buffered channel here, or the two slower goroutines will leak forever, blocked on sending a result no one will receive. Structured concurrency rules out that bug.)

The same pattern implements a timeout, by racing an operation against a sleep:

```swift
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

`@Sendable` on the closure parameter says that it may be called from another task, which it is, since it runs in a child task; the compiler checks that whatever the caller's closure captures is safe to share.

### 8.7.2. Merging Events

The second technique applies when a task must respond to several independent sources of events, as Go programs do with a `select` in a loop. The program below does the countdown for a rocket launch. It prints a countdown, one line per second, but the launch can be aborted by pressing the Return key. So the main task must respond to two kinds of events: a tick from a timer, and input from the keyboard. We'll merge them into a single stream:

```swift
// swiftpl/ch8/countdown
import Foundation

enum Event: Sendable {
    case tick
    case abort
}

let (events, eventsOut) = AsyncStream.makeStream(of: Event.self)

// Read a single byte from standard input, on a separate thread,
// since readLine blocks the calling thread.
Thread.detachNewThread {
    _ = readLine()
    eventsOut.yield(.abort)
}

// Produce a tick each second.
let ticker = Task {
    while !Task.isCancelled {
        try? await Task.sleep(for: .seconds(1))
        eventsOut.yield(.tick)
    }
}

print("Commencing countdown. Press return to abort.")
var countdown = 10
for await event in events {
    switch event {
    case .tick:
        print(countdown)
        countdown -= 1
    case .abort:
        print("Launch aborted!")
        exit(0)
    }
    if countdown == 0 {
        break
    }
}
ticker.cancel()
launch()

func launch() {
    print("Lift off!")
}
```

The `for await` loop handles events from both sources in the order they arrive, which is exactly what a `select` statement in a loop does. Each event source is a separate task (or, for the blocking `readLine`, a separate thread) that yields into the shared continuation, which is safe to use from several producers at once.

The `merge` function from `swift-async-algorithms` does the same thing for asynchronous sequences that already exist, combining them into one sequence of their elements as they arrive: `for await event in merge(ticks, aborts)`. Its `timer(interval:)` function provides a ready-made ticker.

**Exercise 8.10:** Write a version of `countdown` that uses `merge` and `AsyncTimerSequence` from `swift-async-algorithms` instead of a hand-built stream.

**Exercise 8.11:** Following the approach of `mirroredQuery`, implement a variant of `fetch` that requests several URLs concurrently. As soon as the first response arrives, cancel the other requests.

## 8.8. Example: Concurrent Directory Traversal

In this section, we'll build a program that reports the disk usage of one or more directories specified on the command line, like the Unix `du` command. Most of its work is done by the `walkDir` function below, which enumerates the entries of a directory and recursively walks its subdirectories:

```swift
// swiftpl/ch8/du1
import Foundation

struct Usage: Sendable {
    var files = 0
    var bytes = 0

    static func + (a: Usage, b: Usage) -> Usage {
        Usage(files: a.files + b.files, bytes: a.bytes + b.bytes)
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

/// Returns the entries of directory dir.
func entries(of dir: URL) -> [URL] {
    do {
        return try FileManager.default.contentsOfDirectory(
            at: dir, includingPropertiesForKeys: [.isDirectoryKey, .fileSizeKey])
    } catch {
        printError("du: \(error.localizedDescription)")
        return []
    }
}
```

The main program walks each root and prints the totals:

```swift
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

This program pauses for a long while before printing its result:

```
$ swift build -c release
$ time .build/release/du1 ~ /usr /var
213201 files  62.7 GB
real    0m12.1s
```

The program would be nicer if it kept us informed of its progress. However, simply moving the `printDiskUsage` call into the loop would cause it to print thousands of lines of output.

The variant of `du` below prints the totals periodically, but only if the `-v` flag is specified, since not all users will want to see progress messages. The walker runs in its own task, and sends an event for each file it finds into a stream; a ticker task sends a tick every 500 milliseconds; and the main loop merges them, as in Section 8.7:

```swift
// swiftpl/ch8/du2
enum Event: Sendable {
    case file(bytes: Int)
    case tick
    case done
}

let verbose = CommandLine.arguments.contains("-v")
let roots = CommandLine.arguments.dropFirst().filter { $0 != "-v" }.map { URL(filePath: $0) }
let (events, eventsOut) = AsyncStream.makeStream(of: Event.self)

Task.detached {
    for root in roots {
        walkDir(root) { bytes in eventsOut.yield(.file(bytes: bytes)) }
    }
    eventsOut.yield(.done)
}
let ticker = Task {
    while verbose && !Task.isCancelled {
        try? await Task.sleep(for: .milliseconds(500))
        eventsOut.yield(.tick)
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
ticker.cancel()
printDiskUsage(usage)  // final totals
```

Here `walkDir` has been rewritten to call a closure for each file rather than returning totals:

```swift
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

The `break loop` statement uses a *label* to break out of the `for await` loop, not just the `switch`. A plain `break` inside a `switch` case breaks out of the `switch`.

The program now prints a gently scrolling stream of updates:

```
$ .build/release/du2 -v ~ /usr /var
28608 files  8.3 GB
54147 files  10.3 GB
93591 files  15.1 GB
127169 files  52.9 GB
...
213201 files  62.7 GB
```

However, it still takes too long to finish. There's no reason why all the calls to `walkDir` can't be done concurrently, thereby exploiting parallelism in the disk system. The third version of `du` makes the recursive walk concurrent, using a task group at each directory level:

```swift
// swiftpl/ch8/du3
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

Each directory's subdirectories are walked by child tasks, which may in turn create their own children, forming a tree of tasks that mirrors the tree of directories. Each level sums its own files and its children's results.

Since `entries(of:)` performs blocking system calls, this program would ideally limit the number of directory reads in progress at once. In this case, though, the size of the cooperative thread pool provides a natural limit: at most one blocking call per core can be in progress, since there are only that many threads. Although this version may not be the best possible, on a machine with fast storage it runs several times faster than the sequential one.

**Exercise 8.12:** Combine `du2` and `du3`, so that the concurrent walker reports progress.

**Exercise 8.13:** Write a version of `du` that computes and periodically displays separate totals for each of the `root` directories.

## 8.9. Cancellation

Sometimes we need to instruct a task to stop what it is doing, for example, in a web server performing a computation on behalf of a client that has disconnected, or in a search that has already found its answer.

Swift's cancellation is *cooperative*. Calling `cancel()` on a task, or cancelling a task group with `cancelAll()`, doesn't stop anything forcibly; it sets a flag on the task, and on all of its child tasks, recursively. The task observes the flag and decides how to respond. There are several ways to do that:

- `Task.isCancelled` returns `true` if the current task has been cancelled. Long-running loops should check it periodically and wind down.
- `try Task.checkCancellation()` throws a `CancellationError` if the current task has been cancelled, which is convenient in throwing code.
- Many asynchronous library functions check cancellation themselves. `Task.sleep` throws `CancellationError` immediately when the task is cancelled, and `URLSession` abandons the request and throws `URLError(.cancelled)`. An `AsyncStream` iterator returns `nil` when the consuming task is cancelled.
- `withTaskCancellationHandler(operation:onCancel:)` runs a closure immediately when the task is cancelled, which lets you bridge cancellation to an API that has its own way of stopping, like closing a socket.

Why not just kill the task? Because a task killed at an arbitrary point could leave a lock held, a file half-written, or a data structure inconsistent. Cooperative cancellation lets each task clean up and stop at a point where its state is consistent.

Let's make the `du` program cancellable: if the user presses Return, it should stop walking promptly and print the totals so far. We'll run the walk in a task, and on a separate thread wait for input and cancel the task:

```swift
// swiftpl/ch8/du4
let walk = Task {
    var total = Usage()
    for root in roots {
        total = total + (await walkDir(root))
    }
    return total
}

Thread.detachNewThread {
    _ = readLine()  // read a single line
    walk.cancel()
}

let usage = await walk.value
printDiskUsage(usage)
if walk.isCancelled {
    print("(cancelled: totals are incomplete)")
}
```

`walk.value` waits for the task to finish and returns its result.

Now we need the walker to respond to cancellation. Because cancellation propagates through the task tree, every child task in every task group is cancelled too. Each one just needs to check:

```swift
func walkDir(_ dir: URL) async -> Usage {
    if Task.isCancelled {
        return Usage()
    }
    return await withTaskGroup(of: Usage.self) { group in
        // ...as before...
    }
}
```

With this check at the start of each directory, cancellation stops the walk within a directory or so. A task group also helps: when the parent is cancelled, `group.addTask` still adds a child, but the child starts out cancelled, so it returns immediately.

It might be surprising that the program has nothing else to do. Since each `withTaskGroup` waits for its children, when the walk returns, every task it created has stopped. There's no question of tasks continuing to run after `main` has decided to print the result.

There's one more thing we should consider: what if a task wants to perform some cleanup even when it's been cancelled? Since cancellation doesn't interrupt anything, ordinary `defer` blocks and error handling work exactly as usual. A cancelled task that throws `CancellationError` unwinds through `catch` clauses and `defer` statements like any other error.

**Exercise 8.14:** `du4` walks its roots one after another. Write an `asyncReduce` extension on `Sequence` that awaits its closure, then a concurrent variant that walks all the roots at once in a task group. Does cancellation still work?

**Exercise 8.15:** Add cancellation to the web crawler of Section 8.6. When it is cancelled, in-progress fetches should be abandoned. (Hint: `URLSession` already responds to cancellation.)

**Exercise 8.16:** Write a function `search(_ paths: [String], for needle: String)` that searches a set of files concurrently for a string and returns the first file that contains it, cancelling the rest of the search as soon as a match is found.

## 8.10. Example: Chat Server

We'll finish this chapter with a chat server that lets several users broadcast textual messages to each other. There are four kinds of tasks in this program. There is one instance apiece of the main task and a *broadcaster* task, and for each client connection there is one *handler* task and one *client writer* task. The broadcaster is a good illustration of how a single task can own state that others need to change, by receiving requests as events in a stream. The main task's job is to accept incoming network connections from clients. For each one, it creates a new handler task, as we saw in the concurrent echo server at the start of this chapter.

The networking setup is the same as for the echo server of Section 8.3, so we'll show only the interesting parts:

```swift
// swiftpl/ch8/chat
enum ChatEvent: Sendable {
    case entering(id: Int, AsyncStream<String>.Continuation)
    case leaving(id: Int)
    case message(String)
}

let (events, eventsOut) = AsyncStream.makeStream(of: ChatEvent.self)

try await withThrowingDiscardingTaskGroup { group in
    group.addTask { await broadcaster() }
    var nextID = 0
    try await server.executeThenClose { connections in
        for try await connection in connections {
            let id = nextID
            nextID += 1
            group.addTask { try? await handleConn(connection, id: id) }
        }
    }
}
```

Next is the broadcaster. Its local variable `clients` records the current set of connected clients. The only information recorded about each client is the identity of its outgoing message stream, about which more later.

```swift
func broadcaster() async {
    var clients: [Int: AsyncStream<String>.Continuation] = [:]  // all connected clients
    for await event in events {
        switch event {
        case .message(let msg):
            // Broadcast incoming message to all clients' outgoing message streams.
            for client in clients.values {
                client.yield(msg)
            }
        case .entering(let id, let client):
            clients[id] = client
        case .leaving(let id):
            clients.removeValue(forKey: id)?.finish()
        }
    }
}
```

The broadcaster listens for three kinds of events, delivered through the global `events` stream: arrivals of clients, departures of clients, and messages to be broadcast. When it receives an arrival or departure event, it updates the `clients` dictionary, and if the event was a departure, it finishes the client's outgoing message stream. When it receives a message, it broadcasts the message to every connected client.

Because `clients` is a local variable of the broadcaster task, and only the broadcaster can see it, it needs no protection, even though changes to it are requested by many handler tasks. This is the Go proverb "don't communicate by sharing memory; share memory by communicating" in action.

Now let's look at the per-client tasks. The `handleConn` function creates a new outgoing message stream for its client and announces the arrival of this client to the broadcaster over the `events` stream. Then it reads every line of text from the client, sending each line to the broadcaster as a message, prefixed by the identity of its sender. Once there is nothing more to read from the client, `handleConn` announces the departure of the client.

```swift
func handleConn(_ connection: NIOAsyncChannel<ByteBuffer, ByteBuffer>, id: Int) async throws {
    let who = connection.channel.remoteAddress?.description ?? "client \(id)"
    let (outgoing, outgoingOut) = AsyncStream.makeStream(of: String.self)

    try await connection.executeThenClose { inbound, outbound in
        try await withThrowingDiscardingTaskGroup { group in
            // Client writer: copy outgoing messages to the network.
            group.addTask {
                for await msg in outgoing {
                    try await outbound.write(ByteBuffer(string: msg + "\n"))
                }
            }

            outgoingOut.yield("You are \(who)")
            eventsOut.yield(.message("\(who) has arrived"))
            eventsOut.yield(.entering(id: id, outgoingOut))
            defer {
                eventsOut.yield(.leaving(id: id))
                eventsOut.yield(.message("\(who) has left"))
            }

            for try await line in inbound {
                eventsOut.yield(.message("\(who): \(String(buffer: line))"))
            }
        }
    }
}
```

In addition, `handleConn` creates a *client writer* task for each client that receives messages broadcast to the client's outgoing message stream and writes them to the client's network connection. The client writer's loop terminates when the broadcaster finishes the stream after receiving a `leaving` notification. That lets the task group, and so the handler, finish.

The `defer` ensures that the departure is announced however the reading loop ends, whether the client disconnected cleanly or the connection failed with an error. If it failed, the task group cancels the client writer as the error propagates, and since a cancelled task's `for await` over an `AsyncStream` ends, the writer stops too.

The display below shows the server in action with three clients in separate windows on the same computer, using `nc`:

```
$ swift build
$ .build/debug/chat &
$ nc localhost 8000
You are [IPv4]127.0.0.1/127.0.0.1:64208     $ nc localhost 8000
[IPv4]127.0.0.1/127.0.0.1:64211 has arrived You are [IPv4]127.0.0.1/127.0.0.1:64211
Hi!
[IPv4]127.0.0.1/127.0.0.1:64208: Hi!        [IPv4]127.0.0.1/127.0.0.1:64208: Hi!
                                            Hi yourself.
[IPv4]...:64211: Hi yourself.               [IPv4]...:64211: Hi yourself.
^C
                                            [IPv4]...:64208 has left
```

While hosting a chat session for *n* clients, this program runs 2*n* + 2 concurrently communicating tasks, yet it needs no explicit locking operations. The `clients` dictionary is confined to a single task, the broadcaster, so it cannot be accessed concurrently. The only variables that are shared by multiple tasks are streams and their continuations, which are concurrency-safe. We'll say more about confinement, concurrency safety, and the implications of sharing variables among tasks in the next chapter, where we'll also see that Swift has a construct, the *actor*, designed precisely for the role that the broadcaster plays here.

**Exercise 8.17:** Make the broadcaster announce the current set of clients to each new arrival. This requires that the `clients` set and the `entering` and `leaving` events record the client name too.

**Exercise 8.18:** Make the chat server disconnect idle clients, such as those that have sent no messages in the last five minutes. (Hint: use `withTimeout` from Section 8.7 around each read.)

**Exercise 8.19:** Failure of any client program to read data in a timely manner ultimately causes all clients to get stuck in Go's version of this program, because its channels are unbuffered. In ours, the outgoing streams are unbounded, so a slow client instead makes the server's memory grow. Modify the client's outgoing stream to use `.bufferingNewest(10)`, so that a slow client simply misses messages rather than holding up anyone else.

**Exercise 8.20:** Rewrite the broadcaster as an actor (see Section 9.3), with methods `enter`, `leave`, and `broadcast`. Which version do you find clearer?
