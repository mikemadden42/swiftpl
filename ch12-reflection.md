# 12. Reflection

Most of the time, a Swift program knows the types of its values. The compiler checks every operation against those types and generates code specialized for them, and that's a large part of why Swift programs are both safe and fast. Occasionally, though, a program has to work with values whose types it doesn't know in advance: a debugger showing the contents of a variable, a logger recording arbitrary values, a serializer writing any type to disk. For that, a language needs *reflection*, the ability of a program to examine the structure of its own values while it runs.

Swift's reflection is intentionally limited. It can *look*: given any value, the standard library's `Mirror` type reports what kind of thing it is and what it contains. But it can't, in general, *touch*: there's no way to set a property by name, enumerate a type's methods, or call a method chosen at run time. That's a deliberate trade. The less a program can do behind the compiler's back, the more the compiler can check and optimize. For the jobs that other languages do with full run-time reflection, Swift offers compile-time machinery instead: synthesized protocol conformances such as `Codable`, key paths, generics, and macros.

This chapter covers both sides. We'll use `Mirror` to build a value inspector and a serializer, see why it can't build a *deserializer*, and then build one anyway with `Decodable`. Along the way we'll look at key paths as a typed alternative to setting properties by name, and at macros, which let a library generate code from a type's declaration at build time.

## 12.1. Why Reflection?

Imagine you're writing a structured logging library. Callers hand it values of their own types, perhaps a `Request`, an `Order`, or an `[Int: String]`, and the library should record each value as a tree of named fields, so that a log viewer can search and filter them.

How can the library possibly do that? It can't know the caller's types. It could require every logged type to conform to some protocol of its own, but that pushes work onto every user and fails for types from third-party modules. It could try casting to the types it knows:

```swift
func record(_ value: Any) -> String {
    switch value {
    case let x as Int: return String(x)
    case let x as Bool: return x ? "true" : "false"
    case let x as String: return x.debugDescription
    // ...
    default:
        // a struct? an enum? an array of who knows what?
        fatalError("can't record \(type(of: value))")
    }
}
```

But that list can never end. Arrays and dictionaries can hold any element type, and the space of user-defined structs and enums is unbounded. What the library needs is a way to ask a value: *what are you made of?*

The standard library faces the same problem every time `print` is given a value whose type has no `description`, and it answers in the same way that we will. First it checks for protocols the type may have adopted to describe itself (`CustomStringConvertible` and `CustomDebugStringConvertible`, as Section 7.12 showed). If there are none, it falls back on `Mirror`.

## 12.2. `Mirror`

A `Mirror` describes the structure of a single value. Create one with `Mirror(reflecting:)` and then ask it about the value:

- `subjectType` is the value's dynamic type.
- `displayStyle` says what kind of value it is: `.struct`, `.class`, `.enum`, `.tuple`, `.optional`, `.collection`, `.dictionary`, `.set`, or `.foreignReference`. It's `nil` for values the standard library treats as indivisible, such as numbers and strings.
- `children` is a collection of `(label: String?, value: Any)` pairs, one for each stored property, element, or associated value.
- `superclassMirror` reflects the superclass part of a class instance.

For example:

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

Every child's value comes back typed as `Any`. To go deeper, you cast it or make another mirror of it, so code built on `Mirror` is almost always recursive.

The shape of `children` depends on the display style:

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

Collections and sets have one unlabeled child per element. A tuple's children carry its labels, if it has any. An enum value has at most one child, labeled with the case name and holding the associated value (or a tuple of them); a case with no associated values has no children at all. A dictionary's children are `(key:value:)` pairs. An optional has one child if it holds a value and none if it's `nil`.

It's just as important to know what a mirror *can't* tell you. It shows stored state only, so it omits computed properties, static members, initializers, and methods. It's read-only: there's no way to change a value through it. And it shows what the type's author allows. A type can supply its own mirror by conforming to `CustomReflectable`, to hide sensitive or irrelevant fields or to present a simpler logical structure:

```swift
struct Account: CustomReflectable {
    var owner: String
    private var pin: String

    var customMirror: Mirror {
        Mirror(self, children: ["owner": owner])  // doesn't reveal pin
    }
}
```

Without such a customization, a mirror *does* show private stored properties. Treat that as a debugging aid, not an API: private properties are private precisely so that they can change, and code that depends on them will break when they do.

## 12.3. `inspect`, a Recursive Value Printer

