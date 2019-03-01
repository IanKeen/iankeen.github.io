Imagine we want to build an app to track books. The API we are building against provides JSON like:

```
// authors
{
    "identifier": "A13424B6",
    "name": "Robert C. Martin"
}

// books
{
    "identifier": "A161F15C",
    "title": "Clean Code",
    "authorIdentifier": "A13424B6"
}
```

Swifts `Codable` features enable us to quickly create matching models:

```swift
struct Author: Codable {
    let identifier: String
    let name: String
}

struct Book: Codable {
    let identifier: String
    let title: String
    let authorIdentifier: String
}
```

Thanks to `Codable` this is all we have to do to get JSON mapping to type-safe models for free!

There are, however, a couple of subtle issues that could cause problems as we progress. The `identifier`s are defined as `String`s, while this seems correct it could lead to a scenario like:

```swift
func allBooks(by authorIdentifier: String) -> [Book] {
    //.. lookup books by id
}

let bobsBooks = allBooks(by: "Robert C. Martin") // oops!
```

This would compile and the call-site _seems_ to read correctly.. but it would never return anything because the function expects an authors identifier not their name. Let's look at a type we can use instead of `String` to improve the type-safety here.

## Identifier\<T>

We can't change the fact that the server is sending us `String`s but we can change how those strings are represented locally using a wrapper and phantom-types.

Usually when we define a generic type like `Identifier<T>` we also use that `T` elsewhere in the type, something like `let value: T`. However when the `T` is only present as part of the declaration it is called a phantom type.

What's the point then? why make something generic if we aren't using the type? Well we actually _are_ using the type, just not in the usual way. Let's take a look:

```swift
struct Identifier<T> {
    let value: String
}
```

Updating our models to use this new type, they become:

```swift
struct Author: Codable {
    let identifier: Identifier<Author>
    let name: String
}

struct Book: Codable {
    let identifier: Identifier<Book>
    let title: String
    let authorIdentifier: Identifier<Author>
}
```

and our function now becomes:

```swift
func allBooks(by authorIdentifier: Identifier<Author>) -> [Book] {
    //.. lookup books by authorIdentifier.value
}

let bobsBooks = allBooks(by: "Robert C. Martin") // failure!! (the good kind)
```

We have now made it impossible to pass the incorrect type to our function otherwise it will not compile.

As for the phantom type, we have now used the same type `Identifier<T>` to represent both `Book` and `Author` identifiers. This can now be used for any number of types and because the generic `T` must be what we specify the complier won't allow us to make any mistakes.

Thats what makes phantom types useful!

## Codable support

There is a new problem our new `Identifier<T>`  has introduced. `Codable`, by default, `Codable` uses the same structure as the type. This means the JSON representation of an identifier would be:

```
{"value": "A13424B6"}
```

This is wrong, we still want its JSON representation to be a `String` so it works seamlessly with our API. Let's fix the `Codable` behaviour:

```swift
extension Identifier: Codable {
    init(from decoder: Decoder) throws {
        self.value = try String(from: decoder)
    }
    func encode(to encoder: Encoder) throws {
        try value.encode(to: encoder)
    }
}
```

Now when we convert between the model and JSON it will be a regular `String` rather than a nested object.

## Adding new data to the API

Listing books and authors is working really well, now it's time to allow our users to submit new entries. The only problem is our API is responsible for determining the identifiers of new data so we want to send JSON containing everything _but_ the object identifier.

There are a few ways we could tackle this:

1. We could maintain a separate model that excludes the `identifier`. This is tedious and fragile but perhaps we could leverage a codegen solution to help? This is a big dependency to add if you aren't already using one though.

2. We could provide a dummy identifier and remove it from the JSON before sending it to our API. This isn't very nice though, using dummy values in production code seems like something just begging to break and/or corrupt things.

Since we are creating new types today let's look at another one that can be used to solve this problem.

## Identified\<T>

What we need is a way to define two version of our models. One with an `identifier` when data is sent down and one without an `identifier` when we send data up.

We can't use the type system to _remove_ properties... but we can use it to _add_ them. 

```swift
struct Identified<T> {
    let identifier: Identifier<T>
    let value: T
}
```

Using this type we can remove the `identifier` property from our models.

```swift
struct Author: Codable {
    let name: String
}

struct Book: Codable {
    let title: String
    let authorIdentifier: Identifier<Author>
}
```

We can then define the two version of a model we need. When we are receiving books from the API we can use `Identified<Book>` for each instance. When we want to add a new book we simply use `Book` as-is.

Having a type like `Identified<T>` gives us the flexibility we want without needing to maintain parallel models or hack out unwanted values before sending data.

## Codable support

As with `Identifier<T>` the default `Codable` behaviour for `Identified<T>` would result in the wrong JSON:

```
{
    "identifier": "A13424B6",
    "value": {
        "name": "Robert C. Martin"
    }
}
```

We need to fix the `Codable` behaviour so everything is still flattened when in it's JSON form:

```swift
extension Identified: Codable where T: Codable {
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: AnyCodingKey.self)
        
        self.identifier = try container.decode(Identifier<T>.self, forKey: AnyCodingKey(stringValue: "identifier")!)
        self.value = try T.init(from: decoder)
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: AnyCodingKey.self)
        
        try container.encode(identifier, forKey: AnyCodingKey(stringValue: "identifier")!)
        try value.encode(to: encoder)
    }
}
```

The `AnyCodingKey` type is used to allow us to dynamically decode certain parts of the data without needing all new types:

```swift
struct AnyCodingKey: CodingKey {
    var stringValue: String
    var intValue: Int?
    
    init?(intValue: Int) {
        self.intValue = intValue
        self.stringValue = "\(intValue)"
    }
    init?(stringValue: String) {
        self.intValue = nil
        self.stringValue = stringValue
    }
}
```

## The end!

There are many ways to skin a cat, but hopefully this has shown a couple of interesting way to solve some common problems using wrapper types and some pretty nifty Swift features.

If you have any feedback feel free to reach out!