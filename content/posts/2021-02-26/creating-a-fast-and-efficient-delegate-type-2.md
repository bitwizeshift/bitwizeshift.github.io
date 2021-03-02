---
title: "Creating a Fast and Efficient Delegate Type (Part 2)"
subtitle: "Upgrading Delegate to C++17"
date: 2021-02-26T22:18:32-05:00
draft: false
tags: [templates]
categories: [intermediate-templates, tutorial, c++17]
series: [delegate]
summary: "In the previous post we saw how we could build a simple and
light-weight `Delegate` type that binds free functions, and member functions;
however there are several notable limitation. Lets expand on this and make it
even better using `c++17` features.
"
---

{{<info>}}
**Note:** This is part 2 of a 3 part series.
{{</info>}}


In the [previous post](/posts/2021-02-24/creating-a-fast-and-efficient-delegate-type),
we saw how we could build a simple and light-weight `Delegate` type that binds
free functions, and member functions. However we have a notable limitation that
we require specifying the _type_ of the members being bound
(e.g. `d.bind<Foo,&Foo::do_something>()`). Additionally, we're forced to bind
only the _exact_ type. We can't bind anything that is *covariant* to it.

Lets improve upon that.

{{<table-of-contents>}}

## Goal

To improve upon our initial `Delegate` implementation from the previous post.

In particular, we will support both covariant functions and improve upon the
`bind` function in the process. For this improvement, we will require `C++17`.

## Supporting Covariance

### Reworking Free-Function Binding

So how can we support covariance?  To do this we need a non-type `template`
parameter that supports *any* function pointer signature that may be similar.

This is where `C++17`'s `auto`-template parameters will play a huge role.
`auto` parameters are non-type parameters that use `auto`-deduction semantics.

