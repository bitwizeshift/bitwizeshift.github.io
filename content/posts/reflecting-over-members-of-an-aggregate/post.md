---
title: "Reflecting Over Members of an Aggregate"
subtitle: "Implementing 'reflection' qualities using standard C++"
date: 2021-03-21T11:31:40-04:00
draft: false
author: "Matthew Rodusek"
tags: [serialization, reflection, templates]
languages: [c++, c++17]
categories: [complex-templates, tutorial]
---

A little while back a friend of mine and I were talking about serialization of
`struct` objects as raw bytes. He was working with generated objects that contain
padding, but the objects needed to be serialized _without_ the padding; for
example:

```cpp
struct Foo
{
  char data0;
  // 3 bytes padding here
  int data1;
};
```

In the case he described, there are dozens of object types that need to be
serialized, and all are:

* Generated by his organization (so they can't be modified), and
* Are guaranteed to be [aggregates][agg.init]

Being a template meta-programmer, I thought it would be a fun challenge to try
to solve this in a generic way using {{<language "c++17" >}} -- and in the process I
accidentally discovered a generic solution for iterating all members of
**any aggregate type**.

<!--more-->

{{<table-of-contents>}}

## Aggregates

Before I continue, it's important to know what an **aggregate** type actually
is, an what its properties are.

Simply put, an aggregate is one of two things:

* an array, or
* a `struct`/`class` with only public members and public base-classes, with no
  custom constructors

There's formally more criteria for this, but this is a simplification.

### What is special about Aggregates?

Aggregates are special for a couple reasons.

The first is that Aggregates cannot have custom constructors; they can only use
either the default-generated ones (copy/move/default), or be
[aggregate initialized][agg.init]. This fact will be important in a moment.

The second is that, since {{<language "c++17">}}, aggregates can be used with
[structured bindings expressions](structured_binding) without any extra work
needed by the compiler author -- for example:

```cpp
struct Foo{
  char a;
  int b;
};

...

auto [x,y] = Foo{'X', 42};
```

It's also important to know that an aggregate can only be decomposed with
structured-bindings into the exact number of members that the aggregate has, so
the number of members must be known before the binding expression is used.

### How does this help?

Knowing the above two points about aggregates is actually all we need to develop
a generic solution. If we can find out *how many members* an aggregate object
contains, then we will be able to *decompose* this object with structured
bindings and do something with each member!

## Detecting members in an aggregate

The obvious first question is how can we know how many members an aggregate
holds?

The C++ language does not offer any `sizeof`-equivalent for the number of
members, so we will have to *compute* this using some template trickery. This
is where the first point about aggregates comes into play: an aggregate can
only have constructors that perform [aggregate initialization][agg.init]. This
means that for any aggregate, we know that it can be constructed from an
expression `T{args...}`, where `args...` can be anywhere between `0` to the
**total number of members in the aggregate itself**.

So really the question we need to be asking now is: "what is the most arguments
I can aggregate-initialize this `T` from?"

### Testing if `T` is aggregate initializable

The first thing we need is a way to test that `T` is aggregate initializable at
all. Since we don't actually _know_ what the argument type is for each member,
we will need something that the C++ language can substitute into the expression
for the *unevaluated* type expression:

```cpp
// A type that can be implicitly converted to *anything*
struct Anything {
    template <typename T>
    operator T() const; // We don't need to define this function
};
```

We don't actually need to define the function at all; we only need to have the
type itself so that the C++ type system can detect the implicit conversion to
any valid argument type.

From here, all we really need is a simple trait that tests whether the
expression `T{ Anything{}... }` is valid for a specific number of arguments.
This is a perfect job for using [`std::index_sequence`][index_sequence] along
with [`std::void_t`][void_t] to evaluate the expression in a SFINAE context:

```cpp
namespace detail {
  template <typename T, typename Is, typename=void>
  struct is_aggregate_constructible_from_n_impl
    : std::false_type{};

  template <typename T, std::size_t...Is>
  struct is_aggregate_constructible_from_n_impl<
    T,
    std::index_sequence<Is...>,
    std::void_t<decltype(T{(void(Is),Anything{})...})>
  > : std::true_type{};
} // namespace detail

template <typename T, std::size_t N>
using is_aggregate_constructible_from_n = detail::is_aggregate_constructible_from_n_impl<T,std::make_index_sequence<N>>;
```

With this, we can now test _how many_ arguments are needed to construct an
aggregate using aggregate-initialization:

```cpp
struct Point{ int x, y; }

// Is constructible from these 3
static_assert(is_aggregate_constructible_from_n<Point,0>::value);
static_assert(is_aggregate_constructible_from_n<Point,1>::value);
static_assert(is_aggregate_constructible_from_n<Point,2>::value);

// Is not constructible for anything above
static_assert(!is_aggregate_constructible_from_n<Point,3>::value);
static_assert(!is_aggregate_constructible_from_n<Point,4>::value);
```


{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAGYAbKQAOqBYXW0eoYmgj5%2BanRWNvZGTi6SAOyKypiqAQwEzMQEQcamnMkqEbQZWQRRdo7Obp4Kmdm5IQV1ZRUxcSCJAJSKqAbEyBwA5FKu1siGWADU4q46BACeXpgA%2BgTEzIQKs9jiAAwAgqPjk5gzcwZqrIQLO/tHkmO0EwbTszrIdehYVHeH93ViAZVFMDrQFgQENZgDMErJDlNEVMCJgjF5hCjzvMllpmEYzgAVP4HJFTVDLDZEYhTAkQLpTNC0OqzaRTAD0bKmAHUzug6GAhgQpjZMOhkagpj9rGdIXgFFMqAZnsV7uIEgARFmqw60PGYBReZiDSWYTJ4Viw%2BEk5Go9HMTHvRbLXX4mmkZE4l1nACSCndTtx%2BNm6oAbqg8OhiYjAcChXKVsxgMBiJhgPbVoyY2kHOwVlRiKgjCtaCs8GiLfdSSApl8QCAqGwlGscWrZBqtf8ETby%2BmsQGvW6awR0HW/AAvVYEAB0M99UaHQJB8cTydT6ZWmfWsbwOdW%2BcLxdL5felaRBNIp%2Bjw7r1iwAA8VkoAI4GLSDd6%2BmdTnYXrtXkcgGGEZrO8WATAGEBEnCEBAegEC%2Bj0YIQlCoitmq6pdF%2B6FdPOOxTNWtYgFuk4tnC6EdkcGrspyXoGkavKmpsrDagcKI9g6cz9nqg6EeOk5TLYxIGH4ohTMuSYpmmKIbnQWZqLueYFkWtDnOqJpmqwN4KAmElrtJm6LvJub7spR7ou856EUYzAANarLemAPs%2Br7PJg7yCa4uyeRRAJbiCAAK4b0K2Yn0FMd4sqFQq3K4rLkbFLEclMvoMrJflGWcJnIgg%2BpnK4vn2ngyAJgoSjZBA4mrlJGZpYZO7GUpxbvIF1gEKQew7HWIZsK%2BOEJYcLRqMVzClc4BAVdpK6SeuBnbgpJlNXMLX0KQnCdYBPWYH1VqDUVJVleNlXTfptVzQ1B60M1QVtZI63dYYW0%2BYcSUpbQqBCrN2bsAqJBTCIyHQn9DioCGbkDZkQ37WNEBgGAR16TVTLpfVe6NZdS3XaQ%2BWeV1m3bQVkMjQdMNw5NunVTJSN1fNaNXa1pDcHdeMUUMPSaUMACsQykKYQx7NzqAgEMOhyHINZ9AMZyjJw3MEEL/NdD0NkgBzexCEL3Dc7z/OkILQzcwoIBq3LfOs6QcCwEgjI0MQ1mueQlBoGi5rOIbsBO14LvEN6I4TCIwBrECtA2brXjFIbQwALRfMGEgyHInAJGbEAe17PsgMACi6l4CgIO9ofh0L0fDrHosyInyep%2Bw3sjiGyBeF4KwhpwACcKwYvqBArHe7i8OShdDGytZx7I5dJ4rpCioQJARoIwiiOnI9yEIO5u3SbNC1zPPywLQt6LQNt28aIZ4JgADuzhTBAuDT9S0v0q4bKcDrJsK0rKtq%2Bzmvb6butCwbRtSCv1ZhvIYkgtY7z/vrIBO8J6g2IH4DQ3AgA%3D%3D"
>}}

