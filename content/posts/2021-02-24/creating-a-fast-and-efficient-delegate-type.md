---
title: "Creating a Fast and Efficient Delegate Type (Part 1)"
subtitle: "A simple solution to lightweight function binding"
date: 2021-02-24T20:03:14-05:00
draft: false
tags: [templates, callback, c++11, c++14]
categories: [basic-templates, tutorial]
series: [delegate]
summary: "It's often desirable when working in C++ to create callbacks that
never leave a certain lifetime, and only bind functions or function pointers.
The C++ standard only offers `std::function` and `std::packaged_task`, both of
which are more heavy than they need to be.

We can do better. Lets produce a better alternative to the existing solutions
that could satisfy this problem in a nice and coherent way."
github:
  repository: delegate
aliases: [/posts/2021-02-24/creating-a-fast-and-efficient-delegate]
---

{{<info>}}
**Note:** This is part 1 of a 3 part series.
{{</info>}}

When working in C++ systems, a frequent design pattern that presents itself is
the need to bind and reuse functions in a type-erased way. Whether it's
returning a callback from a function, or enabling support for binding listeners
to an event (such as through a `signal` or observer pattern), this is a pattern
that can be found everywhere.

In many cases, especially when working in more constrained systems such as an
embedded system, these bound functions will often be nothing more than small
views of existing functions -- either as just a function pointer, or as a
coupling of `this` with a member function call. In almost all cases, the
function being bound is **already known at compile-time**. Very seldomly does
a user _need_ to bind an opaque function (such as the return value from a
call to [`::dlsym`](https://linux.die.net/man/3/dlsym)).

Although the C++ standard does provide some utilities for type-erased callbacks
such as [`std::function`](https://en.cppreference.com/w/cpp/utility/functional/function)
or [`std::packaged_task`](https://en.cppreference.com/w/cpp/thread/packaged_task),
neither of these solutions provide any standards-guaranteed performance
characteristics, and neither of these are guaranteed to be optimized for the
above suggested cases.

We can do better. Lets set out to produce a better alternative to the existing
solutions that could satisfy this problem in a nice and coherent way.

{{<table-of-contents>}}

## Goal

To create a fast, light-weight alternative to `std::function` that
operates on non-owning references. For our purposes, we will name this type
`Delegate`.

The criteria we will focus on in this post will be to be able to bind functions
to this type at **compile-time** in a way that works for `c++11` and above

Over the next few posts, we will iterate on this design to make it even better
by introducing **covariance** and **0-overhead**.

## Basic Structure

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
{
  ...
private:
  using stub_function = R(*)(const void*, Args...);

  const void* m_instance = nullptr; ///< A pointer to the instance (if it exists)
  stub_function m_stub = nullptr;   ///< A pointer to the function to invoke
};
```

Great! Now we just need some way of generating these stub functions.

## Invoking the `Delegate`

Before we go to far, lets implement the invocation of `operator()` with
`m_instance`.

Since we know that we want to erase a possible instance pointer to be
`m_instance`, and our stub is `m_stub` -- all we are really doing is calling
the `m_stub` bound function and passing `m_instance` as the first argument,
forwarding the rest along.

However, we will want to make sure we don't accidentally call this while `m_stub`
is `nullptr`, since that would be **undefined behavior**. Lets throw an
exception in such a case:

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
    return (*m_stub)(m_instance, args...);
  }

  ...
};
```

Okay, now before we can actually call this, we will need to find a way to
generate proper `m_stub` functions and call them

## Generating Stubs

So how can we create a stub function from the actual function we want to bind?

Keeping in mind that we want this solution to work only for functions known at
**compile-time** gives us the answer: `template`s! More specifically:
non-type `template`s.

### Function stubs

So lets take a first crack at how we can create a stub for non-member functions.
We need a **function pointer** that has the same signature as our stub pointer,
and we need to hide the *real* function we want to call in a `template` non-type
argument.

Keep in mind that because we want a regular function pointer, we will want this
function to be marked `static` so that it's not actually a member function of
the class (which would create a pointer-to-member-function, which is not the
same as a function pointer).

So lets do this by making a `static` function template, that will simply
invoke the bound function pointer with the specified arguments:

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
    return (*Function)(args...);
  }

  ...
};
```

Now we have something that models the stub function, so we just need a
way to actually *bind* this to the delegate. Lets do this with a simple `bind`
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
    // Bind the function pointer
    m_stub = &nonmember_stub<Function>;
  }

  ...
};
```

Perfect; now we have a means of binding free functions. But it turns out there
is an even simpler way so that we can avoid having to write a `static` function
at all -- and the answer is **lambdas**.

One really helpful but often unknown feature of non-capturing lambdas is that
they are convertible to a function pointer of the same signature. This allows us
to avoid wiring a whole separate function template:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...

  template <R(*Function)(Args...)>
  auto bind() -> void
  {
    m_instance = nullptr;
    m_stub = static_cast<stub_function>([](const void*, Args...args) -> R {
      return (*Function)(args...);
    });
  }

  ...
};
```

Lets give this a quick test for the sake of sanity:

```cpp
auto square(int x) -> int { return x * x; }

auto d = Delegate<int(int)>{};
d.bind<&square>();

assert(d(2) == 4);)
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEJNJb0AGTy1MAOWMAjTMRAA2AAykADqgWE6rR6hiZmvv6BdDZ2jkYubl6KypiqQQwEzMQEIcam5koqanQZWQQxDs6uHt4Kmdm5YQX15baV8dVeAJSKqAbEyBwA5FIAzLbIhlgA1OKjOpgAHoM%2BxbRz2OKeAIJjE1OYs/MsCkrZG1u7O5PMp9PSzOgAIinAzASYOmys0yDTPgYnKw8MhpnV0CAQEsVmtZgB2WQ7AFAkEgS7TDHTZgGIjTADuCHeEC60zQtDq01oqGhmFW0wAtBtSXQKchCcQAFTTVAAN1cxDwM3ECPRmLFxEwBH6tFmkhkj2mWHYbw%2BpO%2BUkkc0R2zFwqelz1WoNOw%2BRh8wlVcx0BAAnj4tMwjIcGHhgLR3v1MBdrsI7i9le8vaNtZdTebA0drXaHU7pgAlUjTW3291OgB0Ge2xGACm92xuftegatcYgWZzGbTXTzwu1yOByDROwxAHoW9MdBLAwosTKDLQnH1tIqix9Rf7MCrMMSjk8R1RsawCEadqK29MAOqHPEiAhJhCHCdTpOoaYuZk%2BG3MIGYRMBWiDaaEHvA4AIAh4zCu9/j0fTskUkexaSO43IEAexAknMc5YAuBhLiuOpYjip6oPaxDvCQ0EQABe5AWOIFgRBJKMqM2DTPhXqEdB86LsuwbGkhYYWocJYQByABi/ZpHQXRltmCiVtWZGitiuJOLY6AzqR5E8qggqirWooYkYAD6th1CIj40bQ8GsKsxCIWKal1ICs5gpkajIKpJz0ToplOKpVDcWsGwQOIACssgeU8OEsnucmChyibloJGZZDmJFMnG8LamKGISlKxAyuxXEPmsfERWFVZGZierCXF8L6quzbIbiaGuJhxDEvxFbhQJJK4Qy0WKSKpUYngVDTBAJlSk4s46XpBlQW1SFiuBxCoHi9yPJRXysKwSlwvqDHtUVynTIl0rdRyvWAnxakaZkD63liAlCblholdsPgCjygZNlcSEGPewAWYCTkuUE5mlhyfFNYF6DBdMoUXatT0YgD8lA9Mh3ksd2mjHOukLQZWrTG2LZWiD/zyfQrgnvuhxHVphwQJ1T57kseB1Ao1alQ5n3pd9e39YNqMEIZwaYpj2M6n4tgfMQhPgYcznM3QhO2HJADWXqrstiGMYLYIAI4GFk04q7pRgjYVCWSttOvTFyOtK4r10q0YzC2DOSlrWJp7oOZlFWoL5P0MJmwior4NiugaYSdoVpSO4Cjq5rbkFYxYq3GcBAQFJkhQUjNHcNHCtPEMPSsCAQweUMpCmEMniF6gec6HIchgn0AysZIoycIXBB56XXQ9DLIAed4udDNwhfF6XpDl0MhcKCA3gtyX2ekHAsBIEsqQoWQFAQFlAAKIjKAwCBTcXTekGgZp4BaQSb3YrA73vreF0fPgn9UwCcJ45h3w/xAAPI4lfeKD4Xi/IFCnnf%2BixUgZHwMXQuNB6BMDYBwHg/BBDCFECgKuMghB4CcOPSAPQ0JrHHkMekH9Rhj1roMQQ4JbDn23rvX%2BecD54gwj4ehM9e4FyLjfYeedFgAA53D0ncNwaYwBkCgmfmmSQ0xsCgOQMvbquBCAkFlI3EklcZByGbjfdupBO7dyEHnfuHDp5cNHooCepAp5txznnSQA9OEj00dPbRfJiABA0NwIAA%3D"
>}}