Lets try making this change. While we're at it, since we're using `c++17`, lets
also update the function invocation to use
[`std::invoke`](https://en.cppreference.com/w/cpp/utility/functional/invoke) in
the process:

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
auto square(long x) -> long { return x * x; }

auto d = Delegate<int(int)>{};
d.bind<&square>();
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEJM6kt6ADJ5amAHLGARpmIgA7AE5SAB1QKhOq0eoYmZhb%2BgWp0tvZORq7u3orKmKrBDATMxAShxqbmqSoxtFk5BHGOLm6ePgrZufnhRQ0VVQlJdQCUiqgGxMgcAORSAMx2yIZYANTiYzqYAB5DvqXz2OIADACC45PTmHMLLApKuRvbe5ITtFMGs/M6VAZ3pWyXu1dTzGcz0sx0AARNLAZgETA6NisGYgGa%2BAzOVh4ZAzBroEAgZarUpzDyyXYIpEokBXGbkmbMAxEGYAdwQ4Ig3RmaFoDRmtFQ2MwaxmAFoNiy6OzkAziAAqGaoABubmIeEe%2BLJFJVxEwBAGtDmkhkgJmWHYYIhLOhUkk8wJOxV4g8QKuNrtY0tVwhRl8wmNTwIAE9fFpmEYjgw8MBaOCBphPjsfn8QYbwZGnfbdq73QnjjofX6w4GZgAlUgzLP%2BwMAOnLO2IwAUUZjChmccwRsTOjzEEr1fLpe6UZtlqJyOQpN25IA9KOZjo1Qn6yIZq9nP1tPrQQnlY3m0zjkCV1QqawCBbk1aZuOZgB1I60kQEIsII4b9M01xC3ze5hIzCFwJ3I6EevIsACAELSmAhsB66rhCECsuyj4QlIABsUoEPexDMvMO5YHuBgHkeI6UtSqBSn6xDgiQmEwcKt7wZGkjIagqFuMyApjNgDZQXRyGYbu%2B6HkmXwEamHpHE8VI0gAYq8GR0FG5LicRzh2OgW6sex0qoAqyp9sq5JGAA%2BnYbS/tuHK4awazEPhJ56fpDSIqZbRqMg%2BmnPxOj2c4%2BkvG8wQbBA4gAKyyIFQJUWyt4aQq4qFh2Chdjk1YsYKeY6QRqrqpqaIEBiIB2BpADWmAQFJvl0IWiXxeWPYCTZeJAjVlrkg6x7ksJ6Zer6JZHDowhnIWba9b8CiYuKACymCJG4pUybQ3TtlWVXdkKEVyYRz7KeF7JDWckrIMlbEzFF6B4k1FIGUZ2QmTxEhOqeE4NCQRxMSyfX1v4dgQlZ6UzAZnmOdkzmub87med50nrGxAXBUFYWwZFmnoJKvixYtCWLQd7F5qdukUvD62oHtrD1jxTkosDDRPPjO0KOK/m%2BI1rUqjMaoasQWowcTanjZNSQzaU82VV2jM/Q6Isni1gknu1noLMWOY9W9A0QDTo0TVNxD88E81xcLa0KTMSnaCrb17ZjR2IzjP0XRFIhDKZt3SPd2VPXeRx1vCmn0G4uN/RqzgA%2BC5NuU8YM%2BbN/lBSFcPURb0XwqjnblpV5vY2ldVjhODDMFQX6G%2BkVJKHSRwFZytJu57n1uHSvyG0uJ00swHJ0Hy%2BOXXbkY/fJRFEyTYw7vDFPuTTdNQ2TLkhycscj/T3Ti7j5Ks1lnMKNz6t8xD2sQEL1XWdatri81tpMwbqCkeRxBMgtSelinK3smp%2BbaUqP14FQMwQH7DmYTxtDmZZGEX4ZzvMQVA5cATAk4lCVgrAdLH1qvvO0P0l7sw/uKL%2Bzh5o22MkMCqaNd4IPqseXw8ppQJmHHsAiBgfzAGyoicGZUtQ8TbOKea%2BNjoxRmLrAhzoCLsMRpKbBV17a/3/gQKyd1xyjieFwyu3tiBFmIi9duJkIBvxmIQGYyw8ANAUD2AiYdN50F%2BnZf2pk/4wMshaCkUiZFWg%2BvIxRFdw64hpPlVARV7TwN4TsA2CgACOBgcjFVYHQWhSxzahNEKdFmmVUFLBmJKJY1jJaULSZ9X6zA7BbnTiqA2J0eK0SeJ9NR9AaqbCVN43G6BSxG3QE8JCASglqn8vPH6w03AEAgCpSQGF%2B48W4PPW0wxeisBAMMQKwxSCmGGFsKZqBxk6DkHINE/RBiiRuJwKZBBxlzLnqQAqIBApbCEOM7gUyZlzNIAs4YUyRonJ2bMkZpA4CwCQMsdIREyAUG3otAACiIZQDAEBgJmVs0gaA3R4A9MEAF9hWDAtBbsqZkLfDQtqMATgWxJAQtQFC9gxAADy1JEW0kuVMj5yA4rjIpUsdIWR8AzKmTQegTA2AcB4PwQQwhRAoGWTIIQeBnAjUgL0M%2BpQRrDD5ISsY/J0SYQkDIOQnAPB3LWUMQQ6I7BwqBSCsl4zwW0jIr4A1zyxkTIuci654ylgAA5EJ8kQtwGYwBkCoixaWSQMxsB0uQF8j%2BuBCAkG1GMTgzIllKpkNs5F%2BzDnHNOcMc50yrU3LuSAB5MbRnjMkJap51rbmkEeXs3ospiCBA0NwIAA%3D%3D%3D"
>}}

Not bad for a minor improvement! However, notice that at the moment `auto`
parameters are unconstrained, meaning that you could realistically call
`bind<2>()` and this will fail spectacularly with some horrible template error.
However, we can easily fix this by just constraining the template's inputs by
using [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae):