### Testing the max number of initializer members

All we need now, is to test the _max_ number of arguments that an aggregate can
be constructed with.

This could be done in a number of ways:

1. Count iteratively from 0 up to the first failure,
2. Count from some pre-defined high number down until we find our first success, or
3. Binary search between two predefined values until we find the largest scope

The former two options grow in template iteration depth based on the number of
members an aggregate has. The larger the number of members, the more iterations
are required at compile time -- which can increase both compile-time and
complexity.

The latter option will be more complex to _understand_, but also guarantees the
fewest number of template instantiations and thus should reduce overall
compile-time complexity.

For this part, it turns out that [@Yakk][yakk]
on Stack Overflow already provided a [brilliant solution][binary_search]
doing exactly this (modified slightly for this article):

```cpp
namespace detail {
  template <std::size_t Min, std::size_t Range, template <std::size_t N> class target>
  struct maximize
    : std::conditional_t<
        maximize<Min, Range/2, target>{} == (Min+Range/2)-1,
        maximize<Min+Range/2, (Range+1)/2, target>,
        maximize<Min, Range/2, target>
      >{};
  template <std::size_t Min, template <std::size_t N> class target>
  struct maximize<Min, 1, target>
    : std::conditional_t<
        target<Min>{},
        std::integral_constant<std::size_t,Min>,
        std::integral_constant<std::size_t,Min-1>
      >{};
  template <std::size_t Min, template <std::size_t N> class target>
  struct maximize<Min, 0, target>
    : std::integral_constant<std::size_t,Min-1>
  {};

  template <typename T>
  struct construct_searcher {
    template<std::size_t N>
    using result = is_aggregate_constructible_from_n<T, N>;
  };
}

template <typename T, std::size_t Cap=32>
using constructor_arity = detail::maximize< 0, Cap, detail::construct_searcher<T>::template result >;
```

