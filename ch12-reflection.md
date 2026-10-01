# 12. Reflection

Swift provides a mechanism to update variables and inspect their values at run time, to call their methods, and to apply the operations intrinsic to their representation, all without knowing their types at compile time. This mechanism is called *reflection*. Reflection also lets us treat types themselves as first-class values.

Swift's reflection is deliberately more modest than Go's. Swift's designers favor static guarantees, and the language is compiled with aggressive whole-program optimization that benefits from knowing types in advance, so Swift reflection is mostly *read-only*: you can inspect a value's structure with `Mirror`, but you can't, in general, discover methods by name, call them, or assign to arbitrary fields. What Swift offers in place of general run-time reflection are *compile-time* mechanisms that do the same jobs more safely: protocols with synthesized conformances like `Codable`, key paths, generics, and macros. In this chapter, we'll explore the run-time features that exist, and see how Swift programmers get the effects that Go programmers get from reflection by other means.

## 12.1. Why Reflection?

Sometimes we need to write a function capable of dealing uniformly with values of types that don't satisfy a common protocol, don't have a known representation, or don't exist at the time we design the function. Or even all three.

A familiar example is the formatting logic within `print`, which can print an arbitrary value of any type, even a user-defined one. Let's try to implement a function like it, called `format`, which takes an `Any` and returns a string. We might begin with a type switch to test whether the value has a primitive type (`Int`, `Bool`, `String`, and so on), and define a case for each:

```swift
func format(_ value: Any) -> String {
    switch value {
    case let x as Int: return String(x)
    case let x as Bool: return x ? "true" : "false"
    case let x as String: return x.debugDescription
    // ...
    default:
        // array, dictionary, struct, class, enum, ...???
    }
}
```

But how do we deal with other types, like `[String]`, `[String: Int]`, or `Point`? We could add more cases, but the number of such types is infinite. And how would we deal with named types, like `Celsius`? If the type switch had a case for `Double`, it wouldn't match `Celsius` even though its underlying representation is a `Double`, since `Celsius` is a distinct type. And user-defined types like `Point`, `Employee`, or `Movie` can't be known in advance by a library at all.

Without a way to inspect the representation of values of unknown types, we quickly get stuck. What we need is reflection.

Of course, the Swift standard library does solve this problem, and it solves it in two layers. The first is *protocols*: types that want custom output conform to `CustomStringConvertible` or `CustomDebugStringConvertible`, and `print` uses them, as we saw in Section 7.12. The second, for types that don't conform, is `Mirror`, which `print` falls back on, and which is the topic of this chapter.

## 12.2. `Mirror`

A `Mirror` is a value that describes the structure of another value. You create one by passing the value to the initializer `Mirror(reflecting:)`, and then ask it questions. Its most important properties are:

- `subjectType`, the type of the value being reflected, as a `Any.Type`.
- `displayStyle`, an optional enum that says what kind of thing the value is: `.struct`, `.class`, `.enum`, `.tuple`, `.optional`, `.collection`, `.dictionary`, `.set`, or `.foreignReference`.
- `children`, a collection of `(label: String?, value: Any)` pairs, one for each stored property, element, or associated value.
- `superclassMirror`, a mirror of the superclass, for classes.

Here's an example:

```swift
// swiftpl/ch12/mirror
struct Point {
    var x: Int
    var y: Int
}

struct Circle {
    var center: Point
    var radius: Double
    var label: String?
}

let c = Circle(center: Point(x: 1, y: 2), radius: 3.5, label: "unit")
let m = Mirror(reflecting: c)
print(m.subjectType)  // "Circle"
print(m.displayStyle!)  // struct

for child in m.children {
    print(child.label ?? "_", "=", child.value)
}
// center = Point(x: 1, y: 2)
// radius = 3.5
// label = Optional("unit")
```

Each child's `value` has the static type `Any`, so the function that deals with it must itself discriminate with a cast, or build another mirror. That recursion is the core of nearly everything built on `Mirror`.

The mirrors of the other kinds of values show what the `children` of each look like:

```swift
print(Array(Mirror(reflecting: [10, 20]).children).map { "\($0.label ?? "nil"): \($0.value)" })
// ["nil: 10", "nil: 20"]: array elements have no labels

for (label, value) in Mirror(reflecting: (name: "Ada", year: 1815)).children {
    print(label ?? "_", value)  // "name Ada", "year 1815"
}

enum Shape { case circle(radius: Double), square(side: Double) }
let e = Mirror(reflecting: Shape.circle(radius: 2))
print(e.displayStyle!)  // enum
print(e.children.first!.label!, e.children.first!.value)  // "circle (radius: 2.0)"
```

For an enum, there is at most one child: the case name is its label, and its value is the associated value (or tuple of associated values). A case with no associated values has no children, so the case name can't be found from the mirror; you'd print the enum itself, which finds it by other means.

For a dictionary, the `children` are key/value *pairs*, as `(key: ..., value: ...)` tuples; for a set, the elements; for an optional, either no children (`nil`) or one (the wrapped value).

Notice what `Mirror` *doesn't* offer. It can't change a value, can't look up a child by name except by scanning `children`, can't list methods or call them, and can't see computed properties, static members, or the initializers of the type. It shows only a snapshot of *stored* state, which the type's author may even have customized. A type can choose how it appears to a mirror by conforming to `CustomReflectable` and returning its own `Mirror`, for example to hide private fields or to expose a computed summary:

```swift
struct Account: CustomReflectable {
    var owner: String
    private var pin: String

    var customMirror: Mirror {
        Mirror(self, children: ["owner": owner])  // doesn't reveal pin
    }
}
```

That fits the rest of Swift's design: reflection is a tool for debugging, display, and serialization of data, not a back door around encapsulation. (Mirror *can* read `private` stored properties, which is why types with secrets customize it. Treat it as a debugging window, and don't rely on private details for logic.)

## 12.3. `display`, a Recursive Value Printer

Let's use `Mirror` to write a function that prints the structure of any value, much like Swift's `dump` function (which does the same job and is built into the standard library). Our function, `display`, takes an arbitrary value and prints one line for each leaf, labeled with the path used to reach it. This kind of function is useful for debugging, and it illustrates the recursion that underlies every reflective traversal.

