---
title: "Go Tip: Creating Closed Interfaces"
subtitle: "Controlling implementations of an interface"
date: 2024-12-14
draft: false
author: "Matthew Rodusek"
tags: [go, tip]
categories: [design-patterns, go-tips]
header-backlink: title
---

In {{<tag go>}}, interfaces are satisfied by any type that implements all of the
methods defined in the interface. This makes Go interfaces "open" types, meaning
that any type can satisfy an interface at any time.

However, did you know that Go actually has a mechanism for restricting who can
implement these types? This {{<category "go-tip">}} will tell you how you can
accomplish this.

<!-- more -->

{{<table-of-contents>}}

## Goal

To control who can implement an `interface`, rather than allowing
_any_ type to implement it.

## Background

In many programming languages, interfaces need to be implemented in some
explicit way:

* Languages like Java or C# have dedicated keywords like `implements`
to declare that a class implements an interface,
* Languages like Rust have `impl` for `trait` bounds
* Languages like C++ have `virtual` functions, and are implemented by specific
  syntax,
* etc.

Effectively, implementation is an "opt-in" process in many languages.

Go, on the other hand, approaches this differently: `interface`s are _implicitly_
satisfied by defining all of the methods in its `interface` definition. This is
a powerful and flexible feature, since it makes testing code using external
libraries much easier -- but it comes with the notable drawback that
patterns like "marker interfaces" or "closed interfaces" are not immediately
possible.

Thankfully, there actually _is_ a way to accomplish this by leveraging
function visibility in Go.

## Solution

### Unexported Methods

Any type in Go may implement an exported func from an `interface`, but
it turns out only types in the same package as the `interface` may implement
_unexported_ funcs of an `interface`.

Let's see what this means:

```go
package example

type ClosedInterface interface {
  closed() // <-- Unexported func
}

type Widget struct{}

func (w Widget) closed() {}

var _ ClosedInterface = (*Widget)(nil) // <-- This is allowed
```

In this example, the `ClosedInterface` has a single unexported method `closed`,
implemented by `Widget`. From the `var _` assignment, we can see that `Widget`
satisfies the `ClosedInterface`.

But what if we do this from outside the package?

```go
package blogpost

import "rodusek.dev/post/2024-12-14/example"

type OtherWidget struct{}

func (w OtherWidget) closed() {}

var _ example.ClosedInterface = (*OtherWidget)(nil) // <-- This is not allowed!
```

In this one, we get an error that looks like:

```raw
Error: cannot use OtherWidget literal (type *OtherWidget) as type example.ClosedInterface in assignment
```

So we can see that the `OtherWidget` type is not allowed to implement the
unexported `closed` method.

The take-away here is that you can control who can implement an `interface` by
making the methods unexported. But what if you _do_ want to allow other types
to implement it, but limit the scope?

### Embedding Types

Go allows for embedding types within `struct` definitions. The `struct` will
then "inherit" all receivers from the embedded type into its interface, and
more generally, will also now implement _all_ `interface`s that the embedded
type implements.

This has a really interesting consequence: You can control _which_ types
implement an `interface` by forcing it to embed a specific type:

```go
package example

type ClosedInterface interface {
  closed() // <-- Unexported func
}

type BaseClosedInterface struct{}

func (b BaseClosedInterface) closed() {}

var _ ClosedInterface = (*BaseClosedInterface)(nil)
```

In a new package, we can have:

```go
package blogpost

import "rodusek.dev/post/2024-12-14/example"

type OtherWidget struct {
  example.BaseClosedInterface // <-- Embed the BaseClosedInterface
}

var _ example.ClosedInterface = (*OtherWidget)(nil) // <-- This is now allowed!
```

By using the embedded type, we can now control which types are allowed to
implement the `ClosedInterface` by forcing them to embed the
`example.BaseClosedInterface`.

## Effective Use

Okay, so we have a way to control who can implement an `interface`, but lets
see how we can use this effectively in practice.

