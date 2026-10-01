# 4. Composite Types

Chapter 3 covered Swift's basic types, the individual values from which everything else is built. This chapter is about combining them. Swift's standard library provides three general-purpose collections, *arrays*, *dictionaries*, and *sets*, along with array *slices*, which view part of an array; the language provides *structs* for grouping named values into new types, and *tuples* for grouping values without naming the group. At the end of the chapter, we'll use these types together to read structured data from a web service as JSON, and to produce reports from it, as text and as safely escaped HTML.

One property ties the chapter together: all of these types are *values*. Assigning an array, passing a dictionary to a function, or storing a struct in another struct gives the recipient its own independent copy, and changing that copy never affects the original, exactly as with an `Int`. If that sounds expensive, it isn't, because Swift's collections share their storage until the moment one copy is modified. Section 4.2 explains how this *copy-on-write* technique works.

## 4.1. Arrays

An `Array` is an ordered collection of elements, all of one type, that can be accessed by integer position. Swift's arrays grow and shrink as needed, more like Java's `ArrayList`, C++'s `std::vector`, or Go's slices than like the fixed-size arrays of C. The type `Array<Element>` is nearly always written `[Element]`.

An array is usually created with an *array literal*, or with an initializer that repeats a value:

```swift
var queue = ["build", "test", "deploy"]  // [String]
let zeros = [Double](repeating: 0, count: 5)  // [0.0, 0.0, 0.0, 0.0, 0.0]
let empty: [Int] = []
```

Elements are numbered from zero. `count` gives their number and `isEmpty` tests for none. Subscripting reads or writes an element, and every subscript is checked: an index outside `0..<count` traps immediately, rather than reading or writing whatever memory happens to lie beyond the array:

```swift
print(queue[0])  // "build"
queue[2] = "release"
print(queue[3])  // Fatal error: Index out of range
```

When an index might be invalid, check it first, or use one of the many accessors that return an optional instead of trapping, such as `first`, `last`, `firstIndex(of:)`, `randomElement()`, and `popLast()`:

```swift
if let next = queue.first {
    print("next up: \(next)")
}
```

Arrays grow with `append`, `append(contentsOf:)`, `+=`, and `insert(_:at:)`, and shrink with `remove(at:)`, `removeFirst()`, `removeLast()`, and `removeAll()`:

```swift
queue.append("announce")
queue.insert("lint", at: 0)
print(queue)  // "["lint", "build", "test", "release", "announce"]"
queue.removeFirst()
print(queue)  // "["build", "test", "release", "announce"]"
```

Iterating with `for`-`in` visits the elements in order; `enumerated()` supplies their offsets too, and `indices` gives just the valid indices:

```swift
for (i, step) in queue.enumerated() {
    print("\(i + 1). \(step)")
}
```

Two arrays can be compared with `==` when their elements can be, which is to say when the element type conforms to `Equatable`. Arrays are equal if they have the same elements in the same order. There's no `<` for arrays, since there's no single obvious way to order them, but `lexicographicallyPrecedes(_:)` provides the dictionary-style ordering when that's what you want.

Fixed-size sequences of bytes are a common kind of array-like data, and comparing them is a common task. The `swift-crypto` package, which provides Apple's CryptoKit API on every platform, computes cryptographic hashes; a SHA-256 *digest* is 32 bytes that act as a fingerprint of any amount of data. One use is checking that a download arrived intact, by comparing its digest with one published by the source:

```swift
// swiftpl/ch4/verify
import Crypto
import Foundation

let published = "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"

let download = Array("hello".utf8)  // the bytes we received
let digest = SHA256.hash(data: download)
let hex = digest.map { String(format: "%02x", $0) }.joined()
print(hex == published ? "verified" : "corrupted!")  // "verified"

let tampered = SHA256.hash(data: Array("hello!".utf8))
print(digest == tampered)  // "false"
```

A digest behaves like a fixed-length array of bytes: it's a `Sequence` of `UInt8`, so `map` can turn each byte into two hex digits, and it's `Equatable`, so two digests can be compared directly. Changing even one byte of the input changes about half the bits of the digest, which is what makes it a reliable fingerprint.

For the rare cases where you need a genuinely fixed-size array stored inline, without a separate heap allocation (in a tight numeric loop, say, or a struct that mirrors a hardware layout), Swift 6.2 adds `InlineArray`. `InlineArray<4, Float>` is exactly four `Float`s stored in place, like a C array, and can be created from an array literal of the right length. Ordinary code should use `Array`.

**Exercise 4.1:** Write a command, `checksum`, that prints the SHA-256 digest of each file named on its command line. With a `-c` option, it should instead read lines of the form `digest  filename` and report which files fail to match, like `sha256sum -c`.

**Exercise 4.2:** Formatting each byte with `String(format:)` is slow. Write a faster `hex` function that looks up each half-byte in a 16-character table, and measure the difference on a large array of bytes.

## 4.2. Slices and Copy-on-Write

### 4.2.1. Array Slices

Subscripting an array with a range, or calling `prefix`, `suffix`, `dropFirst`, or `dropLast`, produces an `ArraySlice`: a view of a contiguous part of the array that shares its storage:

```swift
let readings = [12.1, 13.4, 15.0, 14.2, 16.8, 18.3, 17.5]  // a week of daily highs
let weekdays = readings[0..<5]
let weekend = readings[5...]
print(weekdays.max()!, weekend.max()!)  // "16.8 18.3"
```