This solution makes use of `template` template arguments which reuses the
`is_aggregate_constructible_from_n` above to find the largest number of members
that can construct a given aggregate from between `0` to `Cap` (default `32`).

Testing our solution with the above `Point` type:

```cpp
static_assert(constructor_arity<Point>::value == 2u);
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEAFZeW9ABk8tTADljAI0zFzATlIAHVAsLqtHqGJua8vv5qdLb2Tkau7mZeSipRtAwEzMQEwcamForKmKqBGVkEMY4ubp6Kmdm5oQUK9RV2VfE1SQCUiqgGxMgcAORSAMx2yIZYANTiYzoEAJ7emAD6BMTMhArz2OIADACC45PTmHMLBmqshEt7hyeSE7RTBrPzOsgt6FhUD8dHi1iAZVDMjrQlgQEHZgHMAOyyY4zFEzAiYIzeYToy6LFZaZhGC4AFQBR1RM1Qqy2RGIM2JEG6MzQtBa82kMwA9JyZgB1C7oOhgYYEGb2TDoNGoGZ/dpomEKGZUAyvNKPcTwgAi7PVx1ohMwCm8zCGMswmTwrARSPJaIxWOYOM%2By1W%2BqJ9NIaPxbouAEkFJ6XQSifNNQA3VB4dBklHA0GivAKNbMYDAYiYYCO9YsuMlZzsNZUYioIxrWhrPCYq2PCkgGY/EAgKhsJQbfEa2RanWA5F2qtZ3FBn0e%2BsEdCN/wAL3WBAAdPP/THRyCwYnk6n05n0Wsc5t43h8%2BsiyWyxWq58a6jiaRL7Gx427FgAB5rJQARwMWiGn3989nexvXs73HEAIyjDZPiwKYgwgUlEQgMD0Agf1eghKEYVEDsNU1bo/2w7olz2GY6wbEA9xndtEWw7sTi1LkeR9I0TQFc1tlYXUjnRfsnQWIcDRHUipxnGYHDJAx/FEGY1xTNMMyzHc6FzNRD0LYtS1oS5NTNC1WAfJMZM3eTdxXZSC2PdSzyxT5r1IoxmAAa3WR9MBfd9P1eTBPlEsZ9h8mjHkY41TSwHTrUvLiHR4nRBLwacNhmABZOxPRiuLRQAJREYBMEDe1sQuT5UuE7zsGZYQFEVepsoIJclJmOyn0rWLPKA4jRxAll8DSNgIIWW8KXq5hGqMZrPiS2hPUy0RME5SRAyyaqHioujQ1DGYIHGuQpuy2bugAWk4QDbQGwbhtGhZNpkbaZrm9brrkThulm%2BbiEWnyjpOlEGqa6cxuSmZruetEFvNJcKSWzttTGG0UQi/LcSK%2BLxty7iCoWRHRRKsrmAq4HXtBvzezq76Rt%2Bi7/sOvG3v2VqSPvFA6C6wIepqvrWopKqCZ0Ta/OWzUPs%2B0i7HRNMet3ERWei%2BmhI2UgeewAWTqF%2BgMy2VgFNZTJ6EK6Xmtl8aDrB1EIeo6HwrygcdZAmXRWRvtIrRqXrb1zGiKmHHKpB1madtYmhp%2BzzyYmmYDhe6n%2BrpkDhdVsXFK1yWMbluxDcJ20sK7M2e1tOHLd471%2BNJVO7xM5k45M19MCyZAEDcMLWpz9ErYnF2RKN8TYRmdMFAMVhRTW6SNzk7djP3FTzLLazPRKmiUVNm1qKzhvHb491r3a5u0pmHRmG8UMxkkMSJLhEfVBIZNiDuTTtLYxsSfOnQQ89bfvE9EKb4ZzXy6UKua%2BIay9kbEvTuhoe59z8pnE4Wc6oAAVIza0RFJegMwnzskQaKe40MERQ3nscFojo8DIGTBVNwBAIAn1pOfO4nxYHCwAaBNgn5NJrUkAYAi0Nhi9F0sMMwwxSCmGGAcXhqAQDDB0HIOQ9Z%2BiDAKs8TgvCCAiMEd0Xo9lzChy4dwXh/DBGkGEcMXhCgQChwUQIjhpA4CwCQCyGgxA7IeXIJQNAmJLRuEMbAJx3gXHEF9OOd2ogNggloPZXR3g0iGOGHtH4oYJAyAevCMxZCSyePYN48cwAFD6m8AoBAqACAhLCSIyJY5oniJkJweJvREnOJST40CyBvDeDWGGTgHg1j5RaGsJ8AA2XgVICnDE5A2GJsgykVN6BKQgJAoyCGEKIWpwy5BCAPG4xknCRE8L4YooRIi9C0BsXY00YY8CYAAO61wgLgSZdJxiPRmGMTknAdEmKUSotRQgRGaM2aY3RIiDFGNIM8jhazhiSC0Vsn5%2BiAVbOUaQMMrjAggG4EAA%3D%3D%3D"
>}}

## Extracting members from an aggregate

Now that we know how many members a given aggregate type can be built from,
we can leverage [structured bindings][structured_binding] to extract the
elements out, and perform some basic operation on them.

For our purposes here, lets simply call some function on it in the same way that
`std::visit` does with visitor functions. Note that because structured bindings
requires a specific number of elements statically specified, we will require N
overloads for extracting N members:

```cpp
namespace detail {
  template <typename T, typename Fn>
  auto for_each_impl(T&& agg, Fn&& fn, std::integral_constant<std::size_t,0>) -> void
  {
    // do nothing (0 members)
  }