```cpp
  template <auto Function,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(Function),Args...>>>
  auto bind() -> void
  {
    ...
  }
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEJIDspAA6oFhdbT1DE0FvXzU6Kxt7IycXd0VlTFV/BgJmYgJA41NOBJVw2lT0gki7R2dXDwU0jKzg3Ori0ujYyoBKRVQDYmQOAHIpAGZrZEMsAGpxQZ1MAA9ezwKp7HEABgBBIZGxzEnplgUlDOW1zclh2lGDCamdKgNLgrYTjdPR5kPx6WZ0ABFE4DMAiYHRsVjjEDjTwGBysPDIcbVdAgEBzBYFSZuWQbaGw%2BEgU7jInjZgGIjjADuCCBEDa4zQtGq41oqDRmEW4wAtMt6XQmchqcQAFTjVAAN2cxDwNyxhOJ8uImAI3Vok0kMh%2B4yw7EBwPpYKkkim2PW8vEbl%2Bp3NlsGJtOwKMnmEetuBAAnp4tMwjLsGHhgLQgd1MC91u9Pv8dUCQ7arRsHU7o3sdO7PYGfeMAEqkcapr0%2BgB0RfWxGAClD4YU40jmF1MZ0mYgJbLRYLbVD5pNuLhyAJGyJAHoB%2BMdIro1WROMHg4utotQDo3Ka3XaXtfvOqKTWARjXHTeMh%2BMAOq7CkiAi5hC7ZdJ8lOXmeN3MWGYHO%2BS67QhVuHABAECmYP6f5LguwIQAyTI3sCUgAGyigQV7EHSUzrlgm4GNuu79iSZKoKKnrEECJAoeBfIXlBIaSHBqAIc4dLcoM2DVqBlFwShG5bjusavNhCbOrstykuSABiDzJHQpByvK0m5h6%2BYCYM65IiiXovgA%2BngVBqVxOjKSAeAKBptBiqgLDqcQalirc2bzqMeYQKJjz%2BB0zYKK2JyMaGRJCXhDjWOgq4MUxJnSnKnZSeMRhGY0H5rsyGGsIsxBYfuRJRdUMJxY0ajIGpBw6RlDhqfcTl0MsEDiAArLIlW/KRjIXiF6BCjmrmtukZb0TymbhdhCpKiqiIEMi%2BnGagADWmAOWJBQ5h1blFu23GpZivxLSaRLWnuRJ8Umrpyemuw6MIhw5o2x0fAoKJCgAspgMTOI54m0G0Talgtba8g1Xk4Xe/n1UyF2HCKyBdYx4xNZiG3ElF1gxb0cUSLaB7DtUJC7LR9InVW3jWMCyV9ZFamFVlaQ5XlHwFcqRUlc95VVTVdUQY1qDSiKnite97XvWDTGZlDEVEszv2oCDrBVux2XwhT1S3MLQMKEK5WeOt23SYqyrEKq4Hi0Ft33bET0FK982tqrhPWub%2B5bTx%2B67S60x5odI7Y2dEAK9dd0PcQRvOW9LaLT9PnjH52ju9jIO8xDrPoALhOww1IgI%2BxSPSCjQ3o5euyVlCrP0M4EXpdTpNAtL%2BW3IVxUzf49PVVVTNkdHbNQpzAcFvNUf871K2DsODDMFQr4h0kpJKJSuzjSyFJZ7nePOJSHwh7OsfkswzJ0Jywtw2kH6CyLYsS4pX3VDLOkK0rjEQFLuXl/sjfn8rbRW3vGuDTrCh617hvV3QJtc4Hy0zQWitptC0atg6oAIkRYgtJ/YfQ7sfC8QUsxhVlITTS4wIBF0yihditAEpJWQmgnul5iCoGnt8P4LFQSsFYOFMBgDiQ2xWq/LWmChTYIcK9BO8Mh6mwAdDZhpxPBSjFNGPsmxsIGHfMAIaMIq6lVVOxRsQpXrCyai1cYbUBFq3UTHEUPCd7JyPvg2hSVjQowHLcLRs987EFzHhTG28k67AgBgwg4w5gGQIAods2FK60wxJwuKpjEoEGSsjXuVjpg2NxnYhxM9An%2BASdYEyk0rQMLtBsYOCgACOBh0hTVYHQWRswo7FNEFDcYrDVSzHGCKWYFihG2zxpFZg1hVzd3lMHWO7EKK3Dxm4%2BgS0ViykyRFdABZQ7oFuLBPJBTFTlWfoTS6zgCAQACpIZCil2LcGfhaPoHRWAgD6JVPopBTB9FWOc1AJydByDkIiLoPQBLnE4OcggJzrlP1IONEAlVVhCBOdwc5lzrmkFuX0c5V1AWfKuYc0gcBYBIDmEkXCZAKAQHmgABREMoBgCByGXPeaQNAjo8DOn8LimwrACVEq%2BecslngKUVGAJwVYkhSWoHJewYgAB5MkdKKRgvOai5ArkTmitmEkVI%2BBLnnJoPQJgbAOA8H4IIYQogUAPJkEIPADgrqQA6JAgoV0%2Bicj5YMLkSIUISBkHITgbhoXPN6IIJE1hqX4sJcKk5JKKSEU8L6hFxzTmgoZRCk5swAAcMFOQwW4OMYAyAETsoLJIcY2BpXIHRZg3AhASBqkGJwOk9z7UyA%2BQyn5fyAVAr6CCi54bIXQpALCytRyTmSDDfCiNULSBwu%2BR0CUxBfAaG4EAA%3D%3D"
>}}

This will now ensure that calling `bind<2>()` will error that there is no
valid overload available, rather than failing with some complicated template
error.

### Reworking Const Member Functions

Now we need to support member functions using `auto`. This will allow us to
support both covariance _and_ remove the redundant type specification.

Unlike before where we had to order the `template` arguments with the `Class`
first, `auto` arguments now allow this order to be reversed to be:

```cpp
template <typename MemberFunction, typename Class>
auto bind(const Class* cls)
```

This now provides us with two things:

1. We can get the type-deduction of `Class` for free, and
2. We can use the desired calling notation of `d.bind<&Foo::do_something>()`

As with before, we will use SFINAE to ensure that this only works correctly with
invocable functions, and `std::invoke` to clean up the code.

```cpp
  template <auto MemberFunction, typename Class,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(MemberFunction),const Class*, Args...>>>
  auto bind(const Class* cls) -> void
  {
    // addressof used to ensure we get the proper pointer
    m_instance = cls;

    m_stub = static_cast<stub_function>([](const void* p, Args...args) -> R{
      // Cast back to the correct type
      const auto* c = static_cast<const Class*>(p);
      return std::invoke(MemberFunction, c, args...);
    });
  }