Slicing is cheap, since no elements are copied, and a slice supports everything an array does: iteration, subscripting, `map`, `sorted`, and even mutation. Many functions can therefore accept either, as we'll see when we write generic functions over `Collection` in Chapter 7.

A slice has one property that surprises nearly everyone at first. It keeps the *indices of the array it came from*. The first element of `weekend` isn't `weekend[0]`:

```swift
print(weekend.startIndex, weekend.endIndex)  // "5 7"
print(weekend[5])  // "18.3"
print(weekend[0])  // Fatal error: Index out of bounds
```

The reason is that it makes slices compose cleanly. An index found in a slice is valid in the original array, and an index from the array is valid in any slice that contains it, so algorithms that repeatedly narrow a range, like binary search and parsers, never have to adjust offsets. The habit to form is to use `startIndex`, `endIndex`, `first`, and `last` with any collection, never the literal `0` or `count`. If you need zero-based indices, make an array: `Array(weekend)`.

Because a slice shares storage with its array, it also keeps the whole array alive. Use slices for temporary work and convert to `Array` before storing one for long.

### 4.2.2. Copy-on-Write

Arrays have value semantics, which guarantees the following:

```swift
var original = [1, 2, 3]
var copy = original
copy[0] = 99
print(original)  // "[1, 2, 3]"
```

Copying an array of a million elements on every assignment would make that guarantee ruinously expensive, so Swift doesn't do it. An array value is really a reference to a heap buffer holding the elements, along with their count and the buffer's capacity. `var copy = original` copies only that reference and increments the buffer's reference count, so the two arrays share a buffer. Every *mutating* operation first checks whether the array's buffer is shared. If it's not, the mutation happens in place. If it is, as for `copy[0] = 99` above, the array first makes its own copy of the buffer, and then mutates the copy. Hence the name, *copy-on-write*.

The effect is that passing collections around, returning them from functions, and storing them in other values costs no more than passing a pointer, while code reasons about them as independent values. A copy happens only when it's needed to keep that illusion intact.

The same technique is available for your own types. `isKnownUniquelyReferenced(_:)` tells you whether a class instance has exactly one strong reference, and that's all copy-on-write requires:

```swift
// swiftpl/ch4/cow
final class Buffer {
    var values: [Double]
    init(_ values: [Double]) { self.values = values }
}

/// A Vector is a value type backed by shared, copy-on-write storage.
struct Vector {
    private var buffer: Buffer

    init(_ values: [Double]) {
        buffer = Buffer(values)
    }

    subscript(i: Int) -> Double {
        get { buffer.values[i] }
        set {
            if !isKnownUniquelyReferenced(&buffer) {
                buffer = Buffer(buffer.values)  // copy before writing
            }
            buffer.values[i] = newValue
        }
    }
}
```

This particular example is artificial, since the `[Double]` inside `Buffer` is already copy-on-write. The technique matters when the shared storage is something that isn't a value on its own, such as a manually managed block of memory, an object from a C library, or a large tree of nodes, and you want to give it value semantics.

### 4.2.3. Growing an Array

An array's buffer has a *capacity*: the number of elements it can hold before it must be replaced by a bigger one. When an append finds the buffer full, the array allocates a new buffer about twice as large and moves the elements over. Most appends are therefore cheap, an occasional one is expensive, and the average cost is constant. We can watch the capacity grow:

```swift
var values: [Int] = []
var capacities: [Int] = []
for i in 0..<100 {
    values.append(i)
    if values.capacity != capacities.last {
        capacities.append(values.capacity)
    }
}
print(capacities)
```

On one 64-bit system, this printed:

```
[2, 4, 8, 16, 32, 64, 128]
```

The details of the growth policy are up to the implementation and vary with element size. When you know how many elements are coming, `reserveCapacity(_:)` allocates once up front. Better still, create the array in one step, with `Array(someSequence)` or `map`, which size the buffer correctly from the start.

Since `append` modifies the array variable in place, and copy-on-write hides whether storage is shared, a Swift programmer never has to wonder whether an append produced an array that aliases another. There's no way to observe it.

### 4.2.4. In-Place Algorithms

Many array algorithms rearrange elements where they are, without allocating a second array. A good example is shuffling. The *Fisher–Yates* algorithm walks backwards through the array, swapping each element with one chosen at random from those not yet placed:

```swift
// swiftpl/ch4/shuffle
func shuffle<T>(_ a: inout [T], using rng: inout some RandomNumberGenerator) {
    for i in stride(from: a.count - 1, to: 0, by: -1) {
        let j = Int.random(in: 0...i, using: &rng)
        a.swapAt(i, j)
    }
}

var deck = Array(1...10)
var rng = SystemRandomNumberGenerator()
shuffle(&deck, using: &rng)
print(deck)  // e.g., "[7, 2, 10, 4, 1, 9, 3, 8, 6, 5]"
```

Every ordering is equally likely, and the algorithm takes linear time with no extra memory. The `<T>` makes `shuffle` *generic*, so that it works on arrays of any element type (Section 7.7), and `swapAt` exchanges two elements in place. The standard library's `shuffle()` and `shuffled()` methods do the same job, of course.

The standard library includes other in-place algorithms that are worth knowing. `removeAll(where:)` deletes every element matching a condition, in a single pass with no reallocation. `partition(by:)` rearranges elements so that those matching a condition come last, and returns the index where they begin:

```swift
struct Job {
    var title: String
    var done: Bool
}

var jobs = [
    Job(title: "write tests", done: true),
    Job(title: "fix bug #12", done: false),
    Job(title: "update docs", done: true),
    Job(title: "release 1.2", done: false),
]
let firstDone = jobs.partition(by: { $0.done })
print(jobs[..<firstDone].map(\.title))  // the two unfinished jobs, in some order
jobs.removeAll(where: { $0.done })
print(jobs.count)  // "2"
```