  template <typename T, typename Fn>
  auto for_each_impl(T& agg, Fn&& fn, std::integral_constant<std::size_t,1>) -> void
  {
    auto& [m0] = agg;

    fn(m0);
  }

  template <typename T, typename Fn>
  auto for_each_impl(T& agg, Fn&& fn, std::integral_constant<std::size_t,2>) -> void
  {
    auto& [m0, m1] = agg;

    fn(m0); fn(m1);
  }

  template <typename T, typename Fn>
  auto for_each_impl(T& agg, Fn&& fn, std::integral_constant<std::size_t,3>) -> void
  {
    auto& [m0, m1, m2] = agg;

    fn(m0); fn(m1); fn(m2);
  }
  // ...

} // namespace detail

template <typename T, typename Fn>
void for_each_member(T& agg, Fn&& fn)
{
  detail::for_each_impl(agg, std::forward<Fn>(fn), constructor_arity<T>{});
}
```

We simply use `integral_constant` for tag-dispatch here for each member, and
forward a function `Fn` that is called on each member. Lets test this quickly:

```cpp
int main()
{
    const auto p = Point{1,2};
    for_each_member(p, [](auto x){
        std::cout << x << std::endl;
    });
}
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEJIAMpLegAyeWpgByxgEaZiXcwGZSAB1QKhOq0eoYmZpb%2BgWp0dg7ORm4enN6KypiqwQwEzMQEocamFmkqMbTZuQRxTq7unj4KOXkF4cWNldUJSfUAlIqoBsTIHADkUl72yIZYANTiXjoEAJ6%2BmAD6BMTMhArz2OLmAILjk9OYcwsGaqyES3sHx5ITtFMGs/M6yI3oWFT3Rw9GsQDKoZodaEsCAh7MA5gB2WRHGbImYETBGXzCNEXRYrLTMIznAAq/0OKJmqFWWyIxBmRIgPRmaFojXm0hmAHoOTMAOrndB0MAjAgzByYdCo1AzX72c5QvAKGZUAwvMoPcRwgAibPVR1oBMwCl8zGG0swOTwrHhiLJqPRmOY2I%2By1W%2BsJdNIqLxbvOAEkFJ6XfjCfNNQA3VB4dCk5FAkEihVrZjAYDETDAR3rZlxzIudhrKjEVBGNa0NZ4DFWh7kkAzb4gEBUNhKDZ4jWyLU6gFIu2VzM4oM%2Bj11gjoBuBABe6wIADo5/6YyPgaDE8nU%2BnM2ts5t43g8%2BtC8XS%2BXKx9qyiiaRz7HRw37FgAB5rJQARwMWmGH39c5neyvPZvMcQAjKMNg%2BLApiDCASQRCAQPQCB/T6cFIWhUR2w1TUeh/TCekXPYZlresQB3ac2wRTCu2OLVOW5H0jRNflzW2VhdUONE%2BydBZBwNYdiMnacZkcUkDECUQZlXFM0wzNEtzoHM1H3AsixLWgLk1M0LVYO8FCTKSN1k7dl0U/ND1Uk9MQ%2BS9iKMZgAGt1nvTAn1fd8XkwD5hK8fZvKoh56ONU0sC061zw4h0uJ0fi8CnDYZgAWXsT1otikUACURGATBA3tLFzg%2BFLBK87AmWEBRFSaLKCEXBSZlsh8KxijyAMIkcgOZfAyjYMCFmvck6uYBqjCaj5EtoT0MtETAOUkQNciq%2B4KJo0NQxmCAxrkSaspmnoAFpOH/W1%2BoGoaRoWDaZC26bZrWq65E4HoZrm4gFu8w7juRerGqnUakpmK6ntRebzUXclFo7bUvBtZFwrynFCrisacs4/KFgRkVitK5hyqBl6Qd8ntaq%2B4afvOv6Dtx179haojbxQOhOuCbrqt6lryUq/GdA23yls1d6PuI%2Bw0VTbrtxEFmorpgSNlIbnsH547BfodMtlYOSWRyegCqlpqZbG/bQZRcHKKhsLcv7bWgOlkUkd7CLUclq3dYxgipmxirgZZ6nbSJwbvo8snxpmSxKfx72azau9lZFtWxa1tGddS2X7ANgnbQwztTe7W1YYt7jvV4kk05vYymXkndVGfTBcmQBB3FClrc7RS3x2doTDdEmEZjTBQDFYEVVsk9cZKzcvjL3UyVNLKzPWKqjkRNm1KOzpuHZ491L0jkBrZmHRmF8UMvEkESxNhIz4xIJNiFudTNJYhtibOnRg89PffE9YL7/pjXjKrmu6%2BIFZPYDZV7d0NH3Aevks7HGzgFRid9LQNxzubSK69iSBgLu6AAYrQRczArhShoMQNY1da4WVYNBKQAA2ahMw1yehwdQ2hVAg5K2FqrdW7R46O1bkncwexGS7QIvBc8GEWpcmlFKWgqB5TiQgOYOq6IkgKDwj2ZeMDkEowHJg9BXpXS8UYcXOhBClSX1IQgchlDJBULoSmBhuDrHMNYXTIWKtRbl3Fi3beztSCcAETMIR3kZgiLURRFq%2BCiC0PEAAVmkEYfh0SNKrTXH5FqLCIDxLwtAhenZs4wxQWvHRw40EzEMeHYxRBTHEPMZYkk1jbHAHsUw%2BpLDkouOjhwuOEt0akGPt5QRwjIzRlCdDFEETUBRNifEz0Rg/GJNvik6BfV0mZLZEqWgGSHrz3hNqPJds4bOiKZvEpZTzzjKqSQk0FiKyYisTY%2BhpSHE0Jac4oCriY6cM1t0xO05SBeH8YEkqIT05hKOhUiZ9SYlxJDrMmZx95nJJTKksFKzzBZPZCsrZUN1kZMkOi0RuTbQSJwtnTCtFRQGgYkFZilo2KgMOfojeGDGXnFOUceCFyamEkSO4O5DSmmOJeao6iozP6WgbEQy5ZCbkUIecRIhAB3XI0YFhlOoLQPoZcf4X2IbkW4QCeYQ3xQCQltUAAKkYtYIgkvQGYD41lCxmHcbFi82InBeGcHE6ggTVyMKSB4jrbL2AZOqUF5JtzgpmL4W%2BFqhbtgOsfTOozkSSq5Uo3l785ixJiZqCA5yHx4TDQLOmaArg4g%2BHa8tCwt7WFYtko2WpjXUU1CMPo2kRjRJGKQUwIxzBdtQCAEYOg5ByDrAMIY%2BUnicC7QQQdfaeh9DsiAaJlh23cC7T2vtpAB0jC7QoEAlhZ29tbaQOAsAYCIG/kQ2y7lyCUDQBiS07h92wAfb4J9xBfRjjdqIDYwJaB2W3b4Mo%2B6Ri7W%2BKGCQMh7pwhPRAN9H6v0gGAAofUvgFAIBkUBkDg7wOjkgyOmQnBYN9Hg8Wd97BP1jjDMgXwvg1hhk4AATjWHlRoawHxUN4JSHDIwOT1ig7IIjJG%2BjikICQKMghhCiCQ4JuQQg9wvoZG2wdnbu1zv7YOvQtBr0iFNGGPAmAFX1wgLgcTtJxgPRmF4DkKQZ0aYXaQJdK6hCDvXaQIwy7LCbs07uxQB7SBHvnaei9EAkDOQyAQsgFA80vQUGakQygGCYYVT26dpAENYmCAlhwrBkuoFSxpjL5GP3IZSLNTL7gADyVx8uFePVYB8GRDhxcHV2iLyBsj4B7V2mg9AmBsA4DwfgUnMooEI/IG4LglN9B48EUDeGVWajk8Jvd47hiCG%2BPYHLSWUtpa7QqrYvhB3TpUx2jdRWd0PgABxUN2lxmYwBkDIBmCkGckgZjYCa8gKLa0zM0jmFOxkw7oMyHs8exzddmBYA8AyVzIx3OeZcz57dbX/OHoc4urz8OvAXYazu8H86zuSDx1ugngXMekDDM%2B4IIBuBAA%3D%3D"
>}}