```swift
// swiftpl/ch12/display
func display(_ name: String, _ value: Any) {
    print("display \(name) (\(type(of: value))):")
    walk(name, value)
}

private func walk(_ path: String, _ value: Any) {
    let m = Mirror(reflecting: value)
    switch m.displayStyle {
    case .struct, .class, .tuple:
        for (i, child) in m.children.enumerated() {
            let label = child.label ?? ".\(i)"
            walk("\(path).\(label)", child.value)
        }
    case .collection, .set:
        for (i, child) in m.children.enumerated() {
            walk("\(path)[\(i)]", child.value)
        }
    case .dictionary:
        for child in m.children {
            // Each child is a (key, value) tuple.
            let pair = Array(Mirror(reflecting: child.value).children)
            walk("\(path)[\(format(pair[0].value))]", pair[1].value)
        }
    case .optional:
        if let wrapped = m.children.first {
            walk("\(path)!", wrapped.value)
        } else {
            print("\(path) = nil")
        }
    case .enum:
        if let payload = m.children.first {
            walk("\(path).\(payload.label ?? "?")", payload.value)
        } else {
            print("\(path) = \(value)")
        }
    default:
        print("\(path) = \(format(value))")
    }
}

/// Formats a leaf value.
func format(_ value: Any) -> String {
    switch value {
    case let s as String: return s.debugDescription
    case let c as Character: return c.debugDescription
    default: return "\(value)"
    }
}
```

The function dispatches on the *kind* of value, as reported by `displayStyle`. Aggregates (structs, classes, and tuples) are walked field by field, with the field name appended to the path. Collections and sets are walked by index. For a dictionary, each child is a pair whose own mirror has two children, key and value. An optional is either empty or has one child, and an enum with a payload is walked through the payload. Everything else is a leaf, and printed using `format`, which handles the strings and characters that deserve quoting.

(Strings are displayed as leaves because their `displayStyle` is `nil`. A `String` is a collection of characters, but its mirror has no children; the same holds for `Int`, `Double`, `Bool`, and most other basic types.)

Let's apply `display` to some values:

```swift
struct Movie {
    var title: String
    var year: Int
    var color: Bool
    var actors: [String]
    var oscars: [String: Int]
    var sequel: Movie? { nil }  // computed: not shown
}

let strangelove = Movie(
    title: "Dr. Strangelove", year: 1964, color: false,
    actors: ["Peter Sellers", "George C. Scott"],
    oscars: ["Best Picture": 0, "Best Actor": 0])
display("strangelove", strangelove)
```

This prints:

```
display strangelove (Movie):
strangelove.title = "Dr. Strangelove"
strangelove.year = 1964
strangelove.color = false
strangelove.actors[0] = "Peter Sellers"
strangelove.actors[1] = "George C. Scott"
strangelove.oscars["Best Picture"] = 0
strangelove.oscars["Best Actor"] = 0
```

(The dictionary order may differ from run to run.) Notice that the computed property `sequel` doesn't appear, because it isn't stored state.

Reflection works on values from the standard library too: `display("range", 1...3)` shows the range's `lowerBound` and `upperBound`. But it also reveals *implementation* details. The stored properties of a type like `Date` are private, are not part of its API, and nothing promises they'll look the same in the next release. That's one more reason to prefer protocols for anything the program's logic depends on.

### 12.3.1. Cycles

`display` works for any value that is a tree, but class instances can form cycles (a doubly linked list or a graph), and our function would then run forever, or until it overflowed the stack. `dump`, the standard library's version of this function, avoids that by tracking the objects it has already visited. We can do the same, using `ObjectIdentifier`, which is a hashable identity for any class instance:

```swift
private func walk(_ path: String, _ value: Any, _ visited: inout Set<ObjectIdentifier>) {
    let m = Mirror(reflecting: value)
    if m.displayStyle == .class {
        let id = ObjectIdentifier(value as AnyObject)
        guard visited.insert(id).inserted else {
            print("\(path) = (cycle)")
            return
        }
    }
    // ...as before, passing visited along the recursive calls...
}
```

Go's `Display` function of the same name in its book has the same problem for pointers and maps, and the same remedy. Swift's value types (structs, enums, arrays, dictionaries) can't form cycles, so only classes need to be tracked.

**Exercise 12.1:** Extend `display` so that it also shows the superclass fields of a class instance, using `superclassMirror`.

**Exercise 12.2:** Add the cycle-detection logic above and test it with a doubly linked list.

**Exercise 12.3:** Extend `display` to produce the output as JSON, where dictionary keys that are not strings are rendered as strings. How does this compare to the output of a real `JSONEncoder`?

## 12.4. Example: Encoding S-Expressions

`display` is just a debugging routine, but it's not far from being an *encoder*. In this section, we'll write one: a function that encodes an arbitrary Swift value into *S-expression* notation, the syntax of Lisp. S-expressions are an elegantly simple format: they have atoms (numbers, strings, symbols) and lists, and nothing else. Here's the encoding we'll use for each type:

```
42                                   integer
"hello"                              string (a Go-style quoted literal)
t / nil                              booleans (true is t, false is nil)
(1 2 3)                              array or set
((key1 value1) (key2 value2))        dictionary
((Name "Ada") (Born 1815))           struct, as a list of (field value) pairs
(Case payload)                       enum
```

The encoder takes any `Any` and returns a `String`, throwing an error for types it can't handle:

```swift
// swiftpl/ch12/sexpr
enum SExprError: Error {
    case unsupported(type: Any.Type)
}

func marshal(_ value: Any) throws -> String {
    var out = ""
    try encode(value, into: &out, indent: 0)
    return out
}

private func encode(_ value: Any, into out: inout String, indent: Int) throws {
    switch value {
    case let x as Bool:
        out += x ? "t" : "nil"
        return
    case let x as any BinaryInteger:
        out += "\(x)"
        return
    case let x as any BinaryFloatingPoint:
        out += "\(x)"
        return
    case let x as String:
        out += x.debugDescription  // quoted, with escapes
        return
    case let x as Character:
        out += String(x).debugDescription
        return
    default:
        break
    }

    let m = Mirror(reflecting: value)
    switch m.displayStyle {
    case .optional:
        if let wrapped = m.children.first {
            try encode(wrapped.value, into: &out, indent: indent)
        } else {
            out += "nil"
        }

    case .collection, .set:
        out += "("
        for (i, child) in m.children.enumerated() {
            if i > 0 { out += "\n" + String(repeating: " ", count: indent + 1) }
            try encode(child.value, into: &out, indent: indent + 1)
        }
        out += ")"

    case .dictionary:
        out += "("
        for (i, child) in m.children.enumerated() {
            if i > 0 { out += "\n" + String(repeating: " ", count: indent + 1) }
            let pair = Array(Mirror(reflecting: child.value).children)
            out += "("
            try encode(pair[0].value, into: &out, indent: indent + 2)
            out += " "
            try encode(pair[1].value, into: &out, indent: indent + 2)
            out += ")"
        }
        out += ")"

    case .struct, .class:
        out += "("
        for (i, child) in m.children.enumerated() {
            if i > 0 { out += "\n" + String(repeating: " ", count: indent + 1) }
            let name = child.label ?? "_"
            out += "(\(name) "
            try encode(child.value, into: &out, indent: indent + name.count + 3)
            out += ")"
        }
        out += ")"

    case .enum:
        if let payload = m.children.first {
            out += "(\(payload.label ?? "case") "
            try encode(payload.value, into: &out, indent: indent + 2)
            out += ")"
        } else {
            out += "\(value)"  // a case without payload prints as its name
        }

    default:
        throw SExprError.unsupported(type: type(of: value))
    }
}
```

Compare this to `display`. The structure is the same, a walk guided by `displayStyle`, but now each case produces text in a defined syntax, with an indentation scheme that lines up the elements of lists under each other. The first `switch` handles the atoms using *protocol* casts: `any BinaryInteger` matches every integer type, and `any BinaryFloatingPoint` every floating-point type, so we needn't enumerate `Int8`, `UInt32`, `Float`, and the rest. (This is the same technique as in Section 7.12, the sensible use of a cast to a protocol.)

Let's marshal the `Movie` structure of the previous section, adapted to have a nicer set of fields:

```swift
struct Movie {
    var title: String
    var subtitle: String
    var year: Int
    var color: Bool
    var actor: [String: String]
    var oscars: [String]
    var sequel: String?
}

let strangelove = Movie(
    title: "Dr. Strangelove",
    subtitle: "How I Learned to Stop Worrying and Love the Bomb",
    year: 1964,
    color: false,
    actor: [
        "Dr. Strangelove": "Peter Sellers",
        "Grp. Capt. Lionel Mandrake": "Peter Sellers",
        "Pres. Merkin Muffley": "Peter Sellers",
        "Gen. Buck Turgidson": "George C. Scott",
        "Brig. Gen. Jack D. Ripper": "Sterling Hayden",
        "Maj. T.J. \"King\" Kong": "Slim Pickens",
    ],
    oscars: [
        "Best Actor (Nomin.)",
        "Best Adapted Screenplay (Nomin.)",
        "Best Director (Nomin.)",
        "Best Picture (Nomin.)",
    ],
    sequel: nil)

print(try marshal(strangelove))
```

```
((title "Dr. Strangelove")
 (subtitle "How I Learned to Stop Worrying and Love the Bomb")
 (year 1964)
 (color nil)
 (actor (("Dr. Strangelove" "Peter Sellers")
         ("Grp. Capt. Lionel Mandrake" "Peter Sellers")
         ("Pres. Merkin Muffley" "Peter Sellers")
         ("Gen. Buck Turgidson" "George C. Scott")
         ("Brig. Gen. Jack D. Ripper" "Sterling Hayden")
         ("Maj. T.J. \"King\" Kong" "Slim Pickens")))
 (oscars ("Best Actor (Nomin.)"
          "Best Adapted Screenplay (Nomin.)"
          "Best Director (Nomin.)"
          "Best Picture (Nomin.)"))
 (sequel nil))
```

(Again, the order of the entries in the `actor` dictionary varies from run to run; sort the children first if you want a stable output, as in Exercise 12.5.)

### 12.4.1. The Compile-Time Alternative: `Encoder`

The marshaler we wrote shows how reflection works, but it's not how Swift programmers would normally do this job. We discovered in Chapter 4 that `Codable` supports JSON and property lists without any reflection at all. How?

The answer is that the compiler *generates code*. When a type declares `Codable` conformance, the compiler writes, at compile time, an `encode(to:)` method that calls `container.encode(...)` for each stored property by name, with proper types. There's no run-time inspection of layouts, and so no `Any` and no casts, and the generated code is as fast as if you'd written it by hand. The `Encoder` protocol that receives these calls is format-neutral. Anyone who writes a type conforming to `Encoder` (a few hundred lines for a simple format) gets an encoder for *every* `Encodable` type, with the same customization points (`CodingKeys`, custom `encode(to:)`) that JSON has.

The advantages over `Mirror`-based encoding are substantial:

- **Speed.** No type-erased `Any` boxing, no run-time metadata walking.
- **Type safety.** The encoding depends on declared types, not on whatever happens to be inside an `Any`.
- **Control.** A type can customize its encoding by providing its own `encode(to:)`, rename keys with `CodingKeys`, or skip properties, and these customizations apply to *every* format.
- **Decoding.** `Decodable` makes the inverse possible, constructing values of types the library has never seen. As we'll see in Section 12.6, that's the great limitation of `Mirror`.

The price is that a type must declare its conformance (or have it declared in an extension) in advance. A value of a type that isn't `Codable` can't be encoded this way, whereas `Mirror` can look at anything.

For that reason, the S-expression encoder is better written as an `Encoder`. A faithful version is Exercise 12.6. We continue with the `Mirror` version here, because it's the clearest illustration of reflection, and because it's a closer analogue of the original.

**Exercise 12.4:** Add support for floating-point formatting, tuples, and `nil` optionals in the S-expression marshaler, and test it on a variety of values.