The closures passed to these methods are *predicates*: functions that return `true` for the elements of interest. In a closure, `$0` refers to its first argument, and `\.title` is a *key path* that can stand in for the closure `{ $0.title }` (Section 6.4). Note that `partition` doesn't preserve the original order within each group; when order matters, `filter` builds a new array in order instead.

An array also makes a natural *stack*: `append` pushes, `last` peeks, and `popLast()` pops, returning `nil` when the stack is empty. An undo history is a typical use:

```swift
var document = "Hello"
var history: [String] = []

history.append(document)
document += ", world"
history.append(document)
document += "!"

if let previous = history.popLast() {  // undo
    document = previous
}
print(document)  // "Hello, world"
```

**Exercise 4.3:** Implement your own `partition(_:by:)` for arrays, using `swapAt`, and test it against the standard library's version on random inputs.

**Exercise 4.4:** Write an in-place function that moves every zero in an `[Int]` to the end while keeping the other elements in their original order, in a single pass.

**Exercise 4.5:** Write a function that collapses runs of identical adjacent lines in a `[String]`, in place, replacing each run of two or more with a single line followed by a count, like `"connection reset (x3)"`.

**Exercise 4.6:** Reverse the order of the words in a sentence stored as a `[Character]`, in place, without creating a second array. (Hint: reverse the whole array, then reverse each word.)

**Exercise 4.7:** Instrument the `Vector` type from Section 4.2.2 to count how many times it copies its buffer, and write a program that shows exactly when copies occur.

## 4.3. Dictionaries and Sets

### 4.3.1. Dictionaries

A *dictionary* associates keys with values, and finds, adds, or removes the value for a key in constant time on average, however many entries it holds. It's built on a *hash table*, one of the most broadly useful data structures ever devised. Swift's type is `Dictionary<Key, Value>`, written `[Key: Value]`.

Keys must conform to `Hashable`, meaning that they can be compared with `==` and reduced to a hash value. All the basic types are `Hashable`, as are optionals, arrays, and sets of `Hashable` elements, and any struct or enum whose members are `Hashable` can declare the conformance and get it for free. Values may be of any type.

A dictionary literal lists key-value pairs:

```swift
var stock = [
    "apples": 12,
    "pears": 0,
    "plums": 30,
]
```

An empty dictionary is written `[:]` when its type is known from context, or `[String: Int]()`.

Subscripting a dictionary by key gives an *optional*, because the key might be missing:

```swift
print(stock["plums"] as Any)  // "Optional(30)"
print(stock["kiwis"] as Any)  // "nil"
```

Assigning through a subscript adds or replaces an entry, and assigning `nil` removes one (as does `removeValue(forKey:)`, which also returns the old value):

```swift
stock["kiwis"] = 8
stock["pears"] = nil  // sold out: remove it
```

Often a missing key should simply behave like a default value, and the `default:` form of the subscript says so. It also allows the value to be updated in place:

```swift
stock["apples", default: 0] -= 3  // sold three
stock["figs", default: 0] += 20  // a new delivery
```

Iterating over a dictionary yields `(key, value)` pairs, in an order that is unspecified and varies between runs. Swift seeds its hash function randomly in each process, both to keep programs from depending on an accidental order and to defeat attacks that feed a server keys chosen to collide. When order matters, sort:

```swift
for (fruit, count) in stock.sorted(by: { $0.key < $1.key }) {
    print("\(fruit)\t\(count)")
}
```

Dictionaries are values like arrays, so a `let` dictionary is immutable, and they can be compared with `==` when their values are `Equatable`. A few more operations come up often. `Dictionary(grouping:by:)` sorts a sequence into buckets by a computed key; `mapValues` transforms every value; and `merge(_:uniquingKeysWith:)` combines two dictionaries, resolving clashes with a closure:

```swift
let words = ["apple", "avocado", "banana", "blueberry", "cherry"]
let byInitial = Dictionary(grouping: words, by: { $0.first! })
// ["a": ["apple", "avocado"], "b": ["banana", "blueberry"], "c": ["cherry"]]

var morning = ["apples": 5, "plums": 2]
morning.merge(["plums": 3, "figs": 1], uniquingKeysWith: +)
// ["apples": 5, "plums": 5, "figs": 1]
```

### 4.3.2. Sets

A `Set` is an unordered collection of distinct `Hashable` elements, with constant-time membership testing and the operations of set algebra: `union`, `intersection`, `subtracting`, `symmetricDifference`, `isSubset(of:)`, `isDisjoint(with:)`, and in-place forms of each. A set can be written with an array literal when its type is known:

```swift
let swiftTopics: Set = ["concurrency", "generics", "macros", "testing"]
let goTopics: Set = ["concurrency", "generics", "testing", "tooling"]

print(swiftTopics.intersection(goTopics).sorted())  // "["concurrency", "generics", "testing"]"
print(swiftTopics.subtracting(goTopics))  // "["macros"]"
```

The `insert` method returns a pair whose `inserted` component tells whether the element was new, which makes "have I seen this before?" a single operation. This program prints each distinct visitor in an access log, one name per line, the first time it appears:

```swift
// swiftpl/ch4/visitors
var seen = Set<String>()
while let line = readLine() {
    guard let name = line.split(separator: " ").first else { continue }
    if seen.insert(String(name)).inserted {
        print(name)
    }
}
```