Our first reflective program prints every leaf of a value's structure on its own line, together with the path that leads to it, like `order.items[2].price`. The standard library's `dump` function prints a similar tree. Writing our own shows how such a traversal works.

```swift
// swiftpl/ch12/inspect
func inspect(_ name: String, _ value: Any) {
    print("inspect \(name) (\(type(of: value))):")
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

`walk` asks each value what kind it is and handles each kind by extending the path appropriately: a property name for structs, classes, and tuples; a subscript for arrays and sets; the formatted key for dictionaries; a `!` for an unwrapped optional; the case name for an enum with a payload. Anything with no display style (a number, a string, a Boolean) is a leaf, and is printed. `format` quotes strings and characters, so that a leaf containing spaces or quotation marks can't be misread.

Here it is applied to a recipe:

```swift
struct Recipe {
    var title: String
    var servings: Int
    var vegetarian: Bool
    var steps: [String]
    var ingredients: [String: String]  // ingredient: quantity
    var doubled: Int { servings * 2 }  // computed: not shown
}

let shakshuka = Recipe(
    title: "Shakshuka", servings: 4, vegetarian: true,
    steps: ["Soften onions and peppers", "Add tomatoes and spices", "Poach the eggs"],
    ingredients: ["eggs": "6", "tomatoes": "800 g", "onion": "1"])