### Case 1: Future-Proofing APIs

Imagine you have a library that has an `interface` that is widely used, and
users are able to implement this `interface` themselves and provide it to your
library.
You want to add a new method to this interface, but this would become a breaking
change since _all_ types that currently implement the `interface` will need to
be updated to implement the new receiver func.

It turns out, having embeddable types here can help force your clients to
be future-proof. You can force the user to embed a `Base*` type which implements
an unexported func, while also providing "default" implementations to interface
methods. If new methods are added, the `Base*` type updates with it, and all
clients will inherit the new default for free, not forcing a code-change.

#### Example

Imagine you have an interface:

```go
package example

import "errors"

type GreetingService interface {
  SayHello(name string) error
  service()
}

type BaseGreetingService struct{}

func (b BaseGreetingService) SayHello(name string) error {
  return errors.New("unimplemented")
}

func (b BaseGreetingService) service() {}

var _ GreetingService = (*BaseGreetingService)(nil)
```

Users can use this package like so:

```go
package blogpost

import (
  "fmt"

  "rodusek.dev/post/2024-12-14/example"
)

type MyGreetingService struct {
  example.BaseGreetingService
}

func (m MyGreetingService) SayHello(name string) error {
  fmt.Println("Hello", name)
  return nil
}

var _ example.GreetingService = (*MyGreetingService)(nil)
```

You now get a new requirement that the `GreetingService` must also be able to
say goodbye -- thus you also want a `SayGoodbye` method. Supporting this now
just becomes a matter of adding the method to both the `interface` and the
`BaseGreetingService`:

```go
package example

import "errors"

type GreetingService interface {
  SayHello(name string) error
  SayGoodbye(name string) error
  service()
}

type BaseGreetingService struct{}

func (b BaseGreetingService) SayHello(name string) error {
  return errors.New("unimplemented")
}

func (b BaseGreetingService) SayGoodbye(name string) error {
  return errors.New("unimplemented")
}

func (b BaseGreetingService) service() {}

var _ GreetingService = (*BaseGreetingService)(nil)
```

Any users updating their library will now automatically inherit the new
`SayGoodbye` default implementation, and their code will continue to work.
Since a user is _forced_ to embed the `BaseGreetingService`, they are guaranteed
to have the new method available.

### Case 2: Marker Interfaces

Sometimes it can be desirable to design a library that either accepts or
returns a fixed set of known types to the user, even if they may not have any
explicit methods to implement. This is often referred to as a "Marker" interface.

By making the methods unexported, you can control which types are allowed to
be passed to a function, and which are not. Callers or function implementations
are then free to make expectations on the types returned by leveraging
type-switches on these types and controlling the behavior.

#### Example

Imagine you are designing a library that defines a set of semantic types used
as data-transfer-objects (DTOs). You want your API to only be able to speak
in terms of these types, and you want to ensure that the user cannot pass
in arbitrary types.

This can be easily accomplished by leveraging the unexported methods on
`interface`s.

```go
package example

type Primitive interface {
  primitive()
}

type String string
func (String) primitive() {}

type Int int
func (Int) primitive() {}

// ...

var _ Primitive = (*String)(nil)
var _ Primitive = (*Int)(nil)

type Service struct { /* ... */ }

func (s *Service) Process(p Primitive) {
  switch p.(type) {
  case String:
    s.processString(p.(String))
  case Int:
    s.processInt(p.(Int))
  // ...
  }
}
```

In this example, the `Primitive` interface is a marker interface that is
satisfied by `String` and `Int`. The `Service` struct can then process these
types by leveraging type-switches on the `Primitive` interface.

## Closing Thoughts

Go's interfaces are powerful tools that allow for a lot of flexibility in
designing APIs. By leveraging unexported methods, you can provide a tighter
control over who can implement an interface, and how they can be used.

This can be useful for ensuring backwards compatibility, or for designing
APIs that are more restrictive in the types they accept.