```

Lets check to make sure this works correctly:

```cpp
auto str = std::string{"Hello"};
auto d = Delegate<long()>{};
d.bind<&std::string::size>(&str);

assert(d() == str.size());
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAA4ArKQAOqBYXW0eoYmgj5%2BanRWNvZGTi6uAAyKypiqAQwEzMQEQcamnMkqEbQZWQRRdo7Obp4Kmdm5IQV1ZRUxcW4JAJSKqAbEyBwA5FIAzNbIhlgA1OKjOpgAHoNexXPY4gkAgmMTU5iz8ywKStnrmzuS47STBjNzOlQGN8Vs59sXk8wn09LM6AARFLAZgETA6NisaYgaZeAwOVh4ZDTOroEAgJYrYqzADssm2cIRSJAF2mZOmzAMRGmAHcEKCIF1pmhaHVprRUJjMKtpgBadbMuhs5D04gAKmmqAAbs5iHh7njSeTlcRMAR%2BrRZpIZP9plh2CCwczIVJJHN8VtleIcQCLtbbaMLRcwUYvMIjQ8CABPLxaZhGA4MPDAWig/qYd5bL4/IEG0ERx127Yut3xw46b2%2B0MB6YAJVI00zfoDADoy1tiMAFJHowpprHMIaEzpcxAK1WyyWupHrRbCYjkCTtmSAPQj6Y6VXxusiabPBx9bR64HxpUNpuMw4A5dUSmsAjmpOW6Zj6YAdQONJEBELCAO67T1Kcgq8XuYCMwBb8NwOhDriOABACBpTBgyAtcVzBCAWTZB8wSkAA2SUCDvYgmTmbcsF3Ax90PYcKSpVBJV9YhQRIDDoKFG84IjSQkNQFDnCZflRmwetINopCMJ3PcD0TD58JTd0DgeSlqQAMWeNI6FIJVlXkwsfWLETRm3VF0T9D8AH08CoLS%2BJ0dSQDwBQdNoKVUBYbTiC0qUHnzZdJiLCBJJeAIenbBRO3OVjIzJMSiIcax0E3Fi2Is%2BUlV7OTpiMMyWh/Ld2Rw1hVmIPDjzJOK6nhJKWjUZAtOOAycocLSnjcuh1ggcR3FkdwAUo1kbwi9AxQLTzOyyKtmIFXNovwlU1Q1FECDRYzzNQABrTAXKk4oC26ryy27fjMtxAFVotMl7SPMkhLTUTCOmABZTBYmcVzpNoAsi2zA4dGEE5ZMGhTyTu/0VLUsaNNDbTdP0h4jJMsyLKs9gtJsuz5gcrAnKUiAzou4gruKHoYJvR7vgUdrpk6ssfI2Xz8IC6Ygu0Jq2Sxk4JUmBRetY6ZWqixVXtPf50FVE5UCoOclHQQsiK0BRw1pA5gDVW8Di8YhUBI2FUGsMF0teuLrASwYkrpjKYuy9UHDyzICqK74Sv18r5oCaravqxqMaZxW2thDrK2WksloZtj%2BtZ9bR3HCE2QcZhkCmwWpcFYhVVURTfRisl7YC2nDdBJETbqB57epnHqq8La4%2BmVV1WITVgcmmbEfOuJUYCAtkEW13Ozz177Sb49doE48Do9eYPpzLOC1bLP0TFJGq8tugujbBuVr8gin2CiAs9pz2HflXFtvJNXmpETXuIkR0T3HOoSAORjmSeusfCV5xda00rk%2BN4qgfNirruturartqjV6drwXY7MsHs%2BR9XXvnU8DBmBUE/GTVIlIlBi2mFNDkNJw5X3oM4Wk3wyaLgFtSZg7I6C8nturTIP586J3PnWPeVE04GSXtVfKqcn5HG/vQ1iEBc6t3zoXEa0FWAKDCiPSul1x60EnktRuGUrQ2lbjtG0e055ETls4MixBGRTwAe7V2TJ7ZhTzCzDeZJdLTAgHrXKGFuK0BSmldCPsFIoVligv4gIOIQlYKwaK8i1rSNtK9HhxcTFijMQ4SeW8NbQIkTPbxG0jwyzwFKeMQ4dj4QMN%2BYAo14QW0qpqbirYxST3tq1XG%2BMuw63woUx2EowmkN3qpZK7i0rmkPiOB4eMFbX2IGHM%2BJCd4HAgMYwg0wlgmQIPTJUpUsnXVinffWSUrENIIOlA%2BfsWnzDaWg5WXS7zTFftiak1gLIzTtF4p0Hwrh7DuCpQyizrDAB7NsJWsVmDWE3ANdapM6idO4kZT5tzoragABLKFYKgU0u1on%2BWOgLbiNEHggtEIyc4ioTkxXQCWcm6AHiIR%2BTc0Q6I/AAC8EzYBqnRT5XDXrY2cAQCAIV0KqW%2BYskshLZpdC4TaIYPRWAgCGO4IYpBTBDASPy1APKdByDkCiPoAwRJXE4PyggPLhVstIFNEA7gkjcqGNwflgrhWkFFUMflCgQBJEVUKzlpA4CwCQEsVIhEyAUAgEtAACiIZQDAECoBpIK%2BVpA0CujwO6AIbqbCsE9d6vV/KA1eCDdUYAnAEiSH9agQN7BiAAHkqQRp9Uq/ldrkCeR5fmxYqQMj4EFfymg9AmBsA4DwfgghhB4okDIOQQg8AOBNZAHoctigmqGLyDNow%2BSogwq22QMhOA4mNdKwYghUTWFDR6r1uahh%2BppKRLwPL5Vcp5XygVeaDU8sWK4BCvIELcGmMAZAyJE0lkkNMbApbkAOpMbgQgJAtSjE4EycVbaZAKrzSqtVGqhA8p1Yei1x6jWKFNaQc1yq91DEkLqo9hqgMWpVTKYgfgNDcCAA%3D"
>}}

