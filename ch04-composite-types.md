# 4. Composite Types

In Chapter 3 we discussed the basic types that serve as building blocks for data structures in a Swift program; they are the atoms of our universe. In this chapter, we'll take a look at *composite* types, the molecules created by combining the basic types in various ways. We'll talk about five such types: arrays, slices, dictionaries, sets, and structs, along with their anonymous cousin, the tuple. At the end of this chapter, we'll show how structured data using these types can be encoded as and decoded from JSON data and used to generate HTML from templates.

All of the types in this chapter are *value types*. An array, a dictionary, or a struct behaves like an integer: assigning it to another variable or passing it to a function gives the recipient an independent value, and modifying one copy never affects another. Swift implements the collections with *copy-on-write* storage, so this independence doesn't cost a copy unless one is truly needed. This combination of value semantics with efficient sharing is one of the most distinctive things about Swift, and we'll look at how it works in Section 4.2.

## 4.1. Arrays

An array is an ordered, random-access collection of elements of a single type. Swift's `Array` is dynamically sized: it can grow and shrink as elements are added and removed. (In this respect it's closer to Go's slices, C++'s `std::vector`, or Java's `ArrayList` than to Go's fixed-length arrays.) The type `Array<Element>` is almost always written with the shorthand `[Element]`.

Individual array elements are accessed with the conventional subscript notation, where subscripts run from zero to one less than the array's count. The `count` property returns the number of elements, and `isEmpty` tests for zero elements.

```swift
var a = [Int](repeating: 0, count: 3)  // array of 3 integers, all 0
print(a[0])  // print the first element
print(a[a.count - 1])  // print the last element, a[2]

// Print the indices and elements.
for (i, v) in a.enumerated() {
    print(i, v)
}

// Print the elements only.
for v in a {
    print(v)
}
```

Swift has no zero values, so an array is created either from an *array literal*, a list of values in brackets, or with an initializer such as `init(repeating:count:)`:

```swift
let q = [1, 2, 3]
let r: [Double] = [1, 2, 3]  // the literals become Doubles
let empty: [String] = []
```

Every subscript operation is checked. Accessing an element outside the range `0..<count` is a run-time error that traps the program immediately, rather than reading or writing some other memory:

```swift
let r = [1, 2, 3]
print(r[3])  // Fatal error: Index out of range
```

When an index might be out of range, check first, or use the safe accessors that return optionals: `first`, `last`, `randomElement()`, `firstIndex(of:)`, and so on.

```swift
if let last = r.last {
    print(last)  // "3"
}
```

Arrays grow with `append(_:)` and `append(contentsOf:)`, or the `+=` operator; they shrink with `removeLast()`, `removeFirst()`, `remove(at:)`, or `removeAll()`; and `insert(_:at:)` inserts an element anywhere:

```swift
var names = ["Bob"]
names.append("Carol")
names += ["Dave", "Eve"]
names.insert("Alice", at: 0)
print(names)  // "["Alice", "Bob", "Carol", "Dave", "Eve"]"
names.remove(at: 2)
print(names)  // "["Alice", "Bob", "Dave", "Eve"]"
```

If an array's element type conforms to `Equatable`, then the array does too, so we may compare two arrays directly using the `==` operator, which reports whether they have the same elements in the same order:

```swift
let a = [1, 2]
let b = [1, 2]
let c = [1, 3]
print(a == b, a == c, b == c)  // "true false false"
```

Arrays aren't `Comparable`, since there's more than one reasonable way to order them, but they have a method `lexicographicallyPrecedes(_:)` for the most common ordering.