Excellent -- we have something that works. Now onto member functions.

### Member Function Stubs

Member function stubs are a little more complicated -- because now we have to
work with pointer-to-member syntax and take into account both `const` and
non-`const` variations.

The general syntax for a pointer-to-member function is `R(Class::*)(Args...)`
Where `R` is the return type, `Args...` are the arguments, and `Class` is the
type that we are taking the member of.

So how can we get this into a stub?

The first problem you might notice is that it is not possible to use the
same syntax of `c.bind<&Foo::do_something>()` -- and this is due to the
`Class` type now being part of the signature:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...

  template <R(Class::*)(Args...) const>
  //          ^~~~~
  //        Where do we get the type name 'Class' from?
  auto bind(const Class* c) -> void { ... }

  ...
};
```

We still need to find a way to _name_ the `Class` type as a template parameter.
The simplest possibility is for us to just _add_ a `typename` parameter for
`Class`. Lets see what happens if we do that:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...

  template <typename Class, R(Class::*)(Args...) const>
  auto bind(const Class* c) -> Delegate;

  ...
};
```

Good -- so now we have a well-formed template. But notice anything different?

By adding `typename Class` as the first parameter, we can no longer simply call
`c.bind<&Foo::do_something>()` because the pointer is no longer the first
parameter! This effectively changes the call to now be:
`c.bind<Foo,&Foo::do_Something>()`.