inspect("shakshuka", shakshuka)
```

```
inspect shakshuka (Recipe):
shakshuka.title = "Shakshuka"
shakshuka.servings = 4
shakshuka.vegetarian = true
shakshuka.steps[0] = "Soften onions and peppers"
shakshuka.steps[1] = "Add tomatoes and spices"
shakshuka.steps[2] = "Poach the eggs"
shakshuka.ingredients["eggs"] = "6"
shakshuka.ingredients["tomatoes"] = "800 g"
shakshuka.ingredients["onion"] = "1"
```

The dictionary lines may come out in a different order on each run. The computed property `doubled` is absent, as expected.

`inspect` works on standard library types too: `inspect("r", 1...3)` shows a range's `lowerBound` and `upperBound`. But for many library types, what you'll see are private implementation details. The stored properties of `Date` or `URL` aren't part of their APIs and may differ between platforms and releases. A program's *logic* should never depend on them.

### 12.3.1. Cycles

Struct and enum values can't contain themselves, so any value built from them is a tree, and `walk` will eventually reach the leaves. Class instances are different. Two objects can refer to each other, directly or through a chain, and following such a cycle would recurse until the stack overflowed. `dump` guards against this by remembering which objects it has already printed. We can do the same with `ObjectIdentifier`, a `Hashable` value that identifies a class instance:

```swift
private func walk(_ path: String, _ value: Any, _ visited: inout Set<ObjectIdentifier>) {
    let m = Mirror(reflecting: value)
    if m.displayStyle == .class {
        let id = ObjectIdentifier(value as AnyObject)
        guard visited.insert(id).inserted else {
            print("\(path) = (already shown)")
            return
        }
    }
    // ...as before, passing visited along the recursive calls...
}
```

Only class instances need tracking, since only they have identity.

**Exercise 12.1:** Make `inspect` include the fields that a class instance inherits from its superclasses, using `superclassMirror`.

**Exercise 12.2:** Add cycle detection to `inspect` and test it on a doubly linked list.

**Exercise 12.3:** Make `inspect` emit JSON instead of path lines, turning non-string dictionary keys into strings. Compare its output with `JSONEncoder` on a `Codable` type. Where do they differ, and why?

## 12.4. Example: Encoding S-Expressions

With a little more structure in its output, `inspect` would be a serializer. Let's build one that writes any Swift value as an *S-expression*, the parenthesized notation of Lisp. S-expressions have only two kinds of things, atoms and lists, which makes them about the simplest structured format there is and a good vehicle for the technique. We'll encode Swift values like this:

```
42                                   integer
3.5                                  floating-point number
"hello"                              string, with Swift-style escapes
t / nil                              true / false
(1 2 3)                              array or set
((key1 value1) (key2 value2))        dictionary
((title "Soup") (servings 2))        struct or class, as (field value) pairs
(case payload)                       enum case with an associated value
```

The encoder accepts an `Any`, returns a `String`, and throws for anything it doesn't know how to represent:

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

The encoder is `walk` with output rules. Atoms are handled first, by casting. Here the casts are to *protocols*: `any BinaryInteger` matches all ten integer types and `any BinaryFloatingPoint` every floating-point type, so one case covers each family (the same use of protocol casts as in Section 7.12). Everything else goes through a mirror. The `indent` parameter tracks the column where the current list began, so that elements after the first line up under it. The arithmetic for a struct field, `indent + name.count + 3`, accounts for the two parentheses and the space in `(name `.

Encoding our recipe, with one more field added, shows the layout:

```swift
struct Recipe {
    var title: String
    var servings: Int
    var vegetarian: Bool
    var steps: [String]
    var ingredients: [String: String]
    var source: String?
}

let shakshuka = Recipe(
    title: "Shakshuka", servings: 4, vegetarian: true,
    steps: ["Soften onions and peppers", "Add tomatoes and spices", "Poach the eggs"],
    ingredients: ["eggs": "6", "tomatoes": "800 g", "onion": "1"],
    source: nil)

print(try marshal(shakshuka))
```

```
((title "Shakshuka")
 (servings 4)
 (vegetarian t)
 (steps ("Soften onions and peppers"
         "Add tomatoes and spices"
         "Poach the eggs"))
 (ingredients (("eggs" "6")
               ("tomatoes" "800 g")
               ("onion" "1")))
 (source nil))
```

(As with `inspect`, the dictionary entries may come out in any order. Exercise 12.5 fixes that.)

### 12.4.1. The Compile-Time Alternative: `Encoder`

`marshal` works, and it's a clear illustration of reflection. It's not, however, how Swift libraries normally serialize data. `JSONEncoder` and `PropertyListEncoder` don't use `Mirror` at all.

Instead, they rely on code the compiler writes. When a type declares that it conforms to `Encodable`, the compiler synthesizes an `encode(to:)` method that walks the type's stored properties *at compile time*, calling `encode(_:forKey:)` on an encoding container for each one, with its static type. An encoder (a type conforming to the `Encoder` protocol) receives those calls and turns them into its format. The encoder never needs to discover a type's structure, because the type describes itself.

That arrangement has real advantages over reflection:

- **It's faster.** There's no `Any` boxing, no metadata walking, and no casts; the generated code is what you'd write by hand.
- **It's typed.** What gets encoded follows the declared types, not whatever happens to be inside an `Any`.
- **It's customizable.** A type can rename keys with `CodingKeys`, omit properties, or take over entirely with its own `encode(to:)`, and the customization applies to every format.
- **It runs in both directions.** `Decodable` lets a library *create* values of types it has never seen, which, as the next sections show, `Mirror` cannot do.

The cost is that conformance must be declared. A type that isn't `Codable` can't be encoded that way, while `Mirror` can look at anything. Both tools have their place; for a serialization format of your own, an `Encoder` is the better design (Exercise 12.6).

**Exercise 12.4:** Extend `marshal` to encode tuples, as lists of their elements, and test it on optionals, nested collections, and enums with and without payloads.

**Exercise 12.5:** Sort dictionary entries by key so that the output is deterministic. With only an `Any` in hand, how can you tell whether the keys are `Comparable`?

**Exercise 12.6:** Write a type conforming to `Encoder` that produces S-expressions, so that any `Encodable` value can be serialized without reflection. Compare its output with `marshal` on a `Codable` version of `Recipe`.

**Exercise 12.7:** Make a JSON version of `marshal`. For the types of Section 4.5, does it agree with `JSONEncoder`? Identify a case where reflection produces the wrong answer and `Codable` the right one.

## 12.5. Setting Values with Key Paths

Reading arbitrary properties is half of what full reflection offers; the other half is *writing* them, chosen by name at run time. `Mirror` can't do that. When a Swift program needs to read or write a property chosen at run time, it uses a *key path* (Section 6.4) instead.

A key path is a reference to a property that isn't yet tied to an instance. It's checked when it's written, so it can't name a property that doesn't exist, and its type records both the root type and the property type: `KeyPath<Root, Value>` is read-only, `WritableKeyPath<Root, Value>` can write through a mutable value, and `ReferenceWritableKeyPath<Root, Value>` can write through a class reference.

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

Because a key path is an ordinary value, it can be passed around and stored, which lets one function operate on whichever property the caller chooses:

```swift
func setAll<T, V>(_ items: inout [T], _ keyPath: WritableKeyPath<T, V>, to value: V) {
    for i in items.indices {
        items[i][keyPath: keyPath] = value
    }
}

var people = [Person(name: "Ada", age: 36), Person(name: "Alan", age: 41)]
setAll(&people, \.age, to: 0)
```

To choose a property by *name*, put key paths in a dictionary. The set of accessible properties is fixed when the table is written, but which one is used can be decided at run time:

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

`PartialKeyPath<Root>` erases the property type so that paths to properties of different types can share a dictionary; reading through one yields `Any`. (`AnyKeyPath` erases the root type as well.) Writing requires casting back to a `WritableKeyPath` of the right types.

Key paths are everywhere in modern Swift APIs, often in places that look dynamic at first glance: `\.name` passed where a closure is expected, `sorted(using: KeyPathComparator(\.age))`, SwiftUI bindings, the change tracking of `@Observable`, and SwiftData predicates. In each case, the compiler has checked the path.

### 12.5.1. Setting Values by Name from Text

A common task is filling in a configuration struct from text: a config file, environment variables, command-line `key=value` pairs. Each value arrives as a string with a name attached and has to be converted to the right type and stored in the right property. A table of key paths, plus a little generic code, does the job with full type checking:

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

`Setter`'s generic initializer accepts a key path to a property of any type `V` that conforms to `LosslessStringConvertible`, the standard protocol for types that can be created from their own string form, which `Int`, `Double`, `Bool`, `String`, and many others adopt. The initializer wraps the key path and the conversion in a closure, so that setters for properties of different types can live in one dictionary. Invalid text is rejected at run time; an invalid property name or a type mismatch is rejected at compile time. The one chore left is listing each property once, which a macro (Section 12.8) can take over.

## 12.6. Example: Decoding S-Expressions

Writing values out was a job for `Mirror`. Reading them back is not. To decode, a program must *create* a value of a type chosen by its caller and fill in its properties, and a mirror has no way to create anything or to write to anything. Run-time reflection is a dead end here.

`Decodable` takes the problem from the other side. Just as the compiler synthesizes `encode(to:)`, it synthesizes an initializer, `init(from:)`, that knows how to build the type: it asks a *decoder* for a container, then asks the container for each stored property by key and type, as in `container.decode(Int.self, forKey: .servings)`. A decoder for a new format only has to answer those questions. It never needs to know what type it's building.

Our decoder starts by parsing text into a tree:

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

The parser is a straightforward recursive descent. A `(` starts a list, which runs until the matching `)`; a `"` starts a string; anything else is a word, which becomes an integer, a floating-point number, or a symbol, whichever parses first. Building a complete tree before decoding costs some memory but makes the decoder much simpler, since it can look at any part of the input at any time.

The `Decoder` protocol asks for three kinds of *container*, one for each shape of data: a *keyed* container for things with named fields, an *unkeyed* container for sequences, and a *single-value* container for atoms. In our format, a struct is a list of `(name value)` pairs, so making a keyed container means turning that list into a dictionary:

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

The container protocols are long, because they have a separate `decode` method for each primitive type (`Bool`, `String`, `Double`, `Float`, and the ten integer types) as well as a generic `decode` for every other `Decodable` type. Most of that length is repetition. The generic method is where the design comes together: to decode a nested value, it wraps the corresponding subtree in a new `SExprDecoder` and calls `T(from:)`, which asks its own questions of the subtree, and so on down.

The single-value container converts atoms to Swift values. A private generic helper serves all the integer types at once, using `init(exactly:)` so that a number too large for the requested type is reported as an error rather than silently truncated:

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

The keyed container looks up the field for each key and delegates the conversion to an `AtomContainer`:

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

`decodeNil(forKey:)` is how the synthesized code for an optional property asks whether its value is absent. This format treats a missing field and the symbol `nil` the same way.

The unkeyed container walks a list in order, with `currentIndex` marking its position:

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

Finally, a top-level function plays the role that `JSONDecoder.decode(_:from:)` plays for JSON:

```swift
func unmarshal<T: Decodable>(_ type: T.Type, from text: String) throws -> T {
    var parser = SExprParser(text)
    return try T(from: SExprDecoder(value: try parser.parse()))
}
```

Now any `Decodable` type can be read from an S-expression, though the decoder was written without knowledge of any of them:

```swift
struct Recipe: Codable {
    var title: String
    var servings: Int
    var vegetarian: Bool
    var steps: [String]
    var source: String?
}

let text = """
    ((title "Shakshuka")
     (servings 4)
     (vegetarian t)
     (steps ("Soften onions and peppers" "Add tomatoes and spices" "Poach the eggs"))
     (source nil))
    """
let recipe = try unmarshal(Recipe.self, from: text)
print(recipe.title, recipe.servings, recipe.steps.count)  // "Shakshuka 4 3"
```

Note the division of labor. The decoder knows the *format*: how to find a field, what `t` means, how strings are quoted. The compiler-generated `init(from:)` knows the *type*: which fields exist and what type each must be. Neither knows the other's business, and nothing in the program looks at a type's layout at run time. The type checking that a reflective decoder would have to do by hand happens automatically, because `Recipe` asks for an `Int` and receives either an `Int` or an error.

**Exercise 12.8:** The three containers repeat the same fifteen `decode` methods. The protocols require each one, but their bodies can still be shared. How far can you reduce the repetition? Then add `Int128` and `UInt128`, which the protocols support in Swift 6.

**Exercise 12.9:** Make the containers maintain `codingPath`, and use it to produce error messages that pinpoint the problem, such as `recipe.steps[1]: expected string, found 42`.

**Exercise 12.10:** Write the matching `Encoder` (if you haven't already done Exercise 12.6) and test that `Codable` values survive a round trip, using randomized values as in Section 11.2.6.

## 12.7. Customizing Keys with `CodingKeys`

Serialization formats often use names that differ from a program's property names: snake case instead of camel case, abbreviations, or names chosen long ago that the code has since outgrown. Some languages attach such information to fields as strings that run-time reflection reads back. Swift does it with ordinary declarations that the compiler-generated coding methods use, chiefly a nested enum named `CodingKeys`.

We met `CodingKeys` in Section 4.5. Its cases list the properties that take part in coding, and its raw values give their external names:

```swift
struct Recipe: Codable {
    var title: String
    var servings: Int
    var vegetarian: Bool
    var steps: [String]

    enum CodingKeys: String, CodingKey {
        case title
        case servings = "serves"
        case vegetarian = "veg"
        case steps = "method"
    }
}
```

Every encoder and decoder, whether it produces JSON, property lists, or the S-expressions of this chapter, now uses `serves`, `veg`, and `method`. Since `CodingKeys` is code, mistakes are caught at compile time: a case that matches no property is an error, and so is a property with neither a case nor a default value.

For renaming *every* key by a rule, encoders and decoders provide strategies instead: `JSONEncoder.keyEncodingStrategy = .convertToSnakeCase` and its decoding counterpart (Section 4.5).

When a type needs a representation that has little to do with its properties, it can write `encode(to:)` and `init(from:)` itself. A version number, for example, is more naturally a single string like `"1.2.3"` than an object with three fields:

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

Because these methods are written against the `Encoder` and `Decoder` protocols, the custom representation applies in every format at once.

### 12.7.1. Property Wrappers as Field Annotations

Sometimes metadata about a field should come with *behavior*: this value must stay within a range, this one should be trimmed of whitespace, this one is parsed from a command-line flag. Swift's tool for that is the *property wrapper* (we used ArgumentParser's `@Option` and `@Flag` in Section 2.3.2). A property wrapper is a type that owns a property's storage and mediates every read and write:

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

The annotation `@Clamped(0...11)` is both a declaration of intent and its implementation; there's nothing to look up or interpret at run time. Property wrappers and reflection also cooperate: a wrapped property's storage appears in a mirror as an instance of the wrapper type, which is how ArgumentParser discovers a command's options. It mirrors the command struct and looks for children whose values are its own wrapper types.

## 12.8. Macros: Reflection at Compile Time

Many uses of reflection share a pattern: some code needs to know the structure of a type (its property names, their types, their attributes), and it learns that structure at run time because there's no other time to learn it. Swift 5.9 provided another time: compile time. A *macro* is a program that the compiler runs during the build. It receives the syntax of the code it's applied to and returns new Swift code to be compiled alongside it. A macro attached to a type declaration can read the declaration's properties and generate lookup tables, conformances, or serialization code, the same results reflection would compute, but produced once, checked by the compiler, and free at run time.

Macros are either *freestanding* or *attached*. A freestanding macro is invoked with `#` and expands into an expression or declarations; `#expect` from Chapter 11 is one, and it's how a failed expectation can show the values of both sides of a comparison: the macro rewrote the comparison to capture them. An attached macro is written with `@` before a declaration and can add members, conformances, accessors, or neighboring declarations to it; `@Test`, `@Suite`, and `@Observable` are all attached macros.

Here's what using a simple attached macro looks like. `@AllFields` gives a struct a static list of its stored property names:

```swift
@AllFields
struct Person {
    var name: String
    var age: Int
}

print(Person.fieldNames)  // "["name", "age"]"
```

Expanding the macro shows the code it writes:

```swift
struct Person {
    var name: String
    var age: Int

    static var fieldNames: [String] { ["name", "age"] }
}
```

The macro's implementation is itself a Swift program, which uses the `swift-syntax` package to read and build syntax trees. A package that provides a macro has two targets, one for the implementation and one for the declaration that clients import:

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

The `.macro` target is built for the machine running the compiler, not for the program's target platform, and the compiler runs it in a sandbox. The `#externalMacro` declaration in the library target connects the name `AllFields`, which clients use, to the type that implements it.

Macros cover a lot of reflection's territory. The `setters` table of Section 12.5.1 could be generated by a macro from the declaration of `Config`. So could a `CustomStringConvertible` conformance, a memberwise copy-with-changes method, or the `Codable` conformance itself, which predates macros but is the same kind of thing. Compared with reflection, macros have distinct strengths:

- **You can read the result.** Xcode's *Expand Macro* command, or `-Xfrontend -dump-macro-expansions` on the command line, shows the generated code. Nothing happens at run time that isn't in source you could have written.
- **The result is type-checked.** Generated code goes through the compiler like any other, and problems are reported at a source location.
- **Macros can diagnose misuse.** A macro can reject declarations it doesn't support with an error pointing at the offending line.
- **Expansion is hygienic.** Names a macro introduces don't collide with names at the use site unless the macro means them to.
- **The cost is paid at build time.** Implementations depend on `swift-syntax`, which adds to compile time (less so with prebuilt copies), but expanded code costs nothing extra to run.

Macros have limits, though. They work on *syntax*, so a macro attached to a struct sees the text of that struct's declaration and nothing else. It can't ask what properties the type of one of the fields has, if that type is declared elsewhere, because it has no access to the type checker's knowledge. Reflection, by contrast, sees whatever value actually turns up at run time.

That suggests the rule of thumb. If what you need to know is in a declaration you control, generate code from it with a macro, or let the compiler synthesize a conformance. If it only exists at run time, as with values of unknown type arriving in an `Any`, reflect.

**Exercise 12.11:** Write a macro `@Describing` that adds a `description` property listing a struct's property names and values. Compare its output with what `print` shows for a struct with no description.

**Exercise 12.12:** Test `AllFields` with `assertMacroExpansion` from `swift-syntax`, using a struct that has computed properties, `let` properties, and properties with default values.

## 12.9. A Word of Caution

Reflection lets a program step outside its own type system, and most of the cautions about it follow from that.

**Reflection sees representation, not meaning.** A mirror reports how a value happens to be stored, which is not necessarily what the value means. A `Date` is reflected as whatever private fields the current Foundation uses; a type with a cache shows the cache; a type that normalizes its input shows the normalized form. Code that interprets these details makes promises on the type's behalf that the type never made, and breaks without warning when the representation changes.

**Errors move from build time to run time.** Every mistake the compiler would have caught in typed code (a misspelled name, an unexpected type, a missing case) becomes, in reflective code, a failed cast or a wrong answer discovered while the program runs, possibly in production. The function signatures don't help either: a function that takes `Any` and reflects over it declares nothing about what it actually supports, so its contract lives only in documentation. Document it carefully, and fail loudly and precisely on input outside it.

**It's slow.** Creating a mirror allocates; each child is boxed into an `Any`; each cast consults type metadata at run time. That's negligible in a debugging aid and significant in a hot loop, often by one or two orders of magnitude compared with specialized code. Measure (Chapter 11) before letting reflection onto a critical path.

**There's usually a better tool.** For serialization, use `Codable`. For choosing properties dynamically, use key paths. For code that depends on a type's declaration, use a macro or a synthesized conformance. For behavior that varies by type, use a protocol. Reflection is for the remaining cases, where a program truly must cope with values it knows nothing about: debuggers, loggers, test helpers, and generic tooling at the edges of a system.

Used in those places, reflection is invaluable. Kept to those places, it won't undermine the guarantees that make the rest of the program trustworthy.
