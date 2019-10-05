---
layout: post
title:  "DecodableEither"
date:   2019-10-4 12:46:00 +0200
---

It's not uncommon for the back-end guys to return some JSON where the type of a field is *either* a number or a string.
This causes inconvenience at the app development level where it's not enough to declare vanilla `Codable` structs.
Thankfully, `Decodable` makes this way less messy than first imagined.
We can wrap the *either* logic in a separate `Decodable` type that handles this for us:

```swift
struct Product: Decodable {
    let id: EitherIntOrString
    
    struct EitherIntOrString: Decodable {
        let value: Int
        
        init(from decoder: Decoder) throws {
            let values = try decoder.singleValueContainer()
            do {
                value = try values.decode(Int.self)
            } catch {
                let string = try values.decode(String.self)
                guard let int = Int(string) else {
                    throw ParsingError.stringParsingError
                }
                
                value = int
            }
        }
    }
    
    enum ParsingError: Error {
        case stringParsingError
    }
}
```

So, JSON like the following parses successfully:

```json
[
    { "id": 12 },
    { "id": "14" }
]
```

### Generalizing

We can make a generic *either* type out of this.
All we need is two `Decodable` types, and a converter from a type to the other.
Let's see:

```swift
protocol Converter {
    associatedtype T1
    associatedtype T2
    static func convert(_ t2: T2) -> T1?
}

struct DecodableEither<T1: Decodable, T2: Decodable, C: Converter>: Decodable where C.T1 == T1, C.T2 == T2 {
    
    let value: T1

    init(from decoder: Decoder) throws {
        let values = try decoder.singleValueContainer()
        do {
            value = try values.decode(T1.self)
        } catch {
            let t2 = try values.decode(T2.self)
            guard let t1 = C.convert(t2) else {
                throw Error.conversionError
            }
            
            value = t1
        }
    }
    
    enum Error: Swift.Error {
        case conversionError
    }
}
```

Let's break this down:

1. `DecodableEither<T1: Decodable, T2: Decodable, C: Converter>: Decodable`. 
Here we declare a generic struct that conforms to `Decodable`, and depends on three types.
The first two are any `Decodable` types.
The third is just a type that conforms to a protocol called `Converter` that we will use for converting from type `T2` to `T1`.
2. The `Converter` protocol declares a static function that converts from a generic type to another.
Such protocol is called *protocol with associated types*, commonly called "PATs".
3. Now, the Swiftiest thing in this code, the type constraints. `where C.T1 == T1, C.T2 == T2`.
This part after the `DecodableEither` declaration is what ensures type-safety and makes things work together.
Here we tell the Swift compiler to ensure that any `Converter` type passed to us here must have its two associated types be the very two types passed to the `DecodableEither`.
This what makes that line `let t1 = C.convert(t2)` in the alternate decoding phase work, and infer correctly the given types.

Now, we can use this generic type like this:

```swift
enum StringToIntConverter: Converter {
    static func convert(_ t2: String) -> Int? {
        return Int(t2)
    }
}

struct Product: Decodable {
    let id: DecodableEither<Int, String, StringToIntConverter>
}
```
Usage:
```swift
let json = """
    [
        { "id": 12 },
        { "id": "14" }
    ]
"""
        
let products = try! JSONDecoder().decode([Product].self, from: json.data(using: .utf8)!)

products.forEach({
    print($0.id.value) // value is Int 👍
})
```

We can also use typealisases if a particular combination is used frequently:
```swift
typealias DecodableEitherIntOrString = DecodableEither<Int, String, StringToIntConverter>

struct Product: Decodable {
    let id: DecodableEitherIntOrString
}
```

That's it. Thanks for reading!