It turns out that there is nothing, prior to `c++17`, that we can do about this
because we need to have some way of naming the `Foo` type first. This can
however be done with `C++17` using `auto` parameters (see
[part 2](/posts/2021-02-26/creating-a-fast-and-efficient-delegate-type-2) of
this series for more details).

So for now we will have to use the above approach for binding.

### Binding const member functions

Lets start with adding the binding support for `const` member functions. This
will be very similar to our function implementation, except now we finally get
to use the `const void*` parameter:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...

  template <typename Class, R(Class::*MemberFunction)(Args...) const>
  auto bind(const Class* c) -> void {
    m_instance = c; // store the class pointer
    m_stub = static_cast<stub_function>([](const void* p, Args...args) -> R {
      const auto* cls = static_cast<const Class*>(p);

      return (cls->*MemberFunction)(args...);
    });
  }

  ...
};
```

We finally get to use the `p` parameter and simply cast it back to
the `const Class*` type. This is safe because we know that this was what we
bound immediately on the line before.

Lets try this now, using `std::string` and trying to bind `std::string::size`:

```cpp
auto str = std::string{"Hello"};

auto d = Delegate<std::string::size_type()>{};
d.bind<std::string, &std::string::size>(&str);

assert(d() == str.size());
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAHYADKQAOqBYXW0eoYmgj5%2BanRWNvZGTi4AbLxKKhG0DATMxARBxqacisqYqgHpmQRRdo7Obp4KGVk5Ifl1ZRUxcSCJAJSKqAbEyBwA5FIAzNbIhlgA1OKjOpgAHoNeqXPY4u4AgmMTU5iz8ywKSlnrmzvbk8wn09LM6AAihcDMBJg6bKzTINNeBg5WHhkNM6ugQCAlitUrNXLJtv9AcCQBdpmjpswDERpgB3BBvCBdaZoWh1aa0VBQzCraYAWnWxLoZOQ%2BOIACppqgAG7OYh4GbiOGo9Ei4iYAj9WizSQyB7TLDsV7vYlfKSSObwrYiwWPC46jV67bvIxeYTKuY6AgATy8WmYRgODDwwFob36mHOV2Et2eireHtGmouxtN/sOlptdod0wASqRptbba6HQA6NNbYjABSerbXH0vf0WmMQDNZtMpro5wWaxFA5Ao7ZogD0TemOjF/oUGKlBloDj62nlBfewt9mCVmEJh0eQ6omNYBAN22FLemAHUDjiRAQEwgDmOJwnUNMnIyvFbmIDMPG/LRBtNCF2gcAEAQcZhna/R8PJySyQfC0keJOQIPdiCJOYZywOcDAXJctQxLFj1QW1iDeEhIIgP8dwAkcgJAsCiXpUZsGmXCPXwyDZ3nRdA0NBCQzNA4iwgNkADFe2KOguhLTMFHLSsSOFTFsQcax0CnYjSK5VB%2BWFathTRIwAH1rBaO9mNGGdaFg1hVmIeCRRUuoAWnUEMjUZBlOOWidBMhxlKoTi1hIiBxAAVlkdzHiwpkdxk/k2XjUt%2BLTTIsyIhkY1hTURTRMUJWIKVWI4u9Uh48LQorQz0R1QTYthXVl0bBNMBNJjw0TKMDh0b0FHjYtapuBQITZABZMq4lSrjaB4kKBMZUlaI2EqROPMTtF8oa2zqjlkEikjpgC9AYsU6YVLUjINLMiRA2mVc6hIA5QIOPMux8ax3gMkqlOU%2ByzJaSzrJuWz7Mc5yAnWNzPI8nzsKW2T0A5Lxgr48tMoW0jooUm70X%2Bsa5tYLsqMe4FnrqC1/qak42S%2Brx8vouLpgSyVpiwpGpPazrnG69KIEygScrRPKcv1YrLgQsbOVQ9DiEJXiyzCviiX%2BqTY3koVYbwKgyeMiUHGnKidNYPSCHA1bYbRUDiFQHE7gecjPhVhTXF1OjYbZhD4vFUnWLlgEeI2oaREGeMGbTAmSsti4vD5Ll/QbDm0QMW9gHMgF3rSgIzOLNkeP%2B5agumfqPfg4UE8BjknfU%2B8ld0/SNX2lsLWTv5ZPoZwj13A5Npdg4IGlh8dyWPA6gUSsSrepyo7oda7vlszldVgy9ubYv5lLi6K%2BIKuTumbueqr6wZIAaw9ZdTbT5dJHGO99nDOo%2BVEKttku9bmGsKcYatxDsUPh6CHBEBD%2BsYAFJlAAJZRWFQNV9XNoO6IuYrSouRC0YIIQv1EJAvAAAvTAykqqEnOEKTeACRToBTBNdA4DH6QLVq/eMUh4gQOfgQ6Bz84EBmwG5ICh9PaALRM1ZwBAIASQglpFGasUx%2BHgYSBhOohg9FYCAIY7khikFMEMdwEjUCiJ0HIOQoI%2BgDGYjvTgEiCCiJkV0HoK8QDuU8CIoY3AJFSJkaQORQwJEtU8Fo6RQjSBwFgEgJYRQkJkAoPTPiAAFEQygGAIF1lIjRpA0AmjwGaAIfibCsECcE7REjwleEidUYAnB3CSDCagCJ7BiAAHksTxJxOYiRbjkAhVEWUxYRR0j4CkRImg9AmBsA4DwfgghhDQIkDIOQQg8AOBapAHoKFUgtSGLSfJow6Rgkgj02QMgeA2JUYMQQYJrAxICUEkpojQk4jQl4XZjjjHiMkYkyxojFgAA54i0kSNMYAyAQQZJTJIaY2AanIA8WTXAhASDSlGJwIkCjekyE0Yk3RpB9GGKEKI0xZyHEXOsYoEAdiIXCNEZIMx5yrHgocZCnkxA/AaG4EAA%3D"
>}}