**Exercise 12.5:** Sort the entries of dictionaries by key so that the output is stable. Which type constraint do you need to put on the dictionary's key type, given that you only have an `Any`?

**Exercise 12.6:** Implement a type conforming to `Encoder` that produces S-expressions, so that any `Encodable` value can be written with it. Compare its output with the `Mirror`-based marshaler for the `Movie` type, once you've made `Movie` conform to `Codable`.

**Exercise 12.7:** Adapt the encoder to produce JSON. Do you reproduce `JSONEncoder`'s output for the types in Section 4.5? What cases does `Mirror` get wrong that `Codable` gets right?

## 12.5. Setting Values with Key Paths

So far, we've used reflection only to *inspect* values. Go's reflection can also *modify* them, through `reflect.Value`'s `Set` methods. `Mirror` can't do that, and Swift has no general counterpart. Swift's answer to "I want to read or write a property chosen at run time" is the *key path*, which we introduced in Section 6.4.

A key path is a value that describes a path to a property. It's checked by the compiler, so you can't make one for a property that doesn't exist, and it has a precise type: `KeyPath<Root, Value>` for read-only access, `WritableKeyPath<Root, Value>` for properties of a mutable value, and `ReferenceWritableKeyPath<Root, Value>` for properties of a class instance.

```swift
// swiftpl/ch12/keypath
struct Person {
    var name: String
    var age: Int
}

var p = Person(name: "Ada", age: 36)

let age = \Person.age  // WritableKeyPath<Person, Int>
p[keyPath: age] += 1
print(p[keyPath: age])  // "37"
```

The point is that a key path is a *value*, which can be passed to functions, stored in tables, and chosen at run time. For example, this function sets a field to a value across an array of records:

```swift
func setAll<T, V>(_ items: inout [T], _ keyPath: WritableKeyPath<T, V>, to value: V) {
    for i in items.indices {
        items[i][keyPath: keyPath] = value
    }
}

var people = [Person(name: "Ada", age: 36), Person(name: "Alan", age: 41)]
setAll(&people, \.age, to: 0)
```

and a *table* of key paths lets us choose a property by name at run time, like Go's `FieldByName`, but with the choice of properties fixed (and checked) at compile time:

```swift
let fields: [String: PartialKeyPath<Person>] = [
    "name": \Person.name,
    "age": \Person.age,
]

func get(_ name: String, from p: Person) -> Any? {
    fields[name].map { p[keyPath: $0] }
}

print(get("age", from: p) as Any)  // "Optional(37)"
```

`PartialKeyPath<Root>` and `AnyKeyPath` are type-erased forms that allow keys of different value types to be stored in one collection; reading through them gives an `Any?`, and writing requires a cast to the concrete `WritableKeyPath` type.

Although the *table* is hand-written, this is exactly the mechanism that powers dynamic-looking APIs in Swift: SwiftUI's property bindings, the `@Observable` machinery, `sorted(using: KeyPathComparator(\.age))`, the `\.name` shorthand for closures, and Swift Data's predicates and sort descriptors. Because key paths are statically typed, mistakes like `\.agee` or assigning a string to an `Int` property are compile-time errors.

### 12.5.1. Setting Values by Name from Text

Parsing a text format, such as command-line flags or an INI file, into the fields of a configuration struct is a classic use of Go's `reflect.Value.Set`. In Swift, we register the key paths and write a conversion function for each type that can be set:

```swift
// swiftpl/ch12/populate
struct Config {
    var host = "localhost"
    var port = 8080
    var debug = false
    var ratio = 1.0
}

/// A setter knows how to parse text and store it in one property of T.
struct Setter<T> {
    let set: (inout T, String) -> Bool  // reports whether the text was valid

    init<V: LosslessStringConvertible>(_ keyPath: WritableKeyPath<T, V>) {
        set = { target, text in
            guard let v = V(text) else { return false }
            target[keyPath: keyPath] = v
            return true
        }
    }
}

let setters: [String: Setter<Config>] = [
    "host": Setter(\.host),
    "port": Setter(\.port),
    "debug": Setter(\.debug),
    "ratio": Setter(\.ratio),
]

var config = Config()
for (key, text) in [("port", "9000"), ("debug", "true"), ("ratio", "x")] {
    guard let setter = setters[key] else {
        print("unknown key \(key)")
        continue
    }
    if !setter.set(&config, text) {
        print("bad value \(text.debugDescription) for \(key)")
    }
}
print(config)  // Config(host: "localhost", port: 9000, debug: true, ratio: 1.0)
```

The generic initializer takes a key path to a property of *any* type `V` that conforms to `LosslessStringConvertible`, a standard protocol for types that can be converted to and from strings (`Int`, `Double`, `Bool`, `String`, and many others), and captures it in a closure. This does what Go's `populate` function does using reflection and a type switch on `Kind`, but with the types checked by the compiler. The cost is that each field must be listed once. The macro we'll meet in Section 12.8 can eliminate even that.

## 12.6. Example: Decoding S-Expressions

We now turn to decoding, the inverse of encoding. In Go, decoding requires `reflect.Value`'s ability to *create* and *set* values of types not known when the decoder was written. This is exactly what `Mirror` can't do, since a mirror can only read. How, then, can a Swift program decode a format of its own into arbitrary types? Via `Decodable`, the compile-time-generated protocol conformance, as we saw in Section 4.5.

To implement a decoder for S-expressions, we write a type that conforms to the `Decoder` protocol. When a `Decodable` type's synthesized `init(from:)` runs, it asks the decoder for a *container* and then asks the container for values of specific types by key: `try container.decode(Int.self, forKey: .year)`. Our job is to answer those requests from the S-expression text. The compiler-generated code does all the field handling for us.

We start with a lexer and a parser that turn text into a simple tree:

```swift
// swiftpl/ch12/sexprdecode
indirect enum SExpr: Equatable {
    case symbol(String)  // t, nil, or a field name
    case int(Int)
    case double(Double)
    case string(String)
    case list([SExpr])
}

struct SExprParser {
    private let chars: [Character]
    private var pos = 0

    init(_ text: String) { chars = Array(text) }

    mutating func parse() throws -> SExpr {
        skipSpace()
        guard pos < chars.count else { throw DecodeError.unexpectedEnd }
        let c = chars[pos]
        switch c {
        case "(":
            pos += 1
            var items: [SExpr] = []
            while true {
                skipSpace()
                guard pos < chars.count else { throw DecodeError.unexpectedEnd }
                if chars[pos] == ")" {
                    pos += 1
                    return .list(items)
                }
                items.append(try parse())
            }
        case "\"":
            return .string(try parseString())
        default:
            let start = pos
            while pos < chars.count, !chars[pos].isWhitespace, chars[pos] != "(", chars[pos] != ")" {
                pos += 1
            }
            let word = String(chars[start..<pos])
            if let i = Int(word) { return .int(i) }
            if let d = Double(word) { return .double(d) }
            return .symbol(word)
        }
    }

    private mutating func skipSpace() {
        while pos < chars.count, chars[pos].isWhitespace { pos += 1 }
    }

    private mutating func parseString() throws -> String {
        pos += 1  // opening quote
        var s = ""
        while pos < chars.count {
            let c = chars[pos]
            pos += 1
            switch c {
            case "\"": return s
            case "\\":
                guard pos < chars.count else { throw DecodeError.unexpectedEnd }
                let e = chars[pos]
                pos += 1
                s.append(e == "n" ? "\n" : e == "t" ? "\t" : e)
            default: s.append(c)
            }
        }
        throw DecodeError.unexpectedEnd
    }
}

enum DecodeError: Error {
    case unexpectedEnd
    case typeMismatch(expected: String, found: SExpr)
    case missingKey(String)
}
```

This mirrors the structure of the Go decoder's lexer and parser, but produces an `SExpr` tree rather than a stream of tokens, a simplification that costs a little memory and saves a lot of code.

Now the `Decoder` itself. A `Decoder` provides three kinds of containers, depending on the shape of the thing being decoded: a *keyed* container for structs and dictionaries, an *unkeyed* container for arrays, and a *single-value* container for atoms. Our S-expression format encodes structs as a list of `(field value)` pairs, so the keyed container looks up a field by scanning that list:

```swift
struct SExprDecoder: Decoder {
    let value: SExpr
    var codingPath: [any CodingKey] = []
    var userInfo: [CodingUserInfoKey: Any] = [:]

    func container<Key: CodingKey>(keyedBy type: Key.Type) throws -> KeyedDecodingContainer<Key> {
        guard case .list(let items) = value else {
            throw DecodeError.typeMismatch(expected: "list of fields", found: value)
        }
        var fields: [String: SExpr] = [:]
        for item in items {
            guard case .list(let pair) = item, pair.count == 2, case .symbol(let name) = pair[0] else {
                throw DecodeError.typeMismatch(expected: "(name value)", found: item)
            }
            fields[name] = pair[1]
        }
        return KeyedDecodingContainer(FieldContainer<Key>(fields: fields))
    }

    func unkeyedContainer() throws -> any UnkeyedDecodingContainer {
        guard case .list(let items) = value else {
            throw DecodeError.typeMismatch(expected: "list", found: value)
        }
        return ListContainer(items: items)
    }

    func singleValueContainer() throws -> any SingleValueDecodingContainer {
        AtomContainer(value: value)
    }
}
```

The three container types are long but repetitive, because each protocol has a separate `decode` method for every primitive type (`Bool`, `String`, `Double`, `Float`, and the ten integer types) plus a generic one for any other `Decodable`. The generic one is the key to the design: when asked to decode a nested `Decodable` type, it creates a new `SExprDecoder` for the corresponding subtree and calls `T(from:)`, which recursively does the same for *that* type's fields.

The single-value container does the real work of converting atoms. A private generic helper handles all ten integer types at once, using `FixedWidthInteger(exactly:)` to reject values that don't fit:

```swift
struct AtomContainer: SingleValueDecodingContainer {
    let value: SExpr
    var codingPath: [any CodingKey] = []

    func decodeNil() -> Bool { value == .symbol("nil") }

    func decode(_ type: Bool.Type) throws -> Bool {
        switch value {
        case .symbol("t"): return true
        case .symbol("nil"): return false
        default: throw DecodeError.typeMismatch(expected: "t or nil", found: value)
        }
    }

    func decode(_ type: String.Type) throws -> String {
        guard case .string(let s) = value else {
            throw DecodeError.typeMismatch(expected: "string", found: value)
        }
        return s
    }

    func decode(_ type: Double.Type) throws -> Double {
        switch value {
        case .double(let d): return d
        case .int(let i): return Double(i)
        default: throw DecodeError.typeMismatch(expected: "number", found: value)
        }
    }

    func decode(_ type: Float.Type) throws -> Float { Float(try decode(Double.self)) }
    func decode(_ type: Int.Type) throws -> Int { try integer() }
    func decode(_ type: Int8.Type) throws -> Int8 { try integer() }
    func decode(_ type: Int16.Type) throws -> Int16 { try integer() }
    func decode(_ type: Int32.Type) throws -> Int32 { try integer() }
    func decode(_ type: Int64.Type) throws -> Int64 { try integer() }
    func decode(_ type: UInt.Type) throws -> UInt { try integer() }
    func decode(_ type: UInt8.Type) throws -> UInt8 { try integer() }
    func decode(_ type: UInt16.Type) throws -> UInt16 { try integer() }
    func decode(_ type: UInt32.Type) throws -> UInt32 { try integer() }
    func decode(_ type: UInt64.Type) throws -> UInt64 { try integer() }

    func decode<T: Decodable>(_ type: T.Type) throws -> T {
        try T(from: SExprDecoder(value: value))  // recurse into the subtree
    }

    private func integer<I: FixedWidthInteger>() throws -> I {
        guard case .int(let i) = value, let n = I(exactly: i) else {
            throw DecodeError.typeMismatch(expected: "\(I.self)", found: value)
        }
        return n
    }
}
```

The keyed container finds the field for each key and hands its value to an `AtomContainer`, so its many `decode` methods are one line each:

```swift
struct FieldContainer<Key: CodingKey>: KeyedDecodingContainerProtocol {
    let fields: [String: SExpr]
    var codingPath: [any CodingKey] = []
    var allKeys: [Key] { fields.keys.compactMap(Key.init(stringValue:)) }

    func contains(_ key: Key) -> Bool { fields[key.stringValue] != nil }

    private func field(_ key: Key) throws -> SExpr {
        guard let v = fields[key.stringValue] else { throw DecodeError.missingKey(key.stringValue) }
        return v
    }

    private func atom(_ key: Key) throws -> AtomContainer {
        AtomContainer(value: try field(key))
    }

    func decodeNil(forKey key: Key) throws -> Bool {
        guard let v = fields[key.stringValue] else { return true }  // missing means nil
        return v == .symbol("nil")
    }

    func decode(_ type: Bool.Type, forKey key: Key) throws -> Bool { try atom(key).decode(type) }
    func decode(_ type: String.Type, forKey key: Key) throws -> String { try atom(key).decode(type) }
    func decode(_ type: Double.Type, forKey key: Key) throws -> Double { try atom(key).decode(type) }
    func decode(_ type: Float.Type, forKey key: Key) throws -> Float { try atom(key).decode(type) }
    func decode(_ type: Int.Type, forKey key: Key) throws -> Int { try atom(key).decode(type) }
    func decode(_ type: Int8.Type, forKey key: Key) throws -> Int8 { try atom(key).decode(type) }
    func decode(_ type: Int16.Type, forKey key: Key) throws -> Int16 { try atom(key).decode(type) }
    func decode(_ type: Int32.Type, forKey key: Key) throws -> Int32 { try atom(key).decode(type) }
    func decode(_ type: Int64.Type, forKey key: Key) throws -> Int64 { try atom(key).decode(type) }
    func decode(_ type: UInt.Type, forKey key: Key) throws -> UInt { try atom(key).decode(type) }
    func decode(_ type: UInt8.Type, forKey key: Key) throws -> UInt8 { try atom(key).decode(type) }
    func decode(_ type: UInt16.Type, forKey key: Key) throws -> UInt16 { try atom(key).decode(type) }
    func decode(_ type: UInt32.Type, forKey key: Key) throws -> UInt32 { try atom(key).decode(type) }
    func decode(_ type: UInt64.Type, forKey key: Key) throws -> UInt64 { try atom(key).decode(type) }
    func decode<T: Decodable>(_ type: T.Type, forKey key: Key) throws -> T { try atom(key).decode(type) }

    func nestedContainer<NestedKey: CodingKey>(
        keyedBy type: NestedKey.Type, forKey key: Key
    ) throws -> KeyedDecodingContainer<NestedKey> {
        try SExprDecoder(value: field(key)).container(keyedBy: type)
    }
    func nestedUnkeyedContainer(forKey key: Key) throws -> any UnkeyedDecodingContainer {
        try SExprDecoder(value: field(key)).unkeyedContainer()
    }
    func superDecoder() throws -> any Decoder { SExprDecoder(value: .list([])) }
    func superDecoder(forKey key: Key) throws -> any Decoder { try SExprDecoder(value: field(key)) }
}
```

The `decodeNil(forKey:)` method is how the synthesized code for optional properties asks whether a field is `nil`. Our format treats both a missing field and the symbol `nil` as absent.

Last, the unkeyed container decodes the elements of a list in order, keeping track of its position with `currentIndex`:

```swift
struct ListContainer: UnkeyedDecodingContainer {
    let items: [SExpr]
    var codingPath: [any CodingKey] = []
    var currentIndex = 0

    init(items: [SExpr]) { self.items = items }

    var count: Int? { items.count }
    var isAtEnd: Bool { currentIndex >= items.count }

    /// Returns a container for the next element and advances past it.
    private mutating func next() throws -> AtomContainer {
        guard !isAtEnd else { throw DecodeError.unexpectedEnd }
        defer { currentIndex += 1 }
        return AtomContainer(value: items[currentIndex])
    }

    mutating func decodeNil() throws -> Bool {
        guard !isAtEnd, items[currentIndex] == .symbol("nil") else { return false }
        currentIndex += 1
        return true
    }

    mutating func decode(_ type: Bool.Type) throws -> Bool { try next().decode(type) }
    mutating func decode(_ type: String.Type) throws -> String { try next().decode(type) }
    mutating func decode(_ type: Double.Type) throws -> Double { try next().decode(type) }
    mutating func decode(_ type: Float.Type) throws -> Float { try next().decode(type) }
    mutating func decode(_ type: Int.Type) throws -> Int { try next().decode(type) }
    mutating func decode(_ type: Int8.Type) throws -> Int8 { try next().decode(type) }
    mutating func decode(_ type: Int16.Type) throws -> Int16 { try next().decode(type) }
    mutating func decode(_ type: Int32.Type) throws -> Int32 { try next().decode(type) }
    mutating func decode(_ type: Int64.Type) throws -> Int64 { try next().decode(type) }
    mutating func decode(_ type: UInt.Type) throws -> UInt { try next().decode(type) }
    mutating func decode(_ type: UInt8.Type) throws -> UInt8 { try next().decode(type) }
    mutating func decode(_ type: UInt16.Type) throws -> UInt16 { try next().decode(type) }
    mutating func decode(_ type: UInt32.Type) throws -> UInt32 { try next().decode(type) }
    mutating func decode(_ type: UInt64.Type) throws -> UInt64 { try next().decode(type) }
    mutating func decode<T: Decodable>(_ type: T.Type) throws -> T { try next().decode(type) }

    mutating func nestedContainer<NestedKey: CodingKey>(
        keyedBy type: NestedKey.Type
    ) throws -> KeyedDecodingContainer<NestedKey> {
        try SExprDecoder(value: next().value).container(keyedBy: type)
    }
    mutating func nestedUnkeyedContainer() throws -> any UnkeyedDecodingContainer {
        try SExprDecoder(value: next().value).unkeyedContainer()
    }
    mutating func superDecoder() throws -> any Decoder {
        try SExprDecoder(value: next().value)
    }
}
```