### 4.3.3. Composite Keys

Keys needn't be simple. A struct whose properties are all `Hashable` can be a key, if it declares the conformance, and so can an array of `Hashable` elements:

```swift
struct GridPoint: Hashable {
    var row, column: Int
}

var occupied = Set<GridPoint>()
occupied.insert(GridPoint(row: 3, column: 4))

var routeCounts: [[String]: Int] = [:]
routeCounts[["home", "office"], default: 0] += 1
```

The compiler writes `==` and `hash(into:)` for such structs by combining those of their properties. You can write them yourself when the default doesn't fit, for example to compare names without regard to case; the one rule is that values that are equal must produce equal hashes.

Tuples, unfortunately, can't be `Hashable`, so a small struct is the usual way to make a key out of several values.

### 4.3.4. Example: Finding Anagrams

Two words are *anagrams* if they contain the same letters in a different order, like "listen" and "silent." Sorting a word's letters gives a *signature* that all its anagrams share, which makes a dictionary from signatures to words a natural way to group them:

```swift
// swiftpl/ch4/anagrams
// Anagrams reads text and prints each group of words that are anagrams of one another.
var groups: [String: Set<String>] = [:]
while let line = readLine() {
    for word in line.split(whereSeparator: { !$0.isLetter }) {
        let w = word.lowercased()
        groups[String(w.sorted()), default: []].insert(w)
    }
}

for (_, words) in groups.sorted(by: { $0.key < $1.key }) where words.count > 1 {
    print(words.sorted().joined(separator: " "))
}
```

```
$ echo "Listen, silent night: enlist the tinsel, google inlets" | swift run anagrams
enlist inlets listen silent tinsel
```

`split(whereSeparator:)` breaks each line into words at every character that isn't a letter, and `sorted()` on a `String` returns its characters as a sorted array, which `String(_:)` turns back into a string to serve as the key. Each value is a `Set`, so that a word repeated in the input appears only once in its group, and the `default:` subscript creates an empty set the first time a signature is seen, then inserts into it in place. The output loop sorts the groups by signature, so the output is the same on every run, and the `where` clause skips words that have no anagram partners.

**Exercise 4.8:** Real anagram finders ignore accents, so that "résumé" and "sumere" might count. Use `folding(options:locale:)` from Foundation to normalize words before computing their signatures.

**Exercise 4.9:** Write `wordcount`, which reports the ten most frequent words in its input along with their counts, ignoring case.

**Exercise 4.10:** Modify `anagrams` to print, for each group, how many times each of its words appeared in the input.

### 4.3.5. Nested Collections

A dictionary's values can be collections themselves. A *concordance* records, for each word in a text, the lines on which it appears, which is a natural `[String: [Int]]`:

```swift
// swiftpl/ch4/concordance
var index: [String: [Int]] = [:]
var lineNumber = 0
while let line = readLine() {
    lineNumber += 1
    let words = Set(line.lowercased().split(whereSeparator: { !$0.isLetter }))
    for word in words {
        index[String(word), default: []].append(lineNumber)
    }
}

for word in index.keys.sorted() {
    let lines = index[word]!.map(String.init).joined(separator: ", ")
    print("\(word): \(lines)")
}
```

The `default:` subscript creates an empty array for a word the first time it's seen, and `append` then extends that array in place, inside the dictionary, without copying it. Putting each line's words in a `Set` first ensures that a line number is recorded only once, even if a word appears twice on the line.

The force-unwrap `index[word]!` in the output loop is safe because `word` came from the dictionary's own keys. In general, though, a lookup can fail, and *optional chaining* often expresses the intent better: `index["swift"]?.count` is the number of lines mentioning "swift," or `nil` if there are none, and `??` supplies a fallback:

```swift
print(index["swift"]?.count ?? 0)
```

**Exercise 4.11:** Extend the concordance to ignore a list of common words ("the," "and," "of," and so on) read from a file, and to print words in order of how many lines they appear on.

## 4.4. Structs and Tuples

A *struct* groups named values, called *stored properties*, into a single new type. A library catalog, for example, might describe each book with a struct:

```swift
import Foundation

struct Book {
    var title: String
    var author: String
    var isbn: String
    var pages: Int
    var published: Date
    var onLoan: Bool
}

var book = Book(
    title: "The Lighthouse Keeper's Almanac", author: "Mara Quill",
    isbn: "978-0-00-000001-7", pages: 312, published: Date(timeIntervalSince1970: 1_525_132_800),
    onLoan: false)
```

Properties are accessed with dot notation, and if the struct is in a `var`, they can be assigned:

```swift
book.onLoan = true
print(book.title, book.onLoan)
```

