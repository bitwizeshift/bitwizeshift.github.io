---
title: "Builder Pattern: Expressing ownership transfer"
subtitle: "A simple underused approach to express moving members out of an object"
date: 2021-03-02T17:06:40-05:00
draft: false
tags: [ref-qualifiers, design]
languages: [c++, c++11]
categories: [design-patterns]
aliases: [/posts/2020-03-02/builder-pattern-expressing-ownership-transfer]
---

The Builder pattern is a common design in software development for late-binding
inputs to allow for iterative construction. This pattern, when applied in `c++`,
leaves one very obvious question: **Can we present this to consumers in an
optimal way?**

More specifically: when we call `build()`, should we be moving any temporary
internal state, or copy it? Can we express both in a safe and idiomatic way to
consumers?

<!--more-->

--------------------------------------------------------------------------------

This was a problem I faced in a personal project of mine where I was using a builder
pattern for producing a `Mesh` object for the purpose of rendering. I wanted to
allow for the client to call `build()` in one of two ways:

1. To cheaply create a `Mesh` object by moving the internals of the existing
   builder (no copies, but destructively mutates the builder), and
2. To optionally create a `Mesh` without moving the internals of the builder, so
   that the builder may safely be reused.

I wanted this to be expressed in a safe and C++-idiomatic fashion.

{{<table-of-contents>}}

## Goal

The primary goal is to create a builder pattern that gives the user the
opportunity to transfer ownership of internal resources while still conveying
this to the caller in an idiomatic way.

This should support a potentially-destructive `build()` that is clear to
the caller what the intent is to avoid potential use-after-moves errors.

For this post, I will be using a `StringBuilder` as an example of something that
*may* potentially be an expensive object to copy in practice, and eliding such
copies may be desirable.

## Possible Options

There are a couple of obvious and straight-forward ways that this could be
implemented which I will walk through first; however they each have hidden
costs.

### Presenting a `const` and non-`const` `build()` function

The simplest / easiest approach would be to offer two sets of functions as an
overload set: one that copies, one that moves:

```cpp
class StringBuilder
{
public:
  ...
  auto build() -> std::string {
    return std::move(m_data); // move!
  }
  auto build() const -> std::string {
    return m_data; // copy!
  }
  ...
private:

  std::string m_data;
};
```

The problem with this approach is subtle; if your local instance of
`StringBuilder` is `const`, this will result in a copy -- otherwise this will
always **destructively mutate** causing future uses to be **undefined behavior**.
This is not obvious from inspection, and would likely be missed on code-review:

```cpp
auto builder = StringBuilder{};
builder.append(...);
...
auto s1 = builder.build();

// keep reusing builder
builder.append(...);
...
auto s2 = builder.build();
```

Did you catch that bug?

`builder` is a non-`const` `StringBuilder` object. The first call to
`builder.build()` will call the non-`const`-qualified overload of `build()`,
resulting in `std::move(m_data)`. This results in any future operations like
`builder.append(...)` or the future `builder.build()` call to be operating on
a moved-from `std::string` object -- which is **undefined behavior**!

This problem would be subtle, and easy to miss on reviews. Lets try a different
approach.

### Present differently named `build()` functions

The next logical progression from the first option is to distinguish the
destructive `build()` function from the non-destructive `build()` function by
simply *naming them differently*. For example, we could name one `build()` and
the other `build_destructive()`

```cpp
class StringBuilder
{
public:
  ...
  auto build_destructive() -> std::string {
    return std::move(m_data); // move!
  }
  auto build() const -> std::string {
    return m_data; // copy!
  }
  ...
private:

  std::string m_data;
};
```

This would make the use of the code a little more clear, which should hopefully
avoid errors:

```cpp
auto builder = StringBuilder{};
builder.append(...);
...
auto s1 = builder.build_destructive(); // <-- the bug is a little more obvious

// keep reusing builder
builder.append(...);
...
auto s2 = builder.build();
```

