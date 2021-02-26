---
title: "Creating a Fast and Efficient Delegate Type"
date: 2021-02-24T20:03:14-05:00
draft: false
tags: [performance, templates, delegation, c++17]
categories: [intermediate-templates, tutorial]
---

When working in C++ systems, a frequent design pattern that presents itself is
the need to bind and reuse functions in a type-erased way. Whether it's
returning a callback from a function, or enabling support for binding listeners
to an event (such as through a `signal` or observer pattern), this is a pattern
that can be found in many places.

In many cases, especially when working in more constrained systems such as an
embedded system, these bound functions will often be nothing more than small
views of existing functions -- either as just a function pointer, or a coupling
of `this` with a member function call. In almost all cases, the function being
bound is **already known at compile-time**. Very seldomly does a user _need_ to
bind an opaque function (such as the return from [`::dlsym`](https://linux.die.net/man/3/dlsym)).

Although the C++ standard does provide some utilities for type-erased callbacks
such as [`std::function`](https://en.cppreference.com/w/cpp/utility/functional/function)
or [`std::packaged_task`](https://en.cppreference.com/w/cpp/thread/packaged_task),
neither of these solutions provide any standards-guaranteed performance
characteristics, and neither of these are guaranteed to be optimized for the
above suggested cases.

We can do better. Lets set out to produce a better alternative to the existing
solutions that could satisfy this problem in a nice, coherent, and 0-overhead
way.

## Goal

To create a fast, light-weight, 0-overhead alternative to `std::function` that
operates on non-owning references. For our purposes, we will be named
`Delegate`.

The primary criteria will be:

1. [ ] Functions are bound at compile time
2. [ ] Covariant functions can be bound at compile time
3. [ ] 0-Overhead

## A First Attempt

### Basic Structure

The most obvious start for this is to create a class template that works with
any function signature.
Since we need to know both the type of the return and the type of the arguments
in order to invoke the function, we will want to use a partial specialization
that allows us to extract this information.

```cpp
// Primary template intentionally left empty
template <typename Signature>
class Delegate;

// Partial specialization so we can see the return type and arguments
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  // Creates an unbound delegate
  Delegate() = default;

  // We want the Delegate to be copyable, since its lightweight
  Delegate(const Delegate& other) = default;
  auto operator=(const Delegate& other) -> Delegate& = default;

  ...

  // Call the underlying bound function
  auto operator()(Args...args) const -> R;
private:
  ...
};
```

As with any type-erasure, we need to think of how we will intend to normalize
*any* input into a `Delegate<R(Args...)>`.

Keeping in mind that this should support both raw function pointers _and_
bound member functions with a class instance, we will need to be able to store
at least two pieces of data:

1. a pointer to the instance (if one exists), and
2. a function pointer to perform the invocation

Remembering that a member function pointer is conceptually similar to a
function pointer that takes a `this` pointer as the first instance -- lets
model that with `R(*)(This*, Args...)`!

But what should the type of `This` be? There is no _one_ type for `this` if we
are binding _any_ class object. Raw pointers don't even _have_ a `this`
pointer at all!

Lets solve this with the simplest, most C++ way we can think of: `const void*`.
This can be `nullptr` for raw function pointers that have no `this`, or it can
be a pointer to the instance for bound member functions! Then we can simply
cast the type back to the correct `This` pointer as needed -- keep it simple!

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
  ...
private:
  using stub_function = R(*)(const void*, Args...);

  const void* m_instance = nullptr; ///< A pointer to the instance (if it exists)
  stub_function m_stub = nullptr;   ///< A pointer to the function to invoke
};
```

Great! Now we just need some way of generating these stub functions.

### Generating Stubs

So how can we create a stub function from the actual function we want to bind?

Keeping in mind that we want this solution to work only for functions known at
**compile-time** gives us the answer: `template`s! More specifically:
non-type `template`s.

So lets take a first crack at how we can create a stub for non-member functions:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
  ...
private:
  ...

  /// A Stub function for free functions
  template <R(*Function)(Args...)>
  static auto nonmember_stub(const void* /* unused */, Args...args) -> R
  {
    return std::invoke(Function, args...);
  }

  ...
};
```

Now we have something that models the stub function, so we just need a
way to actually _make_ the delegate. Lets do this with a simple `bind`
function:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...

  template <R(*Function)(Args...)>
  auto bind() -> void
  {
    // We don't use this for non-member functions, so just set it to nullptr
    m_instance = nullptr;
    m_stub = &nonmember_stub<Function>;
  }

  ...
};

// Called like Delegate<void()>{}.bind<&foo>();
```

Perfect; now we have a means of binding free functions. But it turns out there
is an even simpler way so that we can avoid having to write a `static` function
at all -- and the answer is **lambdas**.

One really helpful but often unknown feature of non-capturing lambdas is that
they are convertible to a function pointer of the same signature. This allows us
to avoid wring a whole separate function template:

```cpp
  template <R(*Function)(Args...)>
  auto bind() -> void
  {
    m_instance = nullptr;
    m_stub = static_cast<stub_function>([](const void*, Args...args) -> R{
      return std::invoke(Function, args...);
    });
  }
```

### Invoking the `Delegate`

We still need to implement the invocation of `operator()` with `m_instance`.
Lets do this before we continue.

Since `Delegate` may be constructed without any bound functions, lets throw
an exception on failure so that we have feature-parity with `std::function`

```cpp
class BadDelegateCall : public std::exception { ... };

template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...

  auto operator()(Args...args) const -> R
  {
    if (m_stub == nullptr) {
      throw BadDelegateCall{};
    }
    return std::invoke(m_stub, m_instance, args...);
  }
  ...
};
```

### Goal Check

Before we go any further, lets look back at our original goals

#### ✔ Functions are bound at compile time

We definitely satisfied the ability to bind a non-member-function at
compile-time!

#### ❔ Covariant functions can be bound at compile time

Does our implementation support covariance?

A simple way to test this would be to see if we can bind a `void(long)` to a
`Delegate<void(int)>`:

```cpp
auto test(long) -> void;

auto d = Delegate<void(int)>{};
d.bind<&test>();
```

Uh-oh, this yields the following error:

>     <source>:17:19: error: no matching function for call to 'Delegate<void(int)>::bind<test>()'
>        17 |     d.bind<&test>();
>           |                   ^
>     <source>:10:10: note: candidate: 'template<void (* <anonymous>)(int)> void Delegate<R(Args ...)>::bind() [with R (* <anonymous>)(Args ...) = <anonymous>; R = void; Args = {int}]'
>        10 |     auto bind() -> void;
>           |          ^~~~
>     <source>:10:10: note:   template argument deduction/substitution failed:
>     <source>:17:19: error: could not convert template argument 'test' from 'void (*)(long int)' to 'void (*)(int)'
>        17 |     d.bind<&test>();
>           |                   ^

This makes sense, since we only have non-type templates of the specific
signature; but `void(*)(long)` is not the same as `void(*)(int)`.

Lets go back and make this work with covariance!

## Iteration: Covariance

### Reworking Function Binding

So how can we support covariance? We need a non-type `template` parameter that
supports *any* function pointer signature that may be similar.

This is where `C++17`'s `auto`-template parameters will play a huge role.
`auto` parameters are non-type parameters that use `auto`-deduction semantics.

Lets try making this change.

```cpp
  template <auto Function>
  auto bind() -> void
  {
    m_instance = nullptr;
    m_stub = static_cast<stub_function>([](const void*, Args...args) -> R{
      return std::invoke(Function, args...);
    });
  }
```

With this, now the following code compiles:

```cpp
auto test(long) -> void;

auto d = Delegate<void(int)>{};
d.bind<&test>();
```

Right now the `auto` parameters are unconstrained, meaning that you could
realistically call `bind<2>()` and this will fail spectacularly with some
horrible template error. However, we can easily fix this by just constraining
the template's inputs by using [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae):

```cpp
  template <auto Function,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(Function),Args...>>>
  auto bind() -> void
  {
    ...
  }
```

This will now ensure that calling `bind<2>()` will error that there is no
valid overload available, rather than failing with some complicated template
error.

### Supporting Member Functions

Now we need to support member functions. Recall that member functions in C++
may be both `const` and non-`const` -- so there will be two separate overloads
to support.

Lets start with the `const` version. We will want to take in either a pointer
or a reference to the object being bound; let's choose a reference so we don't
have to consider nullptr inputs.

We will only want this to work with member function pointers that are invocable
from the reference, so we should use SFINAE to ensure that this is the case.

```cpp
  template <typename Class, auto MemberFunction,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(MemberFunction),const Class*, Args...>>>
  auto bind(const Class& cls) -> void
  {
    // addressof used to ensure we get the proper pointer
    m_instance = std::addressof(cls);

    m_stub = static_cast<stub_function>([](const void* p, Args...args) -> R{
      // Cast back to the correct type
      const auto* c = static_cast<const Class*>(p);
      return std::invoke(Function, p, args...);
    });
  }
```

Recall that for this to work, we will need to cast the `const void*` back to
a `const Class*`.

Okay, now lets do the same for non-`const` member functions:

```cpp
  template <typename Class, auto MemberFunction,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(MemberFunction),Class*, Args...>>>
  auto bind(Class& cls) -> void
  {
    m_instance = std::addressof(cls);
    m_stub = static_cast<stub_function>([](const void* p, Args...args) -> R{
      // Note: we have to const_cast here
      const auto* c = const_cast<Class*>(static_cast<const Class*>(p));
      return std::invoke(Function, p, args...);
    });
  }
```

Recall back when we created `m_instance`, we stored this as a `const void*`.
Because of this, we need to `const_cast` the pointer so that it can be mutable.
Although `const_cast` is normally a cause for concern, this is one of the few
cases where its completely safe -- since we know that the input was mutable
from the start since `cls` was mutable.

There! We now have raw functions and member functions.

### Goal Check

Time to check back with our goals.

#### ✔ Functions are bound at compile time

We still bind everything at compile-time -- check!

#### ✔ Covariant functions can be bound at compile time

`auto` non-type parameters provide us with covariance support -- check!

#### ❔ 0-Overhead

Does this have zero overhead?

This certainly has a very small memory footprint, weighing in at one
`const void*` and a raw function pointer. But how does this perform? Does this
do any extra work that it shouldn't? Is it as _lightweight_ as a pointer?

## Iteration: 0-overhead

### Binding functions with move-only parameters

Let's do a simple test to see if we are doing any unexpected copies that we
don't want.

In particular, if we pass a move-only type to this constructor, will it do as
we request and move the objects along -- or will it copy?

A simple test:

```cpp
auto test(std::unique_ptr<int>) -> void {}

...

Delegate<void(std::unique_ptr<int>)> d{};
d.bind<&::test>();
```

This reveals a breakage, before even trying to call it:

>     <source>: In instantiation of 'void Delegate<R(Args ...)>::bind() [with auto Function = test; <template-parameter-2-2> = void; R = void; Args = {std::unique_ptr<int, std::default_delete<int> >}]':
>     <source>:58:19:   required from here
>     <source>:31:25: error: no matching function for call to 'invoke(void (*)(std::unique_ptr<int>), std::unique_ptr<int>&)'
>        31 |       return std::invoke(Function, args...);
>           |              ~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~

If we look back to what we did in the stub generated in the `bind` call, you
might notice that we only pass the arguments as-is to the underlying function.
Since the argument is a by-value `std::unique_ptr`, this generates a copy -- and
what we want is a _move_.

The solution here is simple: use `std::forward`! `std::forward` ensures that any
by-value or rvalue reference gets forwarded as an rvalue, and any lvalue
reference gets forwarded as an lvalue reference.

So this now changes the code to be:

```cpp
  template <auto Function,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(Function),Args...>>>
  auto bind() -> void
  {
    m_instance = nullptr;
    m_stub = static_cast<stub_function>([](const void*, Args...args) -> R{
      return std::invoke(Function, std::forward<Args>(args)...);
      //                           ^~~~~~~~~~~~~~~~~~~    ~
      //                           Changed here
    });
  }
```

(The same will also need to be done for the member function versions)

and now we can see the `bind` call succeed.

### Invoking functions with move-only parameters

Now what about invoking it? Lets see what happens when we try to invoke the
delegate:

```cpp
d(std::make_unique<int>(42));
```

>     <source>: In instantiation of 'R Delegate<R(Args ...)>::operator()(Args ...) const [with R = void; Args = {std::unique_ptr<int, std::default_delete<int> >}]':
>     <source>:60:32:   required from here
>     <source>:43:23: error: no matching function for call to 'invoke(void (* const&)(const void*, std::unique_ptr<int>), const void* const&, std::unique_ptr<int>&)'
>        43 |     return std::invoke(m_stub, m_instance, args...);
>           |            ~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The same problem as before occurs!

Well the fix here should be simple: just change it to `std::forward` as well!

```cpp
  auto operator()(Args...args) const -> R
  {
    if (m_stub == nullptr) {
      throw BadDelegateCall{};
    }
    return std::invoke(m_stub, m_instance, std::forward<Args>(args)...);
  }
```

This works, but can we perhaps do this a little bit better? The arguments being
passed here will currently only match the exact argument types of the input --
which means that a `unique_ptr` by-value will see a move constructor invoked
both here, and in the stub. Can we perhaps remove one of these moves?

It turns out, we can -- we just need to use more templates! We can use a
variadic forwarding reference pack of arguments that get deduced by the
function call. This will ensure that the fewest number of copies, moves, and
conversions will occur from `operator()`

```cpp
  template <typename...UArgs,
            typename = std::enable_if_t<std::is_invocable_v<R(Args...),UArgs...>>>
  auto operator()(UArgs&&...args) -> R
  {
    ...
  }
```

This is as few operations as can be performed, since we will always need to
draw the hard-line at the stub-function for type-erasure. This certainly is
0-overhead -- or as close to it -- as we can achieve.

The last thing we really should check for is whether a bound function in a
`Delegate` has similar performance to an opaque function pointer.

### Benchmarking

To test benchmarking, lets use Google's [Microbenchmarking](https://github.com/google/benchmark)
library.

#### A Baseline

The first thing we will need is a baseline to compare against. So lets
create that baseline to compare against. This baseline will use a basic function
pointer that has been made opaque by the benchmark. The operation will simply
write to some `volatile` atomic boolean, to ensure that the operation itself
does not get elided.

```cpp
volatile std::atomic<bool> s_something_to_write{false};

// Baseline function: does nothing
void do_nothing(){
  s_something_to_write.store(true);
}

void baseline(benchmark::State& state)
{
  auto* ptr = &::do_nothing;
  benchmark::DoNotOptimize(ptr); // make the function pointer opaque

  for (auto _ : state) {
    (*ptr)();
  }
}
BENCHMARK(baseline);
```

### The Benchmark

And now for the benchmark of the delegate:

```cpp
void delegate(benchmark::State& state)
{
  auto d = Delegate<void()>{};
  d.bind<&::do_nothing>();
  benchmark::DoNotOptimize(d); // make the delegate opaque

  for (auto _ : state) {
    d();
  }
}

BENCHMARK(delegate);
```

#### Comparing the benchmarks

The results are surprising:

![Benchmark 1](/img/2021-02-24/benchmark-1.png)

[See the results on quick-bench](https://quick-bench.com/q/7Ft0Q82qSOTvmx1lE1nx6MOA5jA)

We can see here that `Delegate` performs _ever so slightly_ slower than
a raw function pointer does. This appears to be a consistent, though tiny,
overhead.

Is there any way that we could potentially address this?

## Iteration: Optimizizing our `Delegate`

Since we are only benchmarking the time for `operator()` and not the time for
binding, that means the source of the time difference may be due to something
in that function.

```cpp
  auto operator()(Args...args) const -> R
  {
    if (m_stub == nullptr) {
      throw BadDelegateCall{};
    }
    return std::invoke(m_stub, m_instance, args...);
  }
```

If we look at `operator()`, we see that we have a branch checking
`m_stub == nullptr` that occurs for each invocation. Since its very unlikely
that this will actually be `nullptr` frequently (or intentionally), this seems
like an unnecessary pessimization for us; but is there any way to get rid of it?

Keeping in mind we only ever bind `nullptr` to `m_stub` on construction, what if
we instead bind a function whose sole purpose is to `throw` the
`BadDelegateCall`?

For example:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
  ...
private:
  ...

  [[noreturn]]
  static auto stub_null(const void* p, Args...) -> R
  {
    throw BadDelegateCall{};
  }

  ...

  stub_function m_stub = &stub_null;
};
```

This will allow us to remove the branch on each `operator()` call:

```cpp
  auto operator()(Args...args) const -> R
  {
    return std::invoke(m_stub, m_instance, args...);
  }
```

### Benchmarking Again

Lets see how this performs now by rerunning our original benchmark:

![Benchmark 2](/img/2021-02-24/benchmark-2.png)

[See the results on Quick Bench](https://quick-bench.com/q/1XppGG1P12SjBfu1sZdj1qLWJVw)

This is what we wanted to see!

We have successfully optimized our `Delegate` class to be exactly as fast as a
raw-function pointer, while also allowing binding class member functions as
well.

### Final Goal Check

With that completed, lets look one last time if we completed our goals.

#### ✔ Functions are bound at compile time

We still bind everything at compile-time -- check!

#### ✔ Covariant functions can be bound at compile time

`auto` non-type parameters provide us with covariance support -- check!

#### ✔ 0-Overhead

This absolutely has 0-overhead; a tiny memory footprint, no extra operations,
_AND_ it's exactly as fast as a raw function pointer. What more could you ask
for?

## Closing Thoughts

So there we have it.

We managed to check off all of the criteria from the goals laid out at the
start. This is just one such example where a few templates and some creative
problem solving can lead to a high-quality, 0-overhead solution.

This works well in `signal`-patterns, or in class delegation callbacks where the
lifetime of the callback is never exceed by the object being called back.

I am sure ther are a number of ways that this solution can be improved-upon, but
for most cases this is "good enough".