Excellent -- we have it working, and with the desired syntax!

Lets do the same for non-`const` member functions.

### Reworking Member Functions

The exact same change as we did for `const` member functions can be done for
the non-`const` member function:

```cpp
  template <auto MemberFunction, typename Class,
            typename = std::enable_if_t<std::is_invocable_r_v<R, decltype(MemberFunction),Class*, Args...>>>
  auto bind(Class* cls) -> void
  {
    m_instance = cls;
    m_stub = static_cast<stub_function>([](const void* p, Args...args) -> R{
      auto* c = const_cast<Class*>(static_cast<const Class*>(p));
      return std::invoke(MemberFunction, c, args...);
    });
  }
```

Lets again do a quick check to verify this works:

```cpp
auto str = std::string{"Hello"};
auto d = Delegate<void(int)>{};
d.bind<&std::string::push_back>(&str);

d('!');

assert(str == "Hello!");
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAA4A7KQAOqBYXW0eoYmgj5%2BanRWNvZGTi6uAAyKypiqAQwEzMQEQcamnMkqEbQZWQRRdo7Obp4Kmdm5IQV1ZRUxcW4JAJSKqAbEyBwA5FIAzNbIhlgA1OKjOpgAHoNexXPY4gkAgmMTU5iz8ywKStnrmzuS47STBjNzOlQGN8Vs59sXk8wn09LM6AARFLAZgETA6NisaYgaZeAwOVh4ZDTOroEAgJYrYqzdyybZwhFIkAXaak6bMAxEaYAdwQoIgXWmaFodWmtFQmMwq2mAFp1ky6KzkHTiAAqaaoABuzmIeHuuJJZKVxEwBH6tFmkhk/2mWHYILBTMhUkkczxWyV4ncAIuVpto3NFzBRi8wkNDwIAE8vFpmEYDgw8MBaKD%2Bph3lsvj8gfrQeGHbbts7XXHDjovT6Q/7pgAlUjTDO%2B/0AOlLW2IwAUEajCmmMcwBvjOhzEHLldLxa6Eat5oJiOQxO2pIA9MPpjoVXHayJps8HH1tLrgXHFfXGwzDgCl1QKawCGbExbpqPpgB1A7UkQEAsIA5r1NUpwCrye5gIzD5vw3A6EWuI4AIAQ1KYEGgGrsuYIQMyrL3mCUgAGwSgQt7EIycxblgO4GHuB5DuSlKoBKPrEKCJDoVBgrXrB4aSIhqDIc4jJ8qM2B1hBNGIeh267vuCYfHhyZugcDwUlSABizxpHQpCKkqckFt6RbCaMW6ouivrvgA%2BngVCabxOhqSAeAKNptCSqgLBacQmmSg8eZLpMhYQBJLwBD0bYKB25wsRGpKiYRDjWOgG7Max5lyoqPaydMRimS036bmy2GsKsxC4UepKxXU8KJS0ajIJpxz6dlDiaU8rl0OsEDiAArLINUAhRLLXuF6CivmHkdlklZMfyOZRXhyqquqKIEGiRlmagADWmDOZJxT5t1nmll2fEZTiAKreapJ2oepKCamIkEdMACymCxM4LlSbQ%2BaFlmBw6MIJwyYN8lkndfrKapY3qSGWk6XpDyGcZpnmZZ7CadZtnzPZWCOYpEBnRdxBXcUPTQdej3fAo7XTJ1pbeRsPl4f50yBdoTWsljJzipMCi9Sx0ytZFCqvSe/zoCqJyoFQs5KOgBaEVoChhjSBzAKqN4HF4xCoMRsKoNYYJpa9sXWPFgyJXT6XRVlaoOLlmT5YV3zFfrZXzQEVW1fVjUY0zittbCHUVstxZLQzrH9az60jmOEKsg4zDIFNgtSwKxAqqoCk%2BtFpL2/5tOG6CSIm3UDz29TONVV4W1x9MKpqsQGrA5NM2I%2BdcSowE%2BbIItrsdnnr12k3R67fxR4He68yk0jVeW9JMdKeOT0KC9vtvR92ZcYZGkQwDZvjSD1hg2%2BENQ3Z%2BZw3uCN95dA%2B0D0We4/jxaE75%2BGPkFEDH0yrD07y/LM3hA3rWrzUiJrXHa2tSp6zlM8japyKkDc25VrrWzqrVO2lEHZynFF4F27ZSwe0fozb2205KJyZFrSiad9LHyqnlYBpsM6wMISxCAudW7yULiNUu5ly57xRgfWu9dkGdnSpaa0NCNp7UvoROWzhSLEAZK2BuKDXaMntqFXMLNMGkh0tMCA/8DboS4rQZKqU0I%2B3kshWW1Jfj/GohCVgrAorWi4WSdu606HF2UaKVRXQVFxUyN%2BdhbteE2IuDLPAko4yDh2HhAwX5gCjXhBbCqGouItlFM4%2B2rUT4SM4WtRUCTHbinfhrL6SUzGpTNMeUcDw8YKyVs4MODFpjqzcZrCASjCDTCWMZAg9NFQlUiddGKmkSqJU0XkggaUHRklHMOYpFofBlOIBU280xwHYipCvaa4YPiWNSR8K4ew7jKQMgM6wwBuzbCVjFZg1gNyvyVKTOoUzAHjSuXsqKWoAASyhWCoBNLtX%2BZJSYCy4tRB4rU6n0FWhsBUqyFG6mLOTdADwEKGTuaIdEcIFAIE0kHEO1taJXNbtFYKYAhhgAJUMbFr1sbOAIBAK5m4uImmeWY1ABKTTYutESoQIAhg1SGKQUwQwEictQGynQcg5Aoj6AMYSVxOCcoIGy3lXQehTRADVJIrA2XcE5dy3lpB%2BVDE5QoEASRpU8pZXAWASAlipAImQCgEAloAAURDKAYAgVA1JuWStIGgF0eA3QBHtTYVgTqXUas5Z6rw3rqjAE4AkSQHrUBevYMQAA8pSQNrqZWcvNcgDybKM2LFSBkfA3LOU0HoEwNgHAeD8EEMIBFEgZByCEHgBwerIA9DlsUPVQweSJtGLyVE6E62yBkJwdwurRWDEEKiawfrHXOrTUMd11ISJeDZZKnoKr2XqvTVqtlixXDwR5PBbg0xgDIGRFG4skhpjYDzcgS1yjcCEBIJqUYnBGSCvrTIKV6a5WkAVUq1lQw1Vcu3dq3V%2BrSCGtleutlkgt1Gp3TqyDP6ejSmIH4DQ3AgA"
>}}

And we have it working!

## Closing Remarks

By making use of `c++17` and `auto` parameters, we were able to enhance the
functionality of our `Delegate` class to now support covariant functions _and_
improve the user-experience by removing any redundant types from having to be
specified -- effectively making this utility even more useful!

And yet, there still is more we can improve on. In the
[next post](/posts/2021-02-26/creating-a-fast-and-efficient-delegate-type-3)
I will cover optimizing this utility so that it has exactly **0-overhead**
over using raw pointers.