A struct is mutable or immutable as a whole. If `book` were a `let`, no property of it could be changed, whether the property is declared with `var` or not. (A property declared with `let` can't be changed even in a `var` struct.) That's a direct consequence of value semantics: a struct value *is* its properties, so changing a property changes the value.

The initializer we called, `Book(title:author:isbn:pages:published:onLoan:)`, is the *memberwise initializer*, which the compiler writes for every struct, with a parameter for each stored property in declaration order; properties with default values get parameters with default arguments. If you write an initializer of your own inside the struct's declaration, the memberwise one is no longer generated; to keep both, put yours in an extension.

Because structs are values, updating one inside a collection means updating it *in place*, through the collection, rather than updating a copy:

```swift
/// Marks the book with the given ISBN as on loan, reporting whether it was available.
func checkOut(isbn: String, from catalog: inout [Book]) -> Bool {
    guard let i = catalog.firstIndex(where: { $0.isbn == isbn }), !catalog[i].onLoan else {
        return false
    }
    catalog[i].onLoan = true
    return true
}
```

A version that wrote `var b = catalog[i]; b.onLoan = true` would change a copy and leave the catalog untouched, a common early mistake. The habit to form is to modify elements through the collection's subscript, and to pass the collection `inout` when a function needs to modify it.

Two struct declarations with identical properties are still different types; each declaration introduces a new type with its own name.

### 4.4.1. Recursive Types

A struct can't contain a stored property of its own type, since the value would contain itself and be infinitely large. Recursive data, such as trees, lists, and nested documents, needs a level of indirection, which can come from a class, an array, or an *indirect enum*.

A file-system tree is a good example. A node is either a file, with a size, or a directory, containing other nodes:

```swift
// swiftpl/ch4/filetree
enum FileNode {
    case file(name: String, size: Int)
    case directory(name: String, children: [FileNode])

    var totalSize: Int {
        switch self {
        case .file(_, let size):
            size
        case .directory(_, let children):
            children.reduce(0) { $0 + $1.totalSize }
        }
    }

    func printTree(indent: String = "") {
        switch self {
        case .file(let name, let size):
            print("\(indent)\(name) (\(size) bytes)")
        case .directory(let name, let children):
            print("\(indent)\(name)/ (\(totalSize) bytes)")
            for child in children {
                child.printTree(indent: indent + "  ")
            }
        }
    }
}

let project = FileNode.directory(name: "app", children: [
    .file(name: "Package.swift", size: 512),
    .directory(name: "Sources", children: [
        .file(name: "main.swift", size: 2048),
        .file(name: "Model.swift", size: 4096),
    ]),
    .file(name: "README.md", size: 1024),
])
project.printTree()
```

```
app/ (7680 bytes)
  Package.swift (512 bytes)
  Sources/ (6144 bytes)
    main.swift (2048 bytes)
    Model.swift (4096 bytes)
  README.md (1024 bytes)
```

Both `totalSize` and `printTree` follow the shape of the data: a `switch` handles each kind of node, and the directory case recurses into the children. Here, the array of children provides the indirection, since an array's elements are stored in a separate heap buffer. When a case holds a value of the enum's own type *directly*, as in a linked list, mark the enum `indirect`, and Swift will store that case's payload on the heap:

```swift
indirect enum List<Element> {
    case end
    case node(Element, List<Element>)
}

let numbers = List.node(1, .node(2, .node(3, .end)))
```

A struct with no stored properties at all, `struct Empty {}`, is legal, occupies no space, and carries no information. Swift's usual "nothing" value, though, is the empty tuple `()`, also spelled `Void`, which is what functions without a result return.

### 4.4.2. Comparing Structs

A struct whose stored properties are all `Equatable` can declare itself `Equatable`, and the compiler writes `==`, comparing the properties one by one:

```swift
struct GridPoint: Equatable {
    var row, column: Int
}

let a = GridPoint(row: 1, column: 2)
let b = GridPoint(row: 2, column: 1)
print(a == b)  // "false"
```

`Hashable` and `Codable` (Section 4.5) are synthesized the same way. `Comparable` is synthesized only for enums whose cases have no associated values, which are ordered by declaration; for a struct, there's no single natural order of its properties, so you write `<` yourself.

### 4.4.3. Composition and Forwarding

Larger types are built by nesting smaller ones as properties:

```swift
struct Address {
    var street: String
    var city: String
    var postalCode: String
}

struct Customer {
    var name: String
    var address: Address
}

struct Order {
    var id: Int
    var customer: Customer
    var total: Double
}

var order = Order(
    id: 1042,
    customer: Customer(
        name: "Ada",
        address: Address(street: "12 Analytical Way", city: "London", postalCode: "N1 9GU")),
    total: 59.90)
order.customer.address.city = "Cambridge"
```

Assignments through a chain of properties like `order.customer.address.city` modify the nested value in place. Composition is explicit: an `Order` *has* a `Customer`, and you reach the customer's properties through `order.customer`.

When a particular nested property is used often, a *computed property* can provide a shortcut. It looks like a stored property to users but runs code to get and set its value:

```swift
extension Order {
    var city: String {
        get { customer.address.city }
        set { customer.address.city = newValue }
    }
}
```

To forward *all* of a nested value's properties, rather than writing a computed property for each, Swift provides *dynamic member lookup*. A type marked `@dynamicMemberLookup` provides a subscript that receives a *key path* (a typed reference to a property, written like `\Customer.name`), and member accesses that the type doesn't recognize are routed through it:

```swift
@dynamicMemberLookup
struct Order {
    var id: Int
    var customer: Customer
    var total: Double

    subscript<T>(dynamicMember keyPath: WritableKeyPath<Customer, T>) -> T {
        get { customer[keyPath: keyPath] }
        set { customer[keyPath: keyPath] = newValue }
    }
}

print(order.name)  // "Ada", meaning order.customer.name
order.address.postalCode = "CB2 1TN"  // means order.customer.address.postalCode
```

Despite the word "dynamic," this is checked entirely at compile time: `order.name` compiles only because `\Customer.name` is a valid key path, and its type is known. Section 6.4 returns to key paths, and Section 6.3 to sharing *behavior* between types, which Swift does with protocols.

### 4.4.4. Tuples

A *tuple* groups values without declaring a type for them. Elements are accessed by position, `.0`, `.1`, and so on, or by name, if the tuple has labels:

```swift
let status = (404, "Not Found")
print(status.0, status.1)  // "404 Not Found"

let range = (low: 9.8, high: 18.4)
print(range.high - range.low)

let (low, high) = range  // destructuring
```

Tuples are ideal for returning several values from a function (Section 5.3) and for short-lived groupings inside one. Tuples of up to six elements can be compared with `==` and `<`, element by element. But tuples can't have methods or conform to protocols, so once a group of values deserves a name, or appears in more than one place, make it a struct.

**Exercise 4.12:** Write a function that, given a `FileNode`, returns the paths of the `n` largest files in the tree, such as `app/Sources/Model.swift`.

## 4.5. JSON

*JSON*, JavaScript Object Notation, is the most common format for exchanging structured data between programs, especially over the web. Its popularity comes from its simplicity: a JSON value is a number, a string, `true`, `false`, `null`, an *array* of values in square brackets, or an *object*, an unordered set of name-value pairs in braces. That's the whole language:

```
number     2.5
string     "Grüße, \"world\""
array      ["tea", "coffee", "cocoa"]
object     {"title": "Dune", "year": 1965, "series": true}
```

These map naturally onto Swift: arrays onto arrays, objects onto structs or dictionaries, and the atoms onto numbers, strings, Booleans, and optionals.

Swift's support for JSON, and for other formats, is built on two protocols, `Encodable` and `Decodable` (together, `Codable`). Foundation's `JSONEncoder` and `JSONDecoder` convert between `Codable` values and JSON text, and the same types work with `PropertyListEncoder` and with third-party encoders for YAML, CBOR, MessagePack, and others.

Here's a small catalog of books:

```swift
// swiftpl/ch4/catalog
import Foundation

struct CatalogEntry: Codable {
    var title: String
    var authors: [String]
    var year: Int
    var copies: Int = 1

    enum CodingKeys: String, CodingKey {
        case title
        case authors
        case year = "published"
        case copies = "copies_owned"
    }
}

let catalog = [
    CatalogEntry(title: "The Mythical Man-Month", authors: ["Frederick P. Brooks Jr."], year: 1975,
                 copies: 2),
    CatalogEntry(title: "Structure and Interpretation of Computer Programs",
                 authors: ["Harold Abelson", "Gerald Jay Sussman"], year: 1985),
    CatalogEntry(title: "Hacker's Delight", authors: ["Henry S. Warren Jr."], year: 2002,
                 copies: 3),
]
```

Converting Swift values to JSON is *encoding*:

```swift
let encoder = JSONEncoder()
encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
let data = try encoder.encode(catalog)
print(String(decoding: data, as: UTF8.self))
```

```
[
  {
    "authors" : [
      "Frederick P. Brooks Jr."
    ],
    "copies_owned" : 2,
    "published" : 1975,
    "title" : "The Mythical Man-Month"
  },
  {
    "authors" : [
      "Harold Abelson",
      "Gerald Jay Sussman"
    ],
    ...
  },
  ...
]
```

Without `outputFormatting`, the encoder produces the most compact form, all on one line. `.prettyPrinted` adds line breaks and indentation for human readers, and `.sortedKeys` orders each object's keys, so that the output is the same on every run, which helps when JSON is stored in version control or compared in tests.

How does `JSONEncoder` know what fields `CatalogEntry` has? It doesn't need to. When a struct declares `Codable` conformance and all its stored properties are `Codable` too, the *compiler* writes the code: an `encode(to:)` method that writes each property, and an `init(from:)` initializer that reads them back. By default, each property's name is used as its JSON key. To use different names, declare a nested enum called `CodingKeys` whose raw values are the JSON names, as above: `year` is written as `published`, and `copies` as `copies_owned`.

Optional properties get special treatment. Encoding leaves out a property whose value is `nil`, and decoding treats a missing key as `nil`. "This field may be absent" is thus expressed by the property's type.

*Decoding* goes the other way:

```swift
let decoded = try JSONDecoder().decode([CatalogEntry].self, from: data)
print(decoded.map(\.title))
```

The first argument says what to decode, written as a *metatype*: `[CatalogEntry].self` is the type `[CatalogEntry]` used as a value. Decoding checks everything. A missing key, a value of the wrong type, or malformed JSON makes `decode` throw a `DecodingError` that pinpoints the problem, down to the path of the offending field. Keys in the input that the type doesn't declare are ignored, which means you can declare a type with only the fields you care about and decode just those from a larger document:

```swift
struct TitleOnly: Decodable {
    var title: String
}

let titles = try JSONDecoder().decode([TitleOnly].self, from: data)
```

### 4.5.1. Example: A Weather Forecast

Many web services speak JSON over HTTP. To illustrate, we'll write a command-line weather forecast using Open-Meteo (`open-meteo.com`), a free forecast service that needs no account or API key. A request names a location by latitude and longitude, and the daily values wanted:

```
https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41
    &daily=temperature_2m_max,temperature_2m_min,precipitation_sum&timezone=auto
```

The response (abridged) looks like this:

```
{
  "latitude": 52.52,
  "longitude": 13.419998,
  "timezone": "Europe/Berlin",
  "daily_units": { "temperature_2m_max": "°C", ... },
  "daily": {
    "time": ["2025-09-30", "2025-10-01", ...],
    "temperature_2m_max": [18.4, 16.9, ...],
    "temperature_2m_min": [9.8, 8.1, ...],
    "precipitation_sum": [0.0, 2.3, ...]
  }
}
```

We'll put the client code in a library module, `Weather`, so that the examples in the next section can share it. First, types to match the response:

```swift
// swiftpl/ch4/weather/Sources/Weather/Weather.swift
import Foundation

public struct Forecast: Decodable, Sendable {
    public var latitude: Double
    public var longitude: Double
    public var timezone: String
    public var daily: Daily
}

public struct Daily: Decodable, Sendable {
    public var time: [String]  // ISO dates, like "2025-09-30"
    public var temperature2mMax: [Double?]
    public var temperature2mMin: [Double?]
    public var precipitationSum: [Double?]
}
```

The service uses `snake_case` keys, while Swift uses `camelCase` names. Rather than writing `CodingKeys` for every property, we'll set the decoder's `keyDecodingStrategy` to `.convertFromSnakeCase`, which converts `temperature_2m_max` to `temperature2mMax` and `precipitation_sum` to `precipitationSum`. The `daily_units` object isn't declared, so it's skipped. The arrays of measurements have optional elements because the service reports `null` for any value it can't provide.

The function that performs the request builds its URL with `URLComponents`, which takes care of escaping query parameters correctly, and checks the HTTP status before decoding:

```swift
// swiftpl/ch4/weather/Sources/Weather/Fetch.swift
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

public enum WeatherError: Error {
    case badStatus(Int)
}

/// Fetches a daily forecast for the given location.
public func fetchForecast(latitude: Double, longitude: Double) async throws -> Forecast {
    var components = URLComponents(string: "https://api.open-meteo.com/v1/forecast")!
    components.queryItems = [
        URLQueryItem(name: "latitude", value: String(latitude)),
        URLQueryItem(name: "longitude", value: String(longitude)),
        URLQueryItem(name: "daily", value: "temperature_2m_max,temperature_2m_min,precipitation_sum"),
        URLQueryItem(name: "timezone", value: "auto"),
    ]
    let (data, response) = try await URLSession.shared.data(from: components.url!)
    if let http = response as? HTTPURLResponse, http.statusCode != 200 {
        throw WeatherError.badStatus(http.statusCode)
    }
    let decoder = JSONDecoder()
    decoder.keyDecodingStrategy = .convertFromSnakeCase
    return try decoder.decode(Forecast.self, from: data)
}
```

The command itself takes coordinates as arguments and prints a table:

```swift
// swiftpl/ch4/weather/Sources/forecast/main.swift
// Forecast prints the daily weather forecast for a latitude and longitude.
import Foundation
import Weather

let args = CommandLine.arguments
guard args.count == 3, let lat = Double(args[1]), let lon = Double(args[2]) else {
    print("usage: forecast LATITUDE LONGITUDE")
    exit(2)
}

let f = try await fetchForecast(latitude: lat, longitude: lon)
print("Forecast for \(f.latitude), \(f.longitude) (\(f.timezone))")
print("date          low    high   rain")
for (i, day) in f.daily.time.enumerated() {
    let values = String(format: "  %5.1f  %5.1f  %5.1f",
                        f.daily.temperature2mMin[i] ?? .nan,
                        f.daily.temperature2mMax[i] ?? .nan,
                        f.daily.precipitationSum[i] ?? .nan)
    print(day + values)
}
```

```
$ swift run forecast 52.52 13.41
Forecast for 52.52, 13.419998 (Europe/Berlin)
date          low    high   rain
2025-09-30    9.8   18.4    0.0
2025-10-01    8.1   16.9    2.3
...
```

(Forecasts change, of course; the figures are illustrative.) Missing values print as `nan`; the next section shows a better way.

**Exercise 4.13:** Add a `--days N` option to `forecast` (the API's `forecast_days` parameter), and print the week's average high and total rainfall at the end.

**Exercise 4.14:** Cache each response in a file named after the coordinates, and reuse it if it's less than an hour old, so that repeated runs don't hit the network.

**Exercise 4.15:** Open-Meteo also offers a geocoding API that turns a place name into coordinates (`https://geocoding-api.open-meteo.com/v1/search?name=Berlin`). Let `forecast` accept a place name instead of coordinates.

## 4.6. Text Templates with String Interpolation

String interpolation took care of all our formatting so far. For more elaborate output, many languages offer template engines: separate mini-languages, interpreted at run time, with placeholders for data. Swift takes a different route. Its string interpolation is *extensible*, so a "template" can be ordinary Swift code with custom interpolations, checked by the compiler like the rest of the program.

### 4.6.1. Custom Interpolations

When Swift compiles a string literal containing interpolations, it builds the string by calling methods on an *interpolation* value: `appendLiteral(_:)` for each piece of literal text, and `appendInterpolation(...)` for each `\(...)`, passing whatever is inside the parentheses as arguments. For ordinary strings, the interpolation type is `DefaultStringInterpolation`, and because `appendInterpolation` is just a method, we can add overloads to it in an extension, including ones with argument labels.

Here are two interpolations for the forecast: one that formats a temperature, or a dash if it's missing, and one that draws a horizontal bar whose length represents a number:

```swift
// swiftpl/ch4/weather/Sources/forecast2/Interpolations.swift
import Foundation

extension DefaultStringInterpolation {
    /// Interpolates an optional temperature with one decimal place, right-aligned.
    mutating func appendInterpolation(temp value: Double?) {
        appendLiteral(value.map { String(format: "%5.1f", $0) } ?? "    –")
    }

    /// Interpolates a bar of block characters whose length is value × scale.
    mutating func appendInterpolation(bar value: Double, scale: Double = 1) {
        let length = max(0, Int((value * scale).rounded()))
        appendLiteral(String(repeating: "█", count: length))
    }
}
```

With these, the report reads almost like a template, but every value has its real type:

```swift
// swiftpl/ch4/weather/Sources/forecast2/main.swift
import Weather

func report(_ f: Forecast) -> String {
    var out = "Forecast for \(f.latitude), \(f.longitude) (\(f.timezone))\n"
    for (i, day) in f.daily.time.enumerated() {
        let low = f.daily.temperature2mMin[i]
        let high = f.daily.temperature2mMax[i]
        out += "\(day) \(temp: low) \(temp: high)  \(bar: high ?? 0, scale: 0.5)\n"
    }
    return out
}

let f = try await fetchForecast(latitude: 52.52, longitude: 13.41)
print(report(f))
```

```
Forecast for 52.52, 13.419998 (Europe/Berlin)
2025-09-30   9.8  18.4  █████████
2025-10-01   8.1  16.9  ████████
...
```

Compared with a separate template language, this approach catches mistakes at compile time, whether a misspelled property or a `Double?` passed where a `String` was expected. It gives templates the full power of Swift, with loops, conditionals, and function calls, and adds no new syntax to learn. The one thing it gives up is changing templates without recompiling, which programs rarely need.

### 4.6.2. HTML with Automatic Escaping

Generating HTML raises a danger that plain text doesn't. Text from outside the program, such as a user's input, a database field, or a response from a web service, might contain `<`, `&`, or quotation marks. Insert it into a page unescaped, and a stray character breaks the layout; worse, a malicious one injects markup or script into the page. *Cross-site scripting*, as this attack is known, has been one of the most common security vulnerabilities on the web for decades.

The reliable defense is to escape *everything* that comes from outside, automatically, and to require an explicit decision to insert raw markup. We can make the type system enforce that with a type, `HTML`, for markup that's already safe. String literals in our own source code are trusted. Interpolated values are escaped, unless they're themselves `HTML`:

```swift
// swiftpl/ch4/weather/Sources/forecastpage/HTML.swift
/// HTML is a fragment of trusted HTML markup.
struct HTML: ExpressibleByStringInterpolation, CustomStringConvertible {
    let description: String

    init(stringLiteral value: String) {
        description = value  // literals in the source are trusted
    }

    init(stringInterpolation: Interpolation) {
        description = stringInterpolation.output
    }

    struct Interpolation: StringInterpolationProtocol {
        var output = ""

        init(literalCapacity: Int, interpolationCount: Int) {
            output.reserveCapacity(literalCapacity * 2)
        }

        mutating func appendLiteral(_ literal: String) {
            output += literal  // trusted
        }

        mutating func appendInterpolation(_ value: some CustomStringConvertible) {
            output += escape(value.description)  // untrusted: escape it
        }

        mutating func appendInterpolation(_ html: HTML) {
            output += html.description  // already HTML: insert as is
        }
    }
}

func escape(_ s: String) -> String {
    var out = ""
    for c in s {
        switch c {
        case "&": out += "&amp;"
        case "<": out += "&lt;"
        case ">": out += "&gt;"
        case "\"": out += "&quot;"
        case "'": out += "&#39;"
        default: out.append(c)
        }
    }
    return out
}
```

A type conforms to `ExpressibleByStringInterpolation` by declaring an `Interpolation` type and an initializer that takes a finished one. Literal text from the program's source is appended as is. Any interpolated value that's `CustomStringConvertible` is converted to text and escaped. An interpolated `HTML` value matches the more specific overload, and is inserted unescaped, since it's markup we've already vouched for. That's what allows HTML fragments to be built separately and assembled without double escaping.

Here's a forecast page, which includes a place name given by the user:

```swift
func forecastPage(place: String, _ f: Forecast) -> HTML {
    var rows = ""
    for (i, day) in f.daily.time.enumerated() {
        let low = f.daily.temperature2mMin[i].map { String($0) } ?? "–"
        let high = f.daily.temperature2mMax[i].map { String($0) } ?? "–"
        let row: HTML = "<tr><td>\(day)</td><td>\(low)</td><td>\(high)</td></tr>\n"
        rows += row.description
    }
    return """
        <h1>Forecast for \(place)</h1>
        <table>
        <tr><th>Date</th><th>Low</th><th>High</th></tr>
        \(HTML(stringLiteral: rows))
        </table>
        """
}
```

Suppose a mischievous user asks for the forecast for `<script>alert('hi')</script>`. The page shows that text, harmlessly, as text:

```html
<h1>Forecast for &lt;script&gt;alert(&#39;hi&#39;)&lt;/script&gt;</h1>
```

The overload is chosen by each value's *static* type, so no value of type `String` can reach the page unescaped, no matter how a future change alters the code. The only way to insert raw text is to construct an `HTML` value from it explicitly, as `forecastPage` does once, for rows it built itself as `HTML`. Such constructions are easy to search for and review. (Exercise 4.17 removes even that one.) The same idea, using distinct types for trusted and untrusted text, protects SQL queries, shell commands, and file paths.

For full web applications, packages such as Elementary and Plot provide complete type-safe HTML DSLs, and Leaf provides a traditional template engine.

**Exercise 4.16:** Serve `forecastPage` from a Hummingbird server (Section 1.7), taking the place name and coordinates from query parameters, as in `/forecast?place=Berlin&lat=52.52&lon=13.41`.

**Exercise 4.17:** Add a static method `HTML.joined(_ fragments: [HTML]) -> HTML`, and use it to remove the `HTML(stringLiteral:)` call from `forecastPage` by keeping the rows as `[HTML]`. Could `init(stringLiteral:)` be restricted so that it can't be called with a variable at all?