Excellent -- it works! Now we just need to do the same for non-`const` members

### Binding non-`const` member functions

The non-`const` version will look very similar to the `const` version. You might
notice one issue from the initial design: our `m_instance` member is a `const void*`,
but now our member pointers are not `const`! How can we work around this?

It turns out, this is one of the few cases where a `const_cast` is actually the
perfect solution. We know that the only `m_instance` pointer we bind to this will
be non-`const` by the time we bound it, so it's completely safe for us to remove
this `const`ness again. After all, we did only add `const` so that we had an
easy heterogeneous type to convert to.

Lets see what this looks like:

```cpp
template <typename R, typename...Args>
class Delegate<R(Args...)>
{
public:
  ...
  template <typename Class, R(Class::*MemberFunction)(Args...)>
  auto bind(Class* c) -> void {
    m_instance = c; // store the class pointer
    m_stub = static_cast<stub_function>([](const void* p, Args...args) -> R {
      // Safe, because we know the pointer was bound to a non-const instance
      auto* cls = const_cast<Class*>(static_cast<const Class*>(p));

      return (cls->*MemberFunction)(args...);
    });
  }
  ...
};
```

This looks almost identical to our `const` version, except we have the one
`const_cast`. Lets give this a quick test using `std::string` and
`std::string::clear` as a quick example:

```cpp
auto str = std::string{"hello"};

auto d = Delegate<void()>{};
d.bind<std::string, &std::string::clear>(&str);

d();
assert(str.empty());
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEAGZeW9ABk8tTADljAI0zEQAdl4AHVAsLqtHqGJuY%2BfgF0tvZORq7uXorKmKqBDATMxATBxqYWSSpqdOmZBNGOLm6evAoZWTmh%2BbWl5bHx1QCUiqgGxMgcAORSZnbIhlgA1OJmOpgAHv3eRbTT2OIADACCw6PjmFMzLApKWasb21tjzMcT0szoACLJwMwEmDpsrBMgE94Gzqw8MgJrV0CAQPNFssph5ZFs/gCgSBzhNURNmAYiBMAO4IV4QDoTNC0WoTWioSGYJYTAC0qyJdFJyDxxAAVBNUAA3NzEPCTcSwlFo4XETAEXq0KaSGT3CZYdgvN5Ez5SSTTOGbYUCh7nbXq3VbN5GbzCJXTHQEACe3i0zCM%2BwYeGAtFevUwZ0uwhuTwVr3dZg15yNJr9Bwt1tt9omACVSBMrTaXfaAHSpzbEYAKD2bK7e55%2B83RiDpzOp5MdbMCjUIwHIZFbVEAekbEx0or9CnRkoMtGcPW0cvzbyFPswiswBIOD0HVAxrAI%2Bq2QubEwA6vtsSICPGEPtR%2BP46gJq4Gd5LcwAZg4/5aP0JoRO4DgAgCNjME6XyOhxPiaT9wXJAANg5Ahd2IQlpmnLBZwMedF01dFMSPVAbWIV4SEgiBf23f9hyAkCwMJOkzGwCZcPdfDIJnOcFwDA0EODU19kLCBWQAMR7VI6A6YsMwUMsKxIoUMSxZw7HQSdiNIzlUD5IUqyFVEjAAfTsZpb2Ysxp1oWDWCWYh4OFFTan%2BKcQQyNRkGUo5aJ0EznGUqhOOWVYIHEABWWR3IeLDGW3GS%2BVZOMS341NMkzIj6WjGENWFVFRXFYhJVYjjb2WHjwtC8tDLRbVBNimEdSXBt40wY0mLDBNI32HQvQUOMi1q64FHBVkAFkyviVKuNoHiQoEhkSVotYSpEo8xO0XyhtbOr2WQSKSImAL0BixSJhUtSMg0syJADCYV1qEh9lA/Zc07Xw7DeAySqU5T7LM5pLOs65bPsxznMCVyPK8nzsKW2T0HZbxgr4stMoW0jooUm60T%2Bsa5tYTsqMeoFntqc0/qa45WVc7x8vouKJgSiUJiwxGpPazq3G69KIEygSctRPKcr1YqGLKkMzRmKqkxquqGogLGWpASm4mpj7uN40tU0EkaELG49xMF2aiQh/6%2BVWmGNqGkQ7yo3bpH2ltDtFHdTrq35ZPoNw1uM8VnAeizUZs803qctLPpItzPI836/PVwHfhB6Xk3B2kos1hDhRXBhmCoK9jxSDElBxfYAGtyWxM3LcutwcWuY9%2BxWrFmDJOgaT%2BzbdfdGHUXholEZ2vy0dsoWca9lGrJdw5/bb3GOnxtnCeJpLSbGBQKY6sXiBpwIMtBmXGcKweENZi55aQjlUPQ4gCSlrKw7%2BqSY3kwUYbwKhSbt0zIKonTWD0ghwMjwnQOIVAs7uR5vw%2BB%2BFI8HUdEYZrxFGKEmrFr7OB4trdS/Q4z00XkA1eAD6LeF5JyP09Z16ogMDeYA5l/jvQ9nQMyRZWQ8T%2BstIKEx%2BqIMDCVShAN2QwK2nrLSZJdL6XVEbRs5oaE52tsQQ82cq7bQgBfe8255h4FqAoCsJU3YS0lJAsy99H4GT2k2ZsfDNQXUEcIk6Ex3Y9WEXYGSaca7bBQUg3UkgRi3j2GGWovJRCVi2JddazA7CTmhlHRCWJnGOzBCAZxdhgAKWlLuB%2BqBVR6hsTDBWK0qLkXNMtAkZxBTWIKqidAyYJroFdgQYJoTRBxikIBUE4ISmgBQOwTIX0gLOJXmtCSK9hTNTcAQCAzjkwcytASZpACBhdFYCAAY7kBikFMAMdYkzUBjJ0HIOQIIeh9GYnYzgkyCBjNmQPUgacQDuXWEIMZ3BJnTNmaQeZAxJktWOdsmZwzSBwFgEgeYKQkJkAoHTPiAAFEQygGAIA/tMzZpA0DGjwKaQI/z7CsCBSCnZkyIXeChVUYAnB1iSHBagSF7BiAAHlMQIuxBcyZ7zkAhTGeSuYKR0j4GmZMmg9AmBsA4DwfgghhCiBQEsmQQg8DOBapALoKFlgtQGDSAlZhaSgkghIGQcgeC3NWf0QQoI7CwsBcC0lYywXYjQt4PVTzRnjPOUiq5Yy5gAA5AI0kAtwCYwBkDAkxcmSQExsC0uQJ80muBCAkClGYTghJFmKpkFspFeyDlHJOQMM5UyLXXNuSAe5UaRljMkOax5lqbmkAebsro3JiD%2BA0NwIAA%3D%3D"
>}}

And now we have something that works that allows us to bind free, member, and
`const`-member functions at compile-time!

## Closing Remarks

There we have it. Using a small amount of templates, we were able to build a
small, light-weight utility for calling functions bound at compile-time.

There is still much we can, and will do, to improve this design.

Check out [part 2](/posts/2021-02-26/creating-a-fast-and-efficient-delegate-type-2)
to see how we can **support covariance**, and
[part 3](/posts/2021-02-26/creating-a-fast-and-efficient-delegate-type-3) to see
this optimizes to have **zero-overhead**.
