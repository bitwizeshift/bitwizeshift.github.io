---
title: "C++wtf: Final Poetry"
date: 2021-02-28T01:23:18-05:00
draft: false
tags: [esoteric, language-lawyer]
categories: []
# series: [cppwtf]
---

{{<info>}}
**Disclaimer:** The C++WTF segment shines a light on the dark and
esoteric corners of the C++ language for fun and profit
{{</info>}}

The C++ standards committee is very strongly against producing any kind of
breaking changes when considering new language features -- especially keywords.
Over the course of standardization, this has lead to terms like `auto` being
repurposed, or awkward keywords like `co_await` and `co_yield` being introduced
in C++20 to avoid breaking a handful of projects in the world that might
possibly be using `await` or `yield` in their vocabulary.

A notable byproduct that comes from this is that some things we take for granted
in every day modern C++ may not actually be **keywords** in the sense that you
would expect. For example, both `final` and `override` are not actually keywords
at all!

These are **context-sensitive specifiers**, rather than official keywords. This
means that their functionality only applies when found in a very specific place,
but otherwise allows for their names to be used in any other context -- such as
variable names, function names, and even type names!

This allows for the following program to be completely syntactically
valid, against all logical reason:

```cpp
struct final final {
  final();
  final(final&&) : final{}{}
} final;
```

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAFZSAB1QLC62nsMTQS8fNTorG3sjJxd3JRUw2gYCZmICAONTTkVlTFU/ZNSCCLtHZ0EFFLSMoOzKopKomK4ASkVUA2JkDgBySuIDVQBqKms2EbHWIfEAdlkABgBBIYnaNggW8QBmBeXV9dG11ikANlOWoZB947nZgBFZ2RmHpfvr7d2ettYQHtce0imHrzAGoX46ORyIYKDpdTDTSRbTgAgi/EEtNoAazc8yEv24AKBINIYJ6AIUIFxqOBX1IcFgMEQKFQRg8eHYZAoEDQrPZ5WAnHmklIo1YBGcFIgDjRAIcY2IAE9fsjSDyjFoCAB5WisJU00hYIwiYDsGUGvDEPJqABumAp%2BswAA88gZxcqAdZxT99aw8A5iKkFXosGaCMQ8EZ3W0aPQmGwODx%2BIJhKIUJCZEI/RTIG1UB5Evbybl8hoIOYallSOZGmUXNkQr46BXgt5G7Qa9FynVi4lCtV9JkKj2ClVitZSp264pR826qOO81OG0YZ1uq08X9CWbSY6ABwnAC0J24Q2AyGQQ0FADpJEMILhCCQEUjSEM9LyOc%2Bl2/09IUTKMVIbFXFxb0CVISMQMBbdfnJSlSGpdE6UZCAkA6AgPFdchKB5NkOTMfAiCnGNGBYU1EwAdwDDwow3f5oP1UkpCRIYKMIBAhj3Q9j1Pc9L3mG9/xpQCEEwZgsBcDYN3AyDcSJUFYMUeDEK%2BLEcQ3LYt0YxSVMA71JC04lSSE9E2ltYgfA0bggA%3D%3D"
>}}

This creates a `final` type named `final` and also creates an object of said
type which is _also_ named `final`. This is C++'s equivalent of the
[buffalo buffalo sentence](https://en.wikipedia.org/wiki/Buffalo_buffalo_Buffalo_buffalo_buffalo_buffalo_Buffalo_buffalo)
or the [shi shi shi poem](https://en.wikipedia.org/wiki/Lion-Eating_Poet_in_the_Stone_Den),
and I think that's _beautiful_. Of course, save this in the back of your mind as
an esoteric fact about C++ -- and don't spring this in any professional code, or
you might have some very upset maintainers!