As a more plausible example of comparison, the `SHA256` type from the `swift-crypto` package (which provides the same API as Apple's CryptoKit framework on all platforms) produces the SHA256 cryptographic hash or *digest* of a message. A SHA256 digest has 256 bits, which the library represents as a value of type `SHA256.Digest`. Digests are `Equatable` and are sequences of `UInt8`, so they can be compared, iterated, and converted to arrays.

```swift
// swiftpl/ch4/sha256
import Crypto

let c1 = SHA256.hash(data: Array("x".utf8))
let c2 = SHA256.hash(data: Array("X".utf8))
print(hex(c1))
print(hex(c2))
print(c1 == c2)

func hex(_ digest: some Sequence<UInt8>) -> String {
    digest.map { String($0, radix: 16).leftPadded(to: 2) }.joined()
}

extension String {
    func leftPadded(to width: Int) -> String {
        String(repeating: "0", count: max(0, width - count)) + self
    }
}
// Output:
// 2d711642b726b04401627ca9fbac32f5c8530fb1903cc4db02258717921a4881
// 4b68ab3847feda7d6c62c1fbcbeebfa35eab7351ed5e78f4ddadea5df64b8015
// false
```

The two inputs differ by only a single bit, but approximately half the bits are different in the digests. The parameter type `some Sequence<UInt8>` means "some type that is a sequence of bytes," which accepts a digest, an array, or a string's `utf8` view. We'll explain this notation in Chapter 7.

If you need a truly fixed-size array of elements stored inline, without a separate heap allocation, Swift 6.2 adds `InlineArray`. The type `InlineArray<4, Int>` holds exactly four `Int`s directly, the way a C array or Go array does, and can be initialized from an array literal of the right length. It's a specialized tool for performance-sensitive code; ordinary programs should use `Array`.

**Exercise 4.1:** Write a function that counts the number of bits that are different in two SHA256 digests.

**Exercise 4.2:** Write a program that prints the SHA256 hash of its standard input by default but supports a command-line flag to print the SHA384 or SHA512 hash instead.

## 4.2. Slices and Copy-on-Write

### 4.2.1. Array Slices

An `ArraySlice` is a view of a contiguous subsequence of an array. Subscripting an array with a range, or calling a method like `prefix`, `suffix`, `dropFirst`, or `dropLast`, produces a slice:

```swift
let months = ["January", "February", "March", "April", "May", "June",
              "July", "August", "September", "October", "November", "December"]
let q2 = months[3..<6]
let summer = months[5..<8]
print(q2)  // "["April", "May", "June"]"
print(summer)  // "["June", "July", "August"]"
```

Making a slice is cheap: it doesn't copy the elements, but shares the array's storage. A slice supports all the same operations as an array (it's a collection, it can be iterated, subscripted, and even appended to), so most functions that work on arrays can be written to accept slices too, as we'll see in Chapter 7 when we write generic functions over `Collection`.

There's one surprise that catches every newcomer. A slice keeps the indices of its base collection. The first element of `q2` is not `q2[0]` but `q2[3]`:

```swift
print(q2.startIndex, q2.endIndex)  // "3 6"
print(q2[3])  // "April"
print(q2[0])  // Fatal error: Index out of bounds
```

This is deliberate. Because indices are preserved, an index found in a slice is valid in the base array, and vice versa, which makes algorithms that progressively narrow a range (binary search, parsing) natural and free of offset arithmetic. The rule to remember is to use `startIndex` and `endIndex`, or `first` and `last`, with slices, never literal 0 and `count`. If you need a zero-based array, convert the slice: `Array(q2)`.

Since a slice shares storage with its base, a slice also keeps the whole base array alive. As with substrings, slices are intended for temporary use. Convert a slice to an `Array` before storing it for the long term.

### 4.2.2. Copy-on-Write

Swift collections are values, which means that this code is guaranteed to leave `a` unchanged:

```swift
var a = [1, 2, 3]
var b = a
b[0] = 99
print(a)  // "[1, 2, 3]"
```

But copying an array of a million elements every time it's assigned or passed to a function would be prohibitively expensive, so Swift doesn't. An array is internally a single reference to a heap-allocated buffer that holds the count, the capacity, and the elements. Assigning the array copies only the reference and increments the buffer's reference count, so after `var b = a` both arrays share one buffer. Before any *mutating* operation, an array checks whether its buffer is uniquely referenced. If it is, it modifies the buffer in place. If not, which is the case for `b[0] = 99` above, it first makes a copy of the buffer for itself, and then modifies the copy. This strategy is called *copy-on-write* (COW).

The upshot is that passing a collection to a function, returning it, or storing it in another variable costs O(1), and copies happen only when they're needed to preserve value semantics. Mutating a uniquely referenced array in place, the common case, is as fast as mutating a C array.

You can use the same technique for your own types. The standard library function `isKnownUniquelyReferenced(_:)` reports whether a class instance has exactly one strong reference, which is all you need:

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

This example is a little contrived, since the `[Double]` inside `Buffer` is already copy-on-write, but in real code the class would hold something that isn't a value, like a manually managed block of memory, a C library's object, or a large tree structure, and the struct would give it value semantics.

### 4.2.3. Growing an Array

An array has a *capacity*, the number of elements its buffer can hold before it must be reallocated. When an append exceeds the capacity, the array allocates a larger buffer, roughly double the size, and moves the elements into it. Doubling means that, although an individual append is occasionally expensive, the average cost of an append is constant. You can watch it happen:

```swift
var x: [Int] = []
var lastCapacity = -1
for i in 0..<100 {
    x.append(i)
    if x.capacity != lastCapacity {
        print("count=\(x.count) capacity=\(x.capacity)")
        lastCapacity = x.capacity
    }
}
```

The exact growth pattern is an implementation detail and varies with element size, but a typical run on a 64-bit machine prints:

```
count=1 capacity=2
count=3 capacity=4
count=5 capacity=8
count=9 capacity=16
count=17 capacity=32
count=33 capacity=64
count=65 capacity=128
```

If you know in advance how many elements you'll add, call `reserveCapacity(_:)` first to do the allocation just once. Or, better, construct the array in a single step from a sequence, with `Array(sequence)` or `map`, which sizes the buffer correctly.

Go programmers must remember to write `s = append(s, x)`, because the append may or may not return a slice that shares the original's storage. In Swift, `append` mutates the array variable in place, and the question of sharing doesn't arise: thanks to copy-on-write, it's impossible to observe whether two array values share a buffer.

### 4.2.4. In-Place Array Techniques

Many useful algorithms modify the elements of an array in place. The `reverse` function below reverses the elements of an `[Int]` array by swapping from both ends toward the middle:

```swift
// swiftpl/ch4/rev
func reverse(_ a: inout [Int]) {
    var (i, j) = (0, a.count - 1)
    while i < j {
        a.swapAt(i, j)
        i += 1
        j -= 1
    }
}

var a = [0, 1, 2, 3, 4, 5]
reverse(&a)
print(a)  // "[5, 4, 3, 2, 1, 0]"
```

(The standard library's `reverse()` method does the same for any mutable, bidirectional collection, so in practice you'd write `a.reverse()`.)

A simple way to *rotate* an array left by *n* elements is to apply the reverse function three times, first to the leading *n* elements, then to the remaining elements, and finally to the whole array. The library method works on slices too, and mutating a slice of a `var` array through a range subscript modifies the array's own elements in place:

```swift
var s = [0, 1, 2, 3, 4, 5]
// Rotate s left by two positions.
s[..<2].reverse()
s[2...].reverse()
s.reverse()
print(s)  // "[2, 3, 4, 5, 0, 1]"
```

Let's see more examples of in-place functions. Given a list of strings, the `nonempty` function removes the empty ones. In Go, one writes a loop that reuses the slice's storage; in Swift, the standard library already provides the in-place algorithm as `removeAll(where:)`:

```swift
// swiftpl/ch4/nonempty
var data = ["one", "", "three"]
data.removeAll(where: { $0.isEmpty })
print(data)  // "["one", "three"]"
```

`removeAll(where:)` takes a *closure*, an anonymous function, that is called for each element and returns `true` for elements to remove. Inside a closure, `$0` refers to the first parameter. It runs in linear time, moving each kept element at most once, and doesn't allocate. If you instead want a new array without disturbing the original, use `filter`, which keeps the elements for which the closure returns `true`:

```swift
let nonEmpty = data.filter { !$0.isEmpty }
```

An array can be used to implement a stack. Given an initially empty array `stack`, we can push a new value onto the end with `append`, look at the top with `last`, and pop with `popLast()`, which returns `nil` if the stack is empty:

```swift
var stack: [Int] = []
stack.append(1)  // push 1
stack.append(2)  // push 2
let top = stack.last  // Optional(2)
let popped = stack.popLast()  // Optional(2), stack is now [1]
```

To remove an element from the middle, `remove(at:)` shifts the later elements down to fill the gap, preserving order, in O(*n*) time. If order doesn't matter, it's faster to swap the element with the last one and then remove the last:

```swift
func remove<T>(_ a: inout [T], at i: Int) {
    a.swapAt(i, a.count - 1)
    a.removeLast()
}
```

The `<T>` makes `remove` a *generic* function that works for arrays of any element type. We'll explain generics in Section 7.7.

**Exercise 4.3:** Rewrite our `reverse` to reverse an `ArraySlice<Int>` in place, using `startIndex` and `endIndex` rather than 0 and `count`.

**Exercise 4.4:** Write a version of `rotate` that operates in a single pass.

**Exercise 4.5:** Write an in-place function to eliminate adjacent duplicates in a `[String]` array.

**Exercise 4.6:** Write an in-place function that squashes each run of adjacent Unicode whitespace in the `utf8` bytes of a `[UInt8]` array into a single ASCII space.

**Exercise 4.7:** Modify `Vector` from Section 4.2.2 to count how many times it copies its buffer, and write a short program that shows when copies happen.

## 4.3. Dictionaries and Sets

### 4.3.1. Dictionaries

The hash table is one of the most ingenious and versatile of all data structures. It is an unordered collection of key/value pairs in which all the keys are distinct, and the value associated with a given key can be retrieved, updated, or removed using a constant number of key comparisons on the average, no matter how large the hash table.

In Swift, the hash table is called a *dictionary*, of type `Dictionary<Key, Value>`, written `[Key: Value]`. All of the keys in a given dictionary are of the same type, and all of the values are of the same type, but the keys need not be of the same type as the values. The key type must conform to the `Hashable` protocol, which means its values can be compared with `==` and fed to a hash function. All the basic types are `Hashable`, as are tuples' components, optionals, arrays, and sets whose elements are `Hashable`, and any struct or enum that declares the conformance and whose members are all `Hashable`.

A dictionary can be created with a *dictionary literal*:

```swift
var ages = [
    "alice": 31,
    "charlie": 34,
]
```

and an empty dictionary is written `[:]` (when the type can be inferred) or `[String: Int]()`.

Dictionary elements are accessed through the usual subscript notation. Because the key might not be present, the result is an optional:

```swift
ages["alice"] = 32
print(ages["alice"] as Any)  // "Optional(32)"
print(ages["bob"] as Any)  // "nil"
```

and removed by assigning `nil`, or with `removeValue(forKey:)`, which returns the removed value:

```swift
ages["alice"] = nil  // remove element ages["alice"]
```

The optional result forces you to decide what a missing key means. Often, a missing key should behave as if it had some default value, and the `default:` subscript expresses that. So we can increment a count even if the key isn't there yet:

```swift
ages["bob", default: 0] += 1  // happy birthday!
```

Because a dictionary is a value, a `let` dictionary can't be modified, and a dictionary inside a struct is copied along with the struct.

To enumerate all the key/value pairs in a dictionary, we use a `for`-`in` loop. Successive iterations of the loop yield a tuple with `key` and `value` components:

```swift
for (name, age) in ages {
    print("\(name)\t\(age)")
}
```

The order of dictionary iteration is unspecified, and different runs of the same program will see different orders. (Swift randomizes its hash seed per process, both to discourage code from depending on the order and to defend against denial-of-service attacks that exploit predictable hashing.) To enumerate the pairs in order, we must sort the keys explicitly:

```swift
for name in ages.keys.sorted() {
    print("\(name)\t\(ages[name]!)")
}
```

The `!` is the *force-unwrap* operator. It extracts the value from an optional, trapping if the optional is `nil`. Use it only when you can prove the value is present, as here, since every name came from the dictionary's own keys. A clearer alternative avoids the unwrap altogether by sorting the pairs:

```swift
for (name, age) in ages.sorted(by: { $0.key < $1.key }) {
    print("\(name)\t\(age)")
}
```

Like arrays, dictionaries can be compared with `==` if their values are `Equatable`; two dictionaries are equal if they contain the same keys with equal values.

A few more dictionary operations are worth knowing. `Dictionary(grouping:by:)` groups a sequence's elements by a key computed from each one; `Dictionary(uniqueKeysWithValues:)` builds a dictionary from pairs; `mapValues` transforms every value; `merge` combines two dictionaries, with a closure to resolve conflicts:

```swift
let words = ["apple", "avocado", "banana", "blueberry", "cherry"]
let byLetter = Dictionary(grouping: words, by: { $0.first! })
// ["a": ["apple", "avocado"], "b": ["banana", "blueberry"], "c": ["cherry"]]
let lengths = byLetter.mapValues { $0.count }
// ["a": 2, "b": 2, "c": 1]
```

### 4.3.2. Sets

Go has no set type, so Go programs use a map whose values are ignored. Swift has `Set<Element>`, an unordered collection of distinct `Hashable` elements, with constant-time membership tests and the full complement of set algebra: `union`, `intersection`, `subtracting`, `symmetricDifference`, `isSubset(of:)`, and their in-place versions. A set can be written with an array literal when its type is known:

```swift
let vowels: Set<Character> = ["a", "e", "i", "o", "u"]
print(vowels.contains("e"))  // "true"
```

To illustrate, the program `dedup` reads a sequence of lines and prints only the first occurrence of each distinct line. (It's a variant of the `dup` program from Section 1.3.)

```swift
// swiftpl/ch4/dedup
var seen = Set<String>()  // a set of strings
while let line = readLine() {
    if seen.insert(line).inserted {
        print(line)
    }
}
```

The `insert` method returns a tuple whose `inserted` component is `true` if the element was newly added and `false` if it was already present, so one call both tests and updates the set.

### 4.3.3. Composite Keys

Sometimes we need a dictionary or set whose keys are not simple strings but something richer: a pair of coordinates, a list of strings, a record with several fields. In Go, keys must be comparable, which rules out slices, and a common workaround is to convert each key to a string first. In Swift, no such trick is necessary. Arrays of `Hashable` elements are themselves `Hashable`, and so is any struct whose stored properties are all `Hashable`, if it declares the conformance:

```swift
struct Point: Hashable {
    var x, y: Int
}

var visited = Set<Point>()
visited.insert(Point(x: 1, y: 2))

var counts: [[String]: Int] = [:]
counts[["a", "b"], default: 0] += 1
```

The compiler synthesizes `==` and `hash(into:)` by combining the members' implementations. You can write them yourself if the synthesized equality isn't what you want, for example to compare strings case-insensitively; the only rule is that two values that are `==` must produce the same hash.

Tuples are not `Hashable` (a long-standing limitation of the language), so a struct is the usual choice for a composite key.

### 4.3.4. Example: Counting Characters

Here's an example that counts the occurrences of each distinct Unicode character in its input. Since there are a large number of possible characters, only a small fraction of which would appear in any particular document, a dictionary is a natural way to keep track of just the ones that have been seen and their corresponding counts.

```swift
// swiftpl/ch4/charcount
// Charcount computes counts of Unicode characters.
import Foundation

var counts: [Character: Int] = [:]  // counts of Unicode characters
var utflen = [Int](repeating: 0, count: 5)  // count of lengths of UTF-8 encodings of scalars
var invalid = 0  // count of invalid UTF-8 sequences

let data = FileHandle.standardInput.readDataToEndOfFile()
let text = String(decoding: data, as: UTF8.self)
for c in text {
    if c == "\u{FFFD}" {
        invalid += 1
        continue
    }
    counts[c, default: 0] += 1
    for scalar in c.unicodeScalars {
        utflen[UTF8.width(scalar)] += 1
    }
}

print("char\tcount")
for (c, n) in counts.sorted(by: { $0.value > $1.value }) {
    print("\(c.debugDescription)\t\(n)")
}
print("\nlen\tcount")
for (i, n) in utflen.enumerated() where i > 0 {
    print("\(i)\t\(n)")
}
if invalid > 0 {
    print("\n\(invalid) invalid UTF-8 characters")
}
```

`String(decoding:as:)` decodes the bytes as UTF-8, replacing each ill-formed sequence with the replacement character U+FFFD, which we count separately. (A genuine U+FFFD in the input would also be counted as invalid. That's a limitation the exercise below asks you to fix.) The program also tallies how many scalars need 1, 2, 3, or 4 bytes in UTF-8.

The output is sorted by decreasing frequency. Using `debugDescription` prints each character as a quoted, escaped literal, so that newlines and tabs are visible:

```
char    count
" "     1203
"e"     871
"t"     602
"\n"    211
"é"     4
...
```

**Exercise 4.8:** Fix `charcount` so that a genuine U+FFFD in the input isn't counted as invalid. (Hint: decode the `utf8` bytes yourself with `Unicode.UTF8.ForwardParser`, or compare `String(validating:)` results line by line.)

**Exercise 4.9:** Modify `charcount` to count letters, digits, and so on in their Unicode categories, using properties like `Character.isLetter` and `Unicode.Scalar.Properties.generalCategory`.

**Exercise 4.10:** Write a program `wordfreq` to report the frequency of each word in an input text file. Use `split(whereSeparator:)` with `\.isWhitespace` to break the input into words.

### 4.3.5. Graphs

The value type of a dictionary can itself be a composite type, such as a set or another dictionary. In the following code, the key type of `graph` is `String` and the value type is `Set<String>`, representing a set of strings. Conceptually, `graph` maps a string to a set of related strings, its successors in a directed graph.

```swift
// swiftpl/ch4/graph
var graph: [String: Set<String>] = [:]

func addEdge(_ from: String, _ to: String) {
    graph[from, default: []].insert(to)
}

func hasEdge(_ from: String, _ to: String) -> Bool {
    graph[from]?.contains(to) ?? false
}
```

The `addEdge` function shows the idiomatic way to populate a dictionary lazily, initializing each value when its key first appears. The `default:` subscript returns an empty set for a new key, and `insert` modifies the set in place inside the dictionary, without a copy.

The `hasEdge` function uses *optional chaining*: `graph[from]?.contains(to)` calls `contains` only if `graph[from]` is non-`nil`, and the whole expression evaluates to `nil` otherwise. The `??` then supplies `false` for missing vertices.

**Exercise 4.11:** Write a program that reads a list of lines of the form `a b`, meaning an edge from `a` to `b`, and prints the vertices in topological order. (We'll return to this problem in Section 5.6.)

## 4.4. Structs and Tuples

A *struct* is an aggregate data type that groups together zero or more named values of arbitrary types as a single entity. Each value is called a *stored property*. The classic example of a struct from data processing is the employee record, whose properties are a unique ID, the employee's name, address, date of birth, position, salary, manager, and the like. All of these properties are collected into a single entity that can be copied as a unit, passed to functions and returned by them, stored in arrays, and so on.

These two statements declare a struct type called `Employee` and a variable called `dilbert` that is an instance of an `Employee`:

```swift
import Foundation

struct Employee {
    var id: Int
    var name: String
    var address: String
    var dob: Date
    var position: String
    var salary: Int
    var managerID: Int
}

var dilbert = Employee(
    id: 1, name: "Dilbert", address: "Cubicle 4B", dob: Date(timeIntervalSince1970: 0),
    position: "Engineer", salary: 5000, managerID: 2)
```

The individual properties of `dilbert` are accessed using dot notation like `dilbert.name` and `dilbert.dob`. Because `dilbert` is a `var`, its properties are variables too, so we may assign to them:

```swift
dilbert.salary -= 5000  // demoted, for writing too few lines of code
```

If `dilbert` were a `let`, no property of it could be changed, even if the property is declared with `var`. A struct value is immutable as a whole or mutable as a whole, depending only on how the variable holding it is declared. (Properties declared with `let`, however, can never be changed after initialization, even in a `var` struct.)

The initializer `Employee(id:name:...)` that we called above is the *memberwise initializer*, which the compiler synthesizes for every struct, with one parameter per stored property in declaration order. If a property has a default value, its parameter has a default too. You can write your own initializers, but if you do so in the struct's body, the memberwise initializer is no longer synthesized. If you want both, put your own initializers in an extension.

Here's a function that looks up an employee by ID and gives the employee a raise. The `employees` array is passed `inout`, and the element is modified in place:

```swift
func giveRaise(to id: Int, in employees: inout [Employee], amount: Int) {
    if let i = employees.firstIndex(where: { $0.id == id }) {
        employees[i].salary += amount
    }
}
```

Note that a function that returned an `Employee` and then modified the result wouldn't change the array, because the returned value would be a copy. This is the essence of value semantics, and you'll soon learn to reach for `inout`, or to modify elements in place through a subscript, when you want changes to stick.

Property order is significant to type identity only in the sense that it determines the order of parameters in the memberwise initializer. Two struct declarations with the same properties in the same order are still different types, since every struct declaration creates a new named type.

A struct named `S` can't declare a stored property of the same type `S`, since an aggregate value can't contain itself; its size would be infinite. To build a recursive data structure like a linked list or a tree, you need a level of indirection, which Swift provides through classes or *indirect enums*. The code below uses an indirect enum to implement an insertion sort using a binary tree:

```swift
// swiftpl/ch4/treesort
indirect enum Tree {
    case empty
    case node(Tree, Int, Tree)

    func inserting(_ value: Int) -> Tree {
        switch self {
        case .empty:
            return .node(.empty, value, .empty)
        case let .node(left, v, right):
            if value < v {
                return .node(left.inserting(value), v, right)
            } else {
                return .node(left, v, right.inserting(value))
            }
        }
    }

    func appendValues(to values: inout [Int]) {
        if case let .node(left, v, right) = self {
            left.appendValues(to: &values)
            values.append(v)
            right.appendValues(to: &values)
        }
    }
}

/// Sorts values in place.
func treeSort(_ values: inout [Int]) {
    var root = Tree.empty
    for v in values {
        root = root.inserting(v)
    }
    values.removeAll(keepingCapacity: true)
    root.appendValues(to: &values)
}
```

The `indirect` keyword tells the compiler to store the associated values of each case in a separate heap allocation, so a `Tree` value is just a pointer in size. The `if case let` form matches a single pattern, a handy alternative to a `switch` when only one case is interesting.

Because the tree is a value, `inserting` returns a new tree rather than modifying the old one, sharing all the subtrees it didn't change. (This is a *persistent* data structure; it's elegant, but for a sort it's slower than an in-place structure built from classes. In real code, call `values.sort()`.)

The struct type with no properties is called the *empty struct*, written `struct Empty {}`. It has size zero and carries no information but may be useful nonetheless. Swift's equivalent of Go's `struct{}` used as a set value or signal is usually the empty tuple `()`, also spelled `Void`, which is the return type of functions that don't return anything.

### 4.4.1. Comparing Structs

If all the properties of a struct are `Equatable`, the struct can be `Equatable` too, by declaring the conformance. The compiler synthesizes `==`, which compares the corresponding properties in order:

```swift
struct Point: Equatable {
    var x, y: Int
}

let p = Point(x: 1, y: 2)
let q = Point(x: 2, y: 1)
print(p.x == q.x && p.y == q.y)  // "false"
print(p == q)  // "false"
```

The same goes for `Hashable` and, for structs that should be encoded, `Codable` (Section 4.5). For `Comparable`, there's no single obvious ordering of properties, so the compiler synthesizes `<` only for enums without associated values, ordering the cases by declaration order; for structs, you write `<` yourself.

### 4.4.2. Composition and Forwarding

Go has a feature called *struct embedding* that lets one struct include another anonymously and access its fields as if they were its own. Swift has no direct equivalent. Instead, you compose structs by nesting them as ordinary named properties, and access the inner properties with a longer path:

```swift
struct Point {
    var x, y: Int
}

struct Circle {
    var center: Point
    var radius: Int
}

struct Wheel {
    var circle: Circle
    var spokes: Int
}

var w = Wheel(circle: Circle(center: Point(x: 8, y: 8), radius: 5), spokes: 20)
w.circle.center.x = 8
w.circle.center.y = 8
w.circle.radius = 5
w.spokes = 20
```

When the forwarding matters (when you want `w.x` to mean `w.circle.center.x`), Swift offers *computed properties*, which look like stored properties but run code to get and set their value:

```swift
extension Wheel {
    var x: Int {
        get { circle.center.x }
        set { circle.center.x = newValue }
    }
}
```

Writing these by hand for every property is tedious, and the more general tool is *dynamic member lookup* with *key paths*. A key path such as `\Circle.center` is a value that names a path to a property; a type marked `@dynamicMemberLookup` can forward any property access through a key path subscript:

```swift
@dynamicMemberLookup
struct Wheel {
    var circle: Circle
    var spokes: Int

    subscript<T>(dynamicMember keyPath: WritableKeyPath<Circle, T>) -> T {
        get { circle[keyPath: keyPath] }
        set { circle[keyPath: keyPath] = newValue }
    }
}

var w = Wheel(circle: Circle(center: Point(x: 8, y: 8), radius: 5), spokes: 20)
w.radius = 6  // means w.circle.radius = 6
w.center.x += 1  // means w.circle.center.x += 1
print(w.radius, w.center)  // "6 Point(x: 9, y: 8)"
```

Despite the name, this is all checked at compile time: `w.radius` is accepted only because `\Circle.radius` is a valid writable key path, and its type is known statically. We'll revisit key paths in Section 6.4. For sharing *behavior* rather than data among types, Swift uses protocol extensions, which are the subject of Section 6.3.

Notice the output of `print(w.center)`. When a struct doesn't provide its own `description`, `print` and string interpolation show its type name and its properties, a convenient default for debugging.

### 4.4.3. Tuples

A *tuple* groups several values without declaring a named type. Tuples are written in parentheses, and their elements may be accessed by position, `.0`, `.1`, and so on, or by label if the tuple has labels:

```swift
let http404 = (404, "Not Found")
print(http404.0, http404.1)  // "404 Not Found"

let response = (code: 200, message: "OK")
print(response.code)  // "200"

let (code, message) = response  // destructuring
```

Tuples are ideal for returning several values from a function, as we'll see in Section 5.3, and for temporary groupings within a function. Tuples of up to six `Equatable` or `Comparable` elements can be compared with `==` and `<` (the comparison is lexicographic). But tuples can't conform to protocols or have methods, so once a grouping of values appears in more than one place or deserves a name, a struct is the better choice.

## 4.5. JSON

JavaScript Object Notation (JSON) is a standard notation for sending and receiving structured information. JSON is not the only such notation. XML, ASN.1, and Google's Protocol Buffers serve similar purposes and each has its niche, but because of its simplicity, readability, and universal support, JSON is the most widely used.

Swift has excellent support for encoding and decoding these formats, provided by the `Codable` system: the standard library protocols `Encodable` and `Decodable` (together, `Codable`), plus format-specific coders, such as Foundation's `JSONEncoder` and `JSONDecoder` and `PropertyListEncoder` and `PropertyListDecoder`. The protocols are format-neutral, so the same type declaration works with JSON, property lists, and third-party formats like YAML, MessagePack, or CBOR.

JSON is an encoding of JavaScript values (strings, numbers, booleans, arrays, and objects) as Unicode text. It's an efficient yet readable representation for the basic data types of Chapter 3 and the composite types of this chapter (arrays, structs, and dictionaries).

The basic JSON types are numbers (in decimal or scientific notation), booleans (`true` or `false`), and strings, which are sequences of Unicode code points enclosed in double quotes, with backslash escapes using a notation similar to Swift's, though JSON's `\uhhhh` numeric escapes denote UTF-16 codes, not Unicode scalars.

These basic types may be combined recursively using JSON arrays and objects. A JSON array is an ordered sequence of values, written as a comma-separated list enclosed in square brackets; JSON arrays are used to encode Swift arrays. A JSON object is a mapping from strings to values, written as a sequence of `name:value` pairs separated by commas and surrounded by braces; JSON objects are used to encode Swift structs and dictionaries. For example:

```
boolean      true
number       -273.15
string       "She said \"Hello, 世界\""
array        ["gold", "silver", "bronze"]
object       {"year": 1980,
              "event": "archery",
              "medals": ["gold", "silver", "bronze"]}
```

Consider an application that gathers movie reviews and offers recommendations. Its `Movie` data type and a typical list of values are declared below.

```swift
// swiftpl/ch4/movie
import Foundation

struct Movie: Codable {
    var title: String
    var year: Int
    var color: Bool = false
    var actors: [String]

    enum CodingKeys: String, CodingKey {
        case title
        case year = "released"
        case color
        case actors
    }
}

let movies = [
    Movie(title: "Casablanca", year: 1942, color: false,
          actors: ["Humphrey Bogart", "Ingrid Bergman"]),
    Movie(title: "Cool Hand Luke", year: 1967, color: true,
          actors: ["Paul Newman"]),
    Movie(title: "Bullitt", year: 1968, color: true,
          actors: ["Steve McQueen", "Jacqueline Bisset"]),
    // ...
]
```

Data structures like this are an excellent fit for JSON, and it's easy to convert in both directions. Converting a Swift data structure like `movies` to JSON is called *encoding* (or *marshaling*). Encoding is done by `JSONEncoder`:

```swift
let encoder = JSONEncoder()
do {
    let data = try encoder.encode(movies)
    print(String(decoding: data, as: UTF8.self))
} catch {
    fatalError("JSON encoding failed: \(error)")
}
```

`encode` produces a `Data` value containing a very long string with no extraneous white space; we've folded the lines so it fits:

```
[{"title":"Casablanca","released":1942,"color":false,"actors":["Humphrey Bogart","Ingr
id Bergman"]},{"title":"Cool Hand Luke","released":1967,"color":true,"actors":["Paul Ne
wman"]},{"title":"Bullitt","released":1968,"color":true,"actors":["Steve McQueen","Jacq
ueline Bisset"]}]
```

This compact representation contains all the information, but it's hard to read. For human consumption, set the encoder's `outputFormatting` option. Here we ask for pretty-printing with keys sorted so that the output is stable from run to run:

```swift
encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
```

The output is now:

```
[
  {
    "actors" : [
      "Humphrey Bogart",
      "Ingrid Bergman"
    ],
    "color" : false,
    "released" : 1942,
    "title" : "Casablanca"
  },
  {
    "actors" : [
      "Paul Newman"
    ],
    "color" : true,
    "released" : 1967,
    "title" : "Cool Hand Luke"
  },
  ...
]
```

How did the encoder know what to produce? When a struct declares conformance to `Codable` and all its stored properties are themselves `Codable`, the compiler synthesizes the implementation: an `encode(to:)` method that writes each property, and an `init(from:)` initializer that reads each property back. The property names are used as JSON keys by default.

To use a different name in JSON than in Swift, which is the job of a *field tag* in Go, declare a nested enum named `CodingKeys` conforming to `CodingKey`. Its cases name the properties that participate in coding, and their raw values give the JSON names. Here, `year` appears in JSON as `released`. A property omitted from `CodingKeys` is not encoded or decoded at all, which requires it to have a default value.

What about Go's `omitempty`? The synthesized `Codable` conformance omits a property whose value is `nil` when encoding, and treats a missing key as `nil` when decoding, for any property of optional type. So "this field may be absent" is expressed in the type, as `String?` or `[String]?`, rather than in a tag.

The inverse operation to encoding, decoding JSON and populating a Swift data structure, is done by `JSONDecoder`:

```swift
let decoded = try JSONDecoder().decode([Movie].self, from: data)
print(decoded.map(\.title))  // "["Casablanca", "Cool Hand Luke", "Bullitt"]"
```

The first argument says what type to decode, written as a *metatype*: `[Movie].self` is the type `[Movie]` used as a value. Decoding is strict. If a required key is missing or a value has the wrong type, `decode` throws a `DecodingError` that explains exactly which key, at which path, was wrong. Unknown keys in the input are ignored.

By defining a suitable Swift data structure, we can select which parts of the JSON input to decode and which to discard. If we were interested only in the titles, we could declare a struct with just that property:

```swift
struct Title: Decodable {
    var title: String
}

let titles = try JSONDecoder().decode([Title].self, from: data)
print(titles)
// "[movie.Title(title: "Casablanca"), movie.Title(title: "Cool Hand Luke"), movie.Title(title: "Bullitt")]"
```

(When `print` shows the elements of a collection, it uses each element's *debug* description, which qualifies a struct's name with its module, here `movie`.)

Many web services provide a JSON interface: make a request with HTTP and back comes the desired information in JSON format. To illustrate, let's query the GitHub issue tracker using its web-service interface. First we'll define the necessary types, in a library module called `GitHub`:

```swift
// swiftpl/ch4/github/Sources/GitHub/GitHub.swift
// Package GitHub provides a Swift API for the GitHub issue tracker.
// See https://docs.github.com/en/rest/search/search#search-issues-and-pull-requests.
import Foundation

public let issuesURL = "https://api.github.com/search/issues"

public struct IssuesSearchResult: Decodable, Sendable {
    public var totalCount: Int
    public var items: [Issue]
}

public struct Issue: Decodable, Sendable {
    public var number: Int
    public var htmlUrl: String
    public var title: String
    public var state: String
    public var user: User
    public var createdAt: Date
    public var body: String?  // in Markdown format
}

public struct User: Decodable, Sendable {
    public var login: String
    public var htmlUrl: String
}
```

The GitHub API uses `snake_case` for its JSON keys, like `total_count` and `created_at`, while Swift convention calls for `camelCase`. Rather than writing `CodingKeys` for every type, we'll tell the decoder to convert all keys with its `keyDecodingStrategy` option. Likewise, dates in the GitHub API are ISO 8601 strings like `"2025-09-30T21:30:00Z"`, and the `dateDecodingStrategy` option tells the decoder to parse them into `Date` values.

The `searchIssues` function makes an HTTP request and decodes the result as JSON. Since the query terms presented by a user could contain characters like `?` and `&` that have special meaning in a URL, we build the URL with `URLComponents`, which escapes them correctly.

```swift
// swiftpl/ch4/github/Sources/GitHub/Search.swift
import Foundation
#if canImport(FoundationNetworking)
import FoundationNetworking
#endif

public enum SearchError: Error {
    case badStatus(Int)
}

/// Queries the GitHub issue tracker.
public func searchIssues(_ terms: [String]) async throws -> IssuesSearchResult {
    var components = URLComponents(string: issuesURL)!
    components.queryItems = [URLQueryItem(name: "q", value: terms.joined(separator: " "))]
    var request = URLRequest(url: components.url!)
    request.setValue("application/vnd.github+json", forHTTPHeaderField: "Accept")

    let (data, response) = try await URLSession.shared.data(for: request)
    if let http = response as? HTTPURLResponse, http.statusCode != 200 {
        throw SearchError.badStatus(http.statusCode)
    }

    let decoder = JSONDecoder()
    decoder.keyDecodingStrategy = .convertFromSnakeCase
    decoder.dateDecodingStrategy = .iso8601
    return try decoder.decode(IssuesSearchResult.self, from: data)
}
```

`SearchError` is an enum conforming to the `Error` protocol, which makes its values throwable; the `badStatus` case carries the HTTP status code. Section 5.4 discusses errors in depth.

The `issues` program prints a table of the issues matching its search terms:

```swift
// swiftpl/ch4/github/Sources/issues/main.swift
// Issues prints a table of GitHub issues matching the search terms.
import GitHub

let result = try await searchIssues(Array(CommandLine.arguments.dropFirst()))
print("\(result.totalCount) issues:")
for item in result.items {
    let number = String(item.number).leftPadded(to: 5, with: " ")
    let login = item.user.login.padding(to: 9)
    print("#\(number) \(login) \(item.title.prefix(55))")
}

extension String {
    func leftPadded(to width: Int, with pad: Character) -> String {
        String(repeating: pad, count: max(0, width - count)) + self
    }
    func padding(to width: Int) -> String {
        count >= width ? String(prefix(width)) : self + String(repeating: " ", count: width - count)
    }
}
```

The command-line arguments specify the search terms. The command below queries the Swift project's issue tracker for open issues about JSON decoding:

```
$ swift run issues repo:swiftlang/swift-foundation is:open json decoder
13 issues:
# 1234 someuser  JSONDecoder: improve error message for type mismatch
# 1187 another   Decoding large integers loses precision
...
```

(The issues you see will of course be different; GitHub's data changes daily, and the results above are illustrative.)

The GitHub web-service interface at `https://docs.github.com/en/rest` has many more features than we have space for here.

**Exercise 4.12:** Modify `issues` to report the results in age categories, say less than a month old, less than a year old, and more than a year old.

**Exercise 4.13:** Build a tool that lets users create, read, update, and close GitHub issues from the command line, invoking their preferred text editor when substantial text input is required.

**Exercise 4.14:** The popular web comic *xkcd* has a JSON interface. For example, a request to `https://xkcd.com/571/info.0.json` produces a detailed description of comic 571, one of many favorites. Download each URL (once!) and build an offline index. Write a tool `xkcd` that, using this index, prints the URL and transcript of each comic that matches a search term provided on the command line.

**Exercise 4.15:** The JSON-based web service of the Open Movie Database lets you search `https://omdbapi.com/` for a movie by name and download its poster image. Write a tool `poster` that downloads the poster image for the movie named on the command line.

## 4.6. Text Templates with String Interpolation

The simplest kind of formatting, as in the `issues` program, is done by string interpolation. But sometimes formatting must be more elaborate, and it's desirable to separate the format from the code more completely. Go solves this with its `text/template` and `html/template` packages, which interpret a small template language at run time. Swift takes a different approach: its string interpolation is *extensible*, so templates are written as ordinary Swift code, checked by the compiler, using interpolations tailored to the job.

### 4.6.1. Custom Interpolations

When the compiler sees a string literal with interpolations, like `"\(n) issues"`, it translates it into a series of calls on an *interpolation* object: `appendLiteral(_:)` for each literal segment and `appendInterpolation(...)` for each `\(...)` segment, passing the contents of the parentheses as the arguments. For ordinary strings, the interpolation type is `DefaultStringInterpolation`. Since `appendInterpolation` is just a method, we can add overloads to it in an extension, with any argument labels we like.

Here's an interpolation that prints how many days ago a date was:

```swift
// swiftpl/ch4/issuesreport
import Foundation

extension DefaultStringInterpolation {
    mutating func appendInterpolation(daysAgo date: Date) {
        let days = Int(Date.now.timeIntervalSince(date) / (24 * 60 * 60))
        appendLiteral(String(days))
    }
}
```

and here's a report on GitHub issues that uses it:

```swift
// swiftpl/ch4/issuesreport
import GitHub

func report(_ result: IssuesSearchResult) -> String {
    var out = "\(result.totalCount) issues:\n"
    for item in result.items {
        out += """
            ----------------------------------------
            Number: \(item.number)
            User:   \(item.user.login)
            Title:  \(item.title.prefix(64))
            Age:    \(daysAgo: item.createdAt) days

            """
    }
    return out
}

let result = try await searchIssues(Array(CommandLine.arguments.dropFirst()))
print(report(result))
```

Compared with a run-time template language, this approach has a number of advantages: typos in property names are compile-time errors, every value has its proper type (`createdAt` is a `Date`, not a string), and the "template" can use any Swift code, such as loops, conditionals, and function calls, with no new syntax to learn. The price is that the template can't be changed without recompiling, which is rarely a problem in practice.

The output looks like this:

```
$ swift run issuesreport repo:swiftlang/swift-foundation is:open json decoder
13 issues:
----------------------------------------
Number: 1234
User:   someuser
Title:  JSONDecoder: improve error message for type mismatch
Age:    41 days
----------------------------------------
...
```

### 4.6.2. HTML with Automatic Escaping

Now let's generate HTML. HTML introduces a hazard that plain text doesn't: any text taken from outside the program, such as an issue title, might contain characters like `<` and `&` that have special meaning in HTML, either by accident or maliciously. Inserting it into a page without escaping is a classic security vulnerability, an *injection attack*. Go's `html/template` package escapes such text automatically.

We can get the same guarantee in Swift with a custom type that is created from a string literal, whose interpolation escapes everything except values that are already known to be HTML:

```swift
// swiftpl/ch4/issueshtml
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

A type conforms to `ExpressibleByStringInterpolation` by providing an initializer that takes a finished interpolation, and the associated `Interpolation` type does the work. The literal parts of the string, which were written by the programmer, are copied verbatim. Every interpolated value is converted to a string and escaped, *unless* it is itself an `HTML` value, in which case the more specific overload is chosen and the value is inserted as is. This is what makes nesting work: a row built as `HTML` can be interpolated into a table without being escaped twice.

With this type, the issue list becomes:

```swift
func issueList(_ result: IssuesSearchResult) -> HTML {
    var rows = ""
    for item in result.items {
        let row: HTML = """
            <tr>
              <td><a href='\(item.htmlUrl)'>\(item.number)</a></td>
              <td>\(item.state)</td>
              <td><a href='\(item.user.htmlUrl)'>\(item.user.login)</a></td>
              <td><a href='\(item.htmlUrl)'>\(item.title)</a></td>
            </tr>

            """
        rows += row.description
    }
    return """
        <h1>\(result.totalCount) issues</h1>
        <table>
        <tr style='text-align: left'>
          <th>#</th><th>State</th><th>User</th><th>Title</th>
        </tr>
        \(HTML(stringLiteral: rows))
        </table>
        """
}

let result = try await searchIssues(Array(CommandLine.arguments.dropFirst()))
print(issueList(result))
```

The `HTML(stringLiteral: rows)` conversion is the one place where we tell the type system "trust this string," and it's safe here because every row was itself built as `HTML`. (A more careful design would keep the rows as `[HTML]` and give `HTML` a `joined()` method, so that no raw string ever needs to be trusted; that's Exercise 4.17.)

If an issue's title were `<script>alert('pwned')</script>`, the page would display that text literally:

```html
<td><a href='...'>&lt;script&gt;alert(&#39;pwned&#39;)&lt;/script&gt;</a></td>
```

The compiler chooses the escaping or non-escaping overload based on the static type of each interpolated value, so there's no way to forget to escape a `String`: you can only avoid escaping by explicitly constructing an `HTML`, which is easy to find and review. This pattern, using the type system to distinguish trusted from untrusted strings, is a powerful one, and it applies equally to SQL queries, shell commands, and log messages.

For larger sites, Swift packages such as Elementary, Plot, and Leaf provide complete type-safe HTML DSLs or traditional template engines.

**Exercise 4.16:** Create a web server that queries GitHub once and then allows navigation of the list of bug reports, milestones, and users.

**Exercise 4.17:** Add a static method `HTML.joined(_ fragments: [HTML]) -> HTML` and use it to eliminate `HTML(stringLiteral:)` from `issueList`. Then make `init(stringLiteral:)` impossible to call with a non-literal string. (Hint: does it need to be? Consider what `ExpressibleByStringLiteral` requires.)