This effectively gives us a working solution.

## Back to Serialization

Lets tie this all together now by brining in serialization. Now that we have an
easy way to access each member of a `struct`, serialization just becomes a
simple aspect of converting all members to a series of bytes with a simple
callback.

If we were ignoring the endianness, then serialiazation of packed data can be
accomplished as simply as:

```cpp
template <typename T>
auto to_packed_bytes(const T& data) -> std::vector<std::byte>
{
  auto result = std::vector<std::byte>{};

  // serialize each member!
  for_each_member(data, [&result](const auto& v){
    const auto* const begin = reinterpret_cast<const std::byte*>(&v);
    const auto* const end = begin + sizeof(v);
    result.insert(result.end(), begin, end);
  });

  return result;
}

...
auto data = Foo{'X', 42};
auto result = to_packed_bytes(data);
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEJIDMpLegAyeWpgByxgEaZiXc51IAHVAsJ1Wj1DEzNLPwC1OjsHZyM3D04vRWVMVSCGAmZiAhDjUwtUlWjaLJyCWKdXd09vBWzc/LCihoqq%2BMS6gEpFVANiZA4Acilze2RDLABqcXMdAgBPH0wAfQJiZkIFOexxAAYAQTGJqcxZ%2BYM1VkJF3YPji1ODGbmdZAb0LCp7o4eG4gGVTTQ60RYEBD2YCzADssiO00R0wImCMPmEKIuC2WWmYRnOABVfockdNUCtNkRiNMCRButM0LQGnNpNMAPRs6YAdXO6DoYGGBGmDkw6GRqGm33s5wheAU0yoBloGToD3EMIAIiy1UdaHjMAofMwhpLMNk8KxYfCScjUejmJi3ksVnr8TTSMica7zgBJBQe524/FzDUAN1QeHQxMRAKBQrlq2YwGAxEwwAda0ZsYyLnYqyoxFQRlWtFWeDRloepJA00%2BIBAVDYSnWOPVsk12r%2BCNtFYzWMD3vdtYI6HrAQAXmsCAA6Wd%2B6PDwHAhNJlNpjOrLMbON4XNrAtFktlitvKtIgmkM8xkf1%2BxYAAeqyUAEcDFohm8/bPp7tL93r6OIDhpG6xvFgkyBhARJwhAwHoBAfq9KC4KQqIbbqhq3Tfhh3QLrs0w1nWIDblOrZwhhnbHJq7Kct6hrGryZpbKwOqHCivaOvMA76kORETlO0yOMSBgBKI0wrsmqbpiim50Nmah7vmhbFrQFwaqa5qsLeCiJpJ64yVuS4KXmB4qce6JvBeRFGMwADWax3pgj4vm%2ByqYG8QnmHsXmUQ8dFGiaWCaVaZ7sfanE6HxeCTus0wALL2B6UUxUKABKIjAJgAZ2hi5xvMlAmedgDLCAo8qNJlBALvJ0w2fe5bRe5/4EcOgGMvgpRsKB8xXqStXMPVRiNW8CW0B66WiJgbKSAGOSVfc5HUSGIbTBAo1yBNmXTd0AC03i9aSdUNZOI32BtGVTTNq2be5MicN002zcQ81eX%2BNp9f1g3DfMo3jRdj3InNZoLqSC3tlq5jWoiYW5ViBWxb9PbhXl8zw0KRUlcwZWA89wM%2Bd2NVHUNJ0/Yl0zeDjL17M1hE3igdAdUEXVVT1zWkhVeM6OtPmLRqb0fQBt70GmmysLJTLZPQ%2BV0/x6ykNz2D8wLRH2CiKZdVuIgs5FMuNXLo17SDSJgxRkOhTlfbS4BstCojMOW6juspYJ%2BGTFj5VAyz1M2oTA3He5pNjdM%2BxPVTvW04BqsixrcmS9raPy/Yhv4za6EdmbXY2vbEXcW6RIp9eRkMrHRlPpgOTIAg7ghc12cBzr1t6%2BjRsiVC0ypgoBisEKK0SWu0mZiXO6KaZJaWR6RWUYipvWhRmd1/2Xo8RerVjk30w6MwPghuYkjCaJ0KGXGJCJsQtxqRpzH1kT306MHHqbz4HpBVf9MS6XSgV1XxCWbs9YLx3LuPcfIZ2OJnfyDFL4WhrlnC2Ocl55wDAg84AAxWgC5mBXAlDQYgqxy6V3MqwKCUgABspDpirg9Gg0h5CqBBxVsLdWYtNZS0do3FKpB9i7HpDtfCcEzzoWahySUEpaCoFlGJCA%2BxaqokSAoXC3Y55gNgRxFG2IXTLyQRot01CC4UKwQqE%2B%2BCECEOIZIEhFDkxUPQeY2h9C6ZRyYeLNorCG5rw4Zwbh0xeFeWmPwxR5FmqYKIOQ8QABWaQRguFhPUitVcvlmp0IgFE3CoDp4dkztDOBajc6Ei0UGVB6C9HBOwUY40JjyzojMRYyh0xqG2PMQqexkdGGi2cXHK27ipykD3l5HhfCIxRgCVDJEJTQkRKiR6IwniYkX3iaA3qSSUksiack%2B6U9YRakyUjWGTpkFDlyXUop3tEQlMMbg4xpiiSNNqfUshjS6FJQca0mOEstadJADbUg5gvE%2BOKv41OgT3r6JCY08JkSQ61QpkYPesy4nJgScCpZ%2BxUmsiWesyGqyYWooERkm0wjsKZwwjRYU%2Bp6KBSYhaViC89naLyZ6OlRziRwXOXg8pqx8QJHcNUyxwBrE0IebQBRVERkvwtPWHBbKCGVKIbUoiOCADuOQozzF0dgagQqPRH1UCfHItxf483Bjiv4eK1RPGVGcLEoZ0hUnwiSoi1qdU/z%2BOayYLw1EfBHN8fCwiiIuHBE1ZRNKuL7Pzt7M5RBVgBXsugVY/qUQKAgFuGk5D0AOmYP03xDqbUkA%2BfGgO3tBE2jOYA7uF9s1OrzQGk26dZ7dl9e4PAbBGrTGMTIrlxAwBgDPJKy5nLEgQDTdkD04LSGlqqjEpNscQWoHIaGXCQLSTJpKQAKmLhLaYbhgD2AvqmKOxAfCpgIJuLG2tk1%2BoDSu3YEBSHzo2YiZdWC13JusBfLdO65C1kaqgKgsFjXAvHdOewShcgQEA9YOkHp31B2sP%2B9JmFEWIiPQMVS47fKmpdeMC17qsTqABOXIwxITjYdePMdQNlaB4G3inf425gQoNQBKNsDIEA5AZCs1W4kVkz1Ypxmy9g6RqkXaMgxQ7mAXwY7OuEAoAAaAoPTcD3rW3qJaDRAIvpG6Noo40BsTWJ/9vUiJoCuFiN4sxJCSAAKq0C06OczkhTPzFXiAawLE0lIkfUQZ907oO7swPuw9ZoT3MnmOeum%2Bar1eRveY/T9710NBnd5jdr6Vq%2Bc/fxH9EBJNwaRDg1aZzCBvrTGdTFhWu0rWsNxmQchCD0iLQLZzxngE6DM0RKu95HN3zaGoZAwXtZKgCMABwKr1Urtq516YApJujHc/BwzdMmsTaIq5xDSIjP9Ga2ZqQkgAAKxoY01m20tumK3ZvnPywYlwLVx0LpGR9dbJnTxOba05Y7Do8C9ZYCFnQA28BDdFNelwdWepOamwKOLSiMLDF6FpYYYThikFMMMfYCPUAgGGC16rMhaz9EGHlCwnAEcEHRyj7ovRbIgDCSHWH3AEdI5R6QNHwwEcKBACHYnyPoekDgLAGAiA344Io0McglA0BogtO4VnsAxc%2BAl8QH0o43aiHWICWgtlGc%2BFKKz4YO1PghgkFj6QnAYRc6neL9g8vRzAAUHqHwCgEDiI11r9HuuRz67kHIY3puZdy4V0BZAPgfCrFDJwAAnKsXKDRVj3hIbwckzvhhsjrAb2Qd0Tdk6sB1EgkZBDCFEH7lPcghC7il3SGH6P4eI5J6j9HehaCC5ECaUMeBMAKurhAXAhASDma8PScwbJOAM456T8nlPqfo9p6QIwY%2Bq%2Bc8Z%2BjlnbPSDD9N3ziASAnLpCwWQCgEA5oKD2w4VgDAHcKqR4T0gPuMRBEP8oE/qAz/V8v0WWXFuQDAEHzNK/7gADyVx7%2BP5z6b7ICHDPTa4I7AFZD4BI4I40D0BMBsAcA8D8C54ZQoAe4yDF4uCl69Dx5BDa6u4qoaiF5p4s645DCCCfD2C37H6n7n4I4KqbA%2BDo6E7l5w505P5M73gAAcJCO0se0wwAyAyA5M%2Bw04Dm2A94W%2BVIq0neMhYw90G8GB0gRO1eGeVczAWAHgkGNOCO0%2BVOs%2BDOTOi%2B7Oaho%2BBhsO5gHBc%2Bxhy%2BZhQg6Okg1hRhC%2BdhnOGe1qxAAQGg3AQAA%3D%3D"
>}}

This is much easier than having N serialization functions defined for each
generated object. All thats needed for this solution is the high-bound of
`Count` in the detection macro to increase, and to have `Count` instances of
the `for_each_impl` overload mentioned earlier.

## Closing Thoughts

This gave us an interesting solution to "reflecting" over members of any
aggregate in a generic way -- all using completely standard C++17.

Originally when I discovered this solution, I had thought that I was the first
to encounter this particular method; however while doing research for this
write-up I discovered that the brilliant  [`magic_get`][magic_get] library beat
me to it. However, this technique can still prove useful in any modern
codebase -- and can be used for a number of weird and wonderful things.

Outside of the serialization example that prompted this discovery, this can
also be used in conjunction with other meta-programming utilities such as
[getting the unmangled type name at compile time][type_name] to generate
`operator<<` overloads for the purposes of printing aggregates on-the-fly.

### Possible Improvements

This is just a basic outline of what can be done since this is a tutorial
article. There are some possible improvements that are worth considering as
well:

* We can propagate the CV-qualifiers of the type by changing `T` to `T&&`, and
`auto&` to `auto&&` in the bindings (which will then require some more
`std::forward`-ing)

* We could detect the existence of specializations of `std::get` and
  `std::tuple_size` so that this works with more than just aggregates




[agg.init]: https://en.cppreference.com/w/cpp/language/aggregate_initialization
[structured_binding]: https://en.cppreference.com/w/cpp/language/structured_binding
[void_t]: https://en.cppreference.com/w/cpp/types/void_t
[index_sequence]: https://en.cppreference.com/w/cpp/utility/integer_sequence
[yakk]: https://stackoverflow.com/users/1774667
[binary_search]: https://stackoverflow.com/a/39779537/1678770
[magic_get]: https://raw.githubusercontent.com/apolukhin/magic_get/develop/include/boost/pfr/detail/core17_generated.hpp
[type_name]: https://bitwizeshift.github.io/posts/2021/03/09/getting-an-unmangled-type-name-at-compile-time/