This now makes the bug from the first iteration a _little more obvious_ -- but
with the one catch that you have to know the API to understand what it's doing.
This effectively forces a nomenclature on the codebase where only developers
who are comfortable with the code or have read the documentation
will be aware of what the distinction and repercussions are for this use.

A newcomer to your codebase might see `build_destructive` but not realize that
this actually produces a possible error downstream; it isn't *idiomatic*.

Perhaps we can find a better solution that the average modern C++ developer may
catch?

## Ref-qualifying functions

It turns out, `c++11` actually offers something that would make this easier to
catch from most general developers.

`c++11` was one of the largest changes to the C++ programming language in
decades, adding dozens of new language features that many developers don't
necessarily know about. Of these features is a peculiar new qualification to
member functions called
[ref-qualified functions](https://en.cppreference.com/w/cpp/language/member_functions#ref-qualified_member_functions).

Similar to CV-qualifications, ref-qualifications allow you to designate
non-static member functions that will be invoked based on whether a type is
used from the context of an lvalue-reference, or an rvalue-reference.

As example, you can have:

```cpp
struct Foo {
  void do_something() & { std::cout << "&-qualified" << std::endl; }
  //                  ^ - lvalue qualified
  void do_something() && { std::cout << "&&-qualified" << std::endl; }
  //                  ^~ - rvalue qualified
};

auto f = Foo{};
f.do_something();            // prints '&-qualfiied'
std::move(f).do_something(); // prints '&&-qualified'
Foo{}.do_something();        // prints '&&-qualified' (since this is a temporary)
```

This allows you to have different behavior based on whether it's called from
an rvalue context compare to an lvalue context.

Ref-qualified overloads are mostly seen in general-purpose utilities like
`std::optional` to allow for the underlying data being accessed to have the
same refness as the caller -- but it turns out this same mechanism can be used
to help us to develop clean, idiomatic APIs.

### Applying ref-qualifications to our builder

So how can we now apply ref-qualifications to our builder?

Simple; we offer an overload set of the `build()` function, where one operates
on a non-`const` rvalue (`&&`) qualified object, and the other operates on
a `const` lvalue (`&`) qualified object. This is effectively fitting our first
implementation with ref-qualifications:

```cpp
class StringBuilder
{
public:
  ...
  auto build() && -> std::string {
    //         ^~ --- new
    return std::move(m_data);
  }
  auto build() const & -> std::string {
    //               ^ --- new
    return m_data; // copy!
  }
  ...
private:

  std::string m_data;
};
```

Lets take a look at how this changes our use of this code:

```cpp
auto builder = StringBuilder{};
builder.append(...);
...
auto s1 = std::move(builder).build();

// keep reusing builder
builder.append(...); // <-- very clear use after move!
...
auto s2 = builder.build();
```

With this relatively simple transformation, we can now make detection of this
use-after-move a lot more idiomatic and visible -- even to developers who are
unfamiliar with the current code.

In general, it's safe to assume that `std::move(x)` will move the contents of
`x` in some way. So it stands to reason that a call of
`std::move(builder).build()` will also be idiomatic to assume that `builder`
is somehow being destructively mutated after the call -- which will result in a
use-after-move. This now communicates the destructiveness from the API
surface-area.

Similarly, a call of `builder.build()` is safe to assume will not result in an
use-after-move effects.

Ultimately, this approach actually lets us offer move-semantics in a way that
won't subtly break clients -- all while still allowing for object reuse to
consumers who desire it. It also *forces* the client of this API to think about
what they are doing -- since the only way to get the data out efficiently is to
call `std::move`, which should be an early warning sign of its destructiveness.

## Closing Remarks

The key take-away from this is that ref-qualified functions are powerful in
their ability to express ownership transfer. Not only does this present a
lightweight function to clients by allowing them to reuse the internal state
of the object -- it also allows this to be done so in an idiomatic way that
would stand out to developers and static-analysis tools if misused.

Next time you want to allow both copying and moving of a class' member, consider
offering a ref-qualified overload set!
