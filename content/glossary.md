+++
title = "Glossary"
description = "Commonly used terms"
aliases = ["glossary", "dictionary"]
author = "Matthew Rodusek"
+++

Some of the terminology I use throughout this blog may be both technical and
uncommon, so this page serves as a quick-reference.

## Table of Contents

1. [Vocabulary Type](#vocabulary-type)
2. [Semantic Type](#semantic-type)
3. [Cardinality](#cardinality)
4. [Correct on Construction](#correct-on-construction)

### Vocabulary Type

A **Vocabulary Type** is, simply put, a type that becomes part of your core
API vocabulary. These are types that convey meaning and purpose in a
ubiquitous way to the user. Additionally, they encapsulate their state
and project a consistent API to their consumers that helps to ensure
correctness.

Such types are very seldomly interfaces; usually they are value objects that
can be composed into higher order constructions in some type-safe way.

Some simple examples of standard vocabulary types:

* `std::string`: A sequence of arbitrary characters of (usually) printable
  characters
* `std::filesystem::path`: A subset of `std::string` that represents only
  valid filesystem paths
* `std::optional`: A type that represents a potentially nullable value object
* etc.

### Semantic Type

A **Semantic Type** is a type that conveys some form of _semantics_ to the
type system.

Some examples of standard semantic types:

* Smart pointers: `std::unique_ptr<T>`, `std::shared_ptr<T>`, and
  `std::weak_ptr<T>` all convey both **nullability** and **ownership** to the
  consumer
* `std::optional<T>`: conveys the possibility of **nullability** but without
  any indirection
* `T*`: Built-in, but this conveys (usually) non-owning indirect viewing of
  a potentially non-owning piece of data
* etc.

**Semantic Type**s are a subset of **Vocabulary Types**. A **Semantic Type**
will always be a **Vocabulary Type**, but a **Vocabulary Type**
will not always be a **Semantic Type**

### Cardinality

The **Cardinality** of a type is its range of valid, logical input values.

For primitive types, this is quite simple because in most cases it maps 1-1 with
its valid numerical ranges (e.g. `uint8_t` is `0` to `255`).

For aggregate types, this will be the product of the cardinality of its members.
For example:

```cpp
struct Foo {
  std::uint8_t u;
  bool b;
};
```

The `cardinality(Foo)` will be `cardinality(u) * cardinality(b)`.

This gets trickier with user-defined types, since different practices may
limit the range of valid input (see [correct on construction](#correct-on-construction))

### Correct on Construction

**Correct on Construction** refers to the practice of only ever being able to
construct a type that is in a valid state. Doing so guarantees that any
valid instance or reference to that object can only refer to something that
is "valid" and thus does not require additional late-checking.

Primitive types are always trivially correct-on-construction because they can
only ever occupy their complete range of values.