Finally, the public entry point, the analogue of `JSONDecoder.decode`:

```swift
func unmarshal<T: Decodable>(_ type: T.Type, from text: String) throws -> T {
    var parser = SExprParser(text)
    return try T(from: SExprDecoder(value: try parser.parse()))
}
```

And with that, we can decode *any* `Decodable` type from an S-expression, including types the decoder's author has never seen:

```swift
struct Movie: Codable {
    var title: String
    var year: Int
    var color: Bool
    var oscars: [String]
    var sequel: String?
}

let text = """
    ((title "Dr. Strangelove")
     (year 1964)
     (color nil)
     (oscars ("Best Actor (Nomin.)" "Best Director (Nomin.)"))
     (sequel nil))
    """
let movie = try unmarshal(Movie.self, from: text)
print(movie.title, movie.year, movie.oscars.count)  // "Dr. Strangelove 1964 2"
```

Compare the two halves of this section. In Go, the same job takes about 200 lines of reflection that sets fields by name through `reflect.Value`. Here, the decoder *itself* never looks at `Movie`; it merely answers questions the compiler-generated `init(from:)` asks: "give me the `Int` for the key `year`." Type-checking the result is automatic, because `Movie` asked for an `Int`, and what the container returns is an `Int` or an error.

**Exercise 12.8:** The three containers repeat the same fifteen `decode` methods. The protocols require each one, but their bodies can still be shared. How far can you reduce the repetition? Then add `Int128` and `UInt128`, which the protocols support in Swift 6.

**Exercise 12.9:** Make the keyed container track the `codingPath`, and use it to produce error messages that tell the user which field of which value was wrong, like `movie.oscars[1]: expected string, found 42`.

**Exercise 12.10:** Write the encoder half, an `Encoder` type that emits the S-expression format, so that `Codable` types round-trip. Test with randomized values (Section 11.2.6).

## 12.7. Customizing Keys with `CodingKeys`

Go's reflection reads *struct field tags*, the strings after a field's type, like `` `json:"color,omitempty"` ``. They're a way of attaching metadata to a field that run-time reflection can find. Swift has no tags and no field metadata, and doesn't need them: the same information is carried by an ordinary nested type, `CodingKeys`, that the compiler-generated code uses.

We saw it in Section 4.5. The enum's cases are the stored properties that participate in coding, and its raw values are the names used in the external format:

```swift
struct Movie: Codable {
    var title: String
    var year: Int
    var color: Bool
    var oscars: [String]

    enum CodingKeys: String, CodingKey {
        case title
        case year = "released"
        case color
        case oscars = "awards"
    }
}
```

Any decoder or encoder, whether for JSON, property lists, or our S-expressions, then uses `released` and `awards` as the field names. Because the keys are real Swift code, the compiler checks them: a case that doesn't correspond to a property is an error, and so is a property that isn't covered by a case and has no default value.

For *systematic* renaming, rather than choosing a name for each property, the encoders offer *strategies*: `JSONEncoder.keyEncodingStrategy = .convertToSnakeCase` converts every property name from `camelCase` to `snake_case`, and `JSONDecoder.keyDecodingStrategy = .convertFromSnakeCase` does the inverse, as we used in Section 4.5.

The finest level of control is a hand-written `encode(to:)` and `init(from:)`. For example, a type that should be encoded as a single string, like a color `"#ff8800"` or a version `"1.2.3"`, can implement them using a single-value container:

```swift
struct Version: Codable, Equatable {
    var major, minor, patch: Int

    init(major: Int, minor: Int, patch: Int) {
        (self.major, self.minor, self.patch) = (major, minor, patch)
    }

    init(from decoder: any Decoder) throws {
        let text = try decoder.singleValueContainer().decode(String.self)
        let parts = text.split(separator: ".").compactMap { Int($0) }
        guard parts.count == 3 else {
            throw DecodingError.dataCorrupted(
                .init(codingPath: decoder.codingPath, debugDescription: "bad version \(text)"))
        }
        self.init(major: parts[0], minor: parts[1], patch: parts[2])
    }

    func encode(to encoder: any Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode("\(major).\(minor).\(patch)")
    }
}
```

This is the same role that Go's `json.Marshaler` and `UnmarshalJSON` methods play: an escape hatch from the default encoding for types that need one.

### 12.7.1. Property Wrappers as Field Annotations

One other piece of Swift deserves a mention here, since it's the closest analogue to a Go struct tag that *attaches behavior* to a field: the *property wrapper*. We met it in Section 2.3.2, where `@Option` and `@Flag` described how to parse each field of a command. A property wrapper is a type that wraps the storage of a property and can run code when it's read or written, and also exposes a *projected value* through the `$` prefix. Swift libraries use them as annotations that other code can discover by reflection, which is how `ArgumentParser` finds the options of a command: it reflects over the command with `Mirror`, and finds children that are instances of its wrapper types.

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    let range: ClosedRange<Value>

    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }

    var wrappedValue: Value {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

struct Volume {
    @Clamped(0...11) var level = 5
}

var v = Volume()
v.level = 20
print(v.level)  // "11"
```

Here the annotation `@Clamped(0...11)` is declarative metadata *and* behavior: every write to `level` is clamped, with no tag parsing and no run-time reflection. That's the Swift style in general: where Go attaches a string to a field and interprets it later, Swift attaches a *type* to a field and lets the compiler check and apply it.

## 12.8. Macros: Reflection at Compile Time

Swift 5.9 added a feature that covers much of the ground that reflection covers in other languages: *macros*. A macro is a function that runs at compile time. It receives the syntax tree of the code it's attached to and produces new Swift code, which the compiler then compiles along with the rest. Because a macro sees the *declaration* of a type, including its property names, types, and attributes, it can do what Go programs do with run-time reflection (enumerate the fields of a struct, generate serialization code, build lookup tables) once, at build time, producing ordinary Swift with no run-time cost and full type checking.

Macros come in two broad kinds. *Freestanding* macros stand alone, are written with a `#` prefix, and produce an expression or declarations. We have been using one throughout Chapter 11: `#expect(a == b)` is a macro that inspects the *syntax* of its argument to build a failure message showing both operands. *Attached* macros are written with `@`, are attached to a declaration, and can add members, conformances, peers, or accessors to it. `@Test` and `@Suite` are attached macros, as is `@Observable`, which rewrites the stored properties of a class to track reads and writes.

Using a macro is simple. Consider a package that provides an `@AllFields` macro that adds, to any struct it's attached to, a static table of its field names:

```swift
@AllFields
struct Person {
    var name: String
    var age: Int
}

print(Person.fieldNames)  // "["name", "age"]"
```

The compiler expands the macro, which shows the code that it writes for us:

```swift
struct Person {
    var name: String
    var age: Int

    static var fieldNames: [String] { ["name", "age"] }
}
```

The macro implementation is a Swift program that uses the `swift-syntax` package to examine and construct syntax trees. It lives in its own target, which is compiled for the host (the machine doing the build) and run by the compiler as a plug-in in a sandbox:

```swift
// Package.swift: a macro target, plus a library that exposes it
.macro(
    name: "AllFieldsMacros",
    dependencies: [
        .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
        .product(name: "SwiftCompilerPlugin", package: "swift-syntax"),
    ]
),
.target(name: "AllFields", dependencies: ["AllFieldsMacros"]),
```

```swift
// Sources/AllFieldsMacros/AllFieldsMacro.swift
import SwiftSyntax
import SwiftSyntaxMacros

public struct AllFieldsMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf decl: some DeclGroupSyntax,
        conformingTo protocols: [TypeSyntax],
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        // Collect the name of each stored property declared with var or let.
        let names = decl.memberBlock.members.compactMap { member -> String? in
            guard let v = member.decl.as(VariableDeclSyntax.self),
                  let binding = v.bindings.first,
                  binding.accessorBlock == nil,  // stored, not computed
                  let id = binding.pattern.as(IdentifierPatternSyntax.self)
            else { return nil }
            return id.identifier.text
        }
        let list = names.map { "\"\($0)\"" }.joined(separator: ", ")
        return ["static var fieldNames: [String] { [\(list)] }"]
    }
}

@main
struct AllFieldsPlugin: CompilerPlugin {
    let providingMacros: [any Macro.Type] = [AllFieldsMacro.self]
}
```

```swift
// Sources/AllFields/AllFields.swift
@attached(member, names: named(fieldNames))
public macro AllFields() = #externalMacro(module: "AllFieldsMacros", type: "AllFieldsMacro")
```

Macros are the Swift answer to the question "how can I write code that depends on the structure of my types?" With a macro, the `setters` table of Section 12.5.1 could be generated from the declaration of `Config`, and the `Codable` conformance itself, which was originally compiler magic, could be written as a macro. A few things set macros apart from reflection:

- They produce *source code*, which you can read, using the *Expand Macro* command in Xcode or `-Xfrontend -dump-macro-expansions` on the command line. There's no hidden run-time behavior.
- The result is *type-checked* like any other code. A macro can't produce something that doesn't compile without a diagnostic pointing at the problem.
- They can emit *diagnostics* of their own. A macro that requires its argument to be a struct, for example, can report an error at the right place in the source.
- They're *hygienic*: a macro can't capture or accidentally interfere with names in the code around it, other than those it's explicitly designed to touch.
- Their *cost* is build time, mostly the compile time of the `swift-syntax` library that macro implementations depend on, though prebuilt versions of it have reduced that. Run-time cost is zero.

On the other hand, they're harder to write than a reflective function: you work with syntax trees rather than values, and you can only act on what's visible in the declaration the macro is attached to, not on the types it refers to, which may be declared elsewhere. A macro can't ask "what are the properties of the type of this field?" Run-time reflection can, since it sees every value's actual type.

A good guideline is that a macro is the right choice when the information is *static*: known at compile time from the declaration. Reflection is for the information that's truly dynamic: the contents of an `Any` that arrived from somewhere else.

**Exercise 12.11:** Write a macro `@Describing` that adds a `description` property to a struct, listing its property names and values, like `Person(name: "Ada", age: 36)`. Compare the result with the output of `print` for a struct with no custom `description`.

**Exercise 12.12:** Use the macro testing support in `swift-syntax` (`assertMacroExpansion`) to write a test for `AllFields` with a struct that has computed properties, `let` properties, and properties with default values.

## 12.9. A Word of Caution

Reflection is a powerful and expressive tool, but it should be used with care, for three reasons.

The first reason is that reflection-based code can be fragile. For every mistake that would cause a compiler to report a type error, there is a corresponding way to misuse reflection, but whereas the compiler reports the mistake at build time, a reflective mistake is reported as a failed cast, a missing key, or a wrong result at run time, possibly long after the program was written, possibly in production. In Swift, some of the mistakes are even quieter: `Mirror`'s output isn't part of any type's API, and it can change when a type's implementation does. Code that formats `Date` by reflection, for instance, could break when the internal layout of `Date` changes.

Reflection-based code should therefore be restricted to the boundaries of a system: the code that reads or writes an external format, or prints values for debugging. Everywhere else, use static types. Where reflection is genuinely necessary, put it behind a small, well-tested function with a clear contract, and use `Codable`, key paths, or macros instead whenever the structure is knowable at compile time.

The second reason to avoid reflection is that, even when types serve as a form of documentation and are checked by the compiler, reflective code can't be: functions that take an `Any` and use `Mirror` have no signature that tells you what they actually support. Define the contract in documentation, and fail loudly, with a precise error message, on unsupported input.

The third reason is that reflection-based functions can be one or two orders of magnitude slower than code specialized for a particular type. Every `Mirror(reflecting:)` allocates, every child is boxed into an `Any`, and every cast walks type metadata. In a test that runs a few times, it doesn't matter. In a hot path of a server, it can. This is the performance side of the advantage of `Codable` over `Mirror`, and the main reason Swift's standard serialization is built on the former. If a function is on a critical path, measure it (Chapter 11) and consider a macro or hand-written conformance instead.

All of that said, reflection is one of the means by which Swift programs examine unknown values, and, like the unsafe operations of the next chapter, a skilled programmer uses it when appropriate, and as sparingly as possible.
