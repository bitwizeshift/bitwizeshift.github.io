---
title: "Getting an Unmangled Type Name at Compile Time"
subtitle: "A really useful and obscure technique"
date: 2021-03-09T19:21:59-05:00
draft: false
author: "Matthew Rodusek"
tags: [c++17, cross-platform, compile-time, constexpr, gcc, clang, msvc]
categories: [tutorial]
aliases: [/posts/2020-03-09/getting-an-unmangled-type-name-at-compile-time]
---

Getting the name of a type in C++ is a hassle. For something that should be
trivially known by the compiler at compile-time, the closest thing we have to
getting the type in a **cross-platform** way is to use
[`std::type_info::name`](https://en.cppreference.com/w/cpp/types/type_info/name) which
is neither at compile-time, nor is it guaranteed to be human-readable.

In fact, both GCC and Clang actually return the compiler's **mangled** name
rather than the human-readable name we are used to. Let's try to make something
better using the modern utilities from {{<tag "c++17">}} and a little creative problem
solving!

<!--more-->

{{<table-of-contents>}}

## Reflecting on the Type Name

The C++ programming language lacks any real form of reflection, even all the
way up to c++20. Some _reflection-like_ functionalities can be done by
leveraging strange or obscure techniques through templates; but overall these
effects tend to be quite limited.

So how can we get the type name without reflection?

Realistically, we shouldn't actually _need_ reflection for this purpose; all
we need is something the compiler can give us that _contains_ the type name
in a string. As long as the string is known at compile time, we can then use
modern C++ utilities to _parse_ the string to deliver it to us also at
compile-time.

## Finding a String at Compile Time

We know that `typeid`/`std::type_info::name` is out -- since this doesn't
operate at compile-time or yield us a reasonable string.
There aren't any specific language constructs explicitly giving us the type
name outside of `typeid`, so we will have to consider other sources of
information.

Checking the C++ standard, the only other sources of strings that may exist
at compile-time come from the preprocessor and other builtins; so lets start
there.

### Macros

One somewhat obvious approach to getting a type name as a string is to leverage
the macro preprocessor to stringize the input. For example:

```cpp
#define TYPE_NAME(x) #x
```

This will work for cases where the type is always statically expressed, such as:

```cpp
std::cout << TYPE_NAME(std::string) << std::endl;
```

which will print

```
std::string
```

But it falls short on indirect contexts where you only have the `T` type, or
you are deducing the type from an expression! For example:

```cpp
template <typename T>
void print(int x)
{
  std::cout << TYPE_NAME(T) << std::endl;
  std::cout << TYPE_NAME(decltype(x)) << std::endl;
}
```

will print

```
T
decltype(x)
```

Which is not correct. So the preprocessor is unfortunately not an option here.
What else could we use?

Perhaps we could extract something that _contains_ the type name in the string
-- like getting the name of a function?

### Function Name

If we had a function that contains the type we want to stringize; perhaps
something like:

```cpp
template <typename T>
void some_function();
```

Where a call of `some_function<foo>()` would produce a function name of
`"some_function<foo>()"`, then we could simply _parse_ it to get the type name
out. Does C++ have such a utility?

Standard C++ offers us a hidden variable called
[`__func__`](https://en.cppreference.com/w/c/language/function_definition#func), which
behaves as a constant `char` array defined in each function scope. This
satisfies the  requirement that it be at compile-time; however its notably
very limited. `__func__` is only defined to be the _name_ of the function, but
it does not carry any other details about the function -- such as overload
information or template information, since `__func__` was actually inherited
from C99 which has neither.

However, this doesn't mean its the end. If you check in your trusty compiler
manual(s), you will find that all major compilers offer a `__func__` equivalent
for C++ that also contains overload and template information!

## Exploiting the Names of Functions

GCC and Clang both offer a `__func__`-like variable for C++ which contains
more detailed information as an extension called
[`__PRETTY_FUNCTION__`](https://gcc.gnu.org/onlinedocs/gcc/Function-Names.html).
MSVC also offers a similar/equivalent one called
[`__FUNCSIG__`](https://docs.microsoft.com/en-us/cpp/preprocessor/predefined-macros?view=msvc-160).

These act as **compiler extensions** and as such are not -- strictly speaking --
portable; however this does not mean that we can't wrap this into a useful
way. Lets try to make a proof-of-concept using GCC's `__PRETTY_FUNCTION__`.

### Format

The first thing we need to know is what GCC's format is when printing a
`__PRETTY_FUNCTION__`. Lets write a simple test:

```cpp
template <typename T>
auto test() -> std::string_view
{
  return __PRETTY_FUNCTION__;
}

...

std::cout << test<std::string>();
```

Yields

```
std::string_view test() [with T = std::__cxx11::basic_string<char>; std::string_view = std::basic_string_view<char>]
```

Which means that, at least for GCC, we need to find where `with T = ` begins
and `;` ends to get the type name in between!

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(fontScale:14,j:1,lang:c%2B%2B,selection:(endColumn:1,endLineNumber:15,positionColumn:1,positionLineNumber:15,selectionStartColumn:1,selectionStartLineNumber:15,startColumn:1,startLineNumber:15),source:'%23include+%3Ciostream%3E%0A%23include+%3Cstring%3E%0A%23include+%3Cstring_view%3E%0A%0Atemplate+%3Ctypename+T%3E%0Aauto+test()+-%3E+std::string_view%0A%7B%0A++return+__PRETTY_FUNCTION__%3B%0A%7D%0A%0Aauto+main()+-%3E+int+%7B%0A++++std::cout+%3C%3C+test%3Cstd::string%3E()%3B%0A%7D%0A%0A'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:51.962323390894824,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:g81,filters:(b:'0',binary:'1',commentOnly:'0',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1'),fontScale:14,j:1,lang:c%2B%2B,libs:!(),options:'-std%3Dc%2B%2B17+-O3',selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'x86-64+gcc+8.1+(Editor+%231,+Compiler+%231)+C%2B%2B',t:'0'),(h:executor,i:(argsPanelShown:'1',compilationPanelShown:'0',compiler:gsnapshot,compilerOutShown:'0',execArgs:'',execStdin:'',fontScale:14,j:1,lang:c%2B%2B,libs:!(),options:'-std%3Dc%2B%2B17+-O3',source:1,stdinPanelShown:'1',wrap:'1'),l:'5',n:'0',o:'x86-64+gcc+(trunk)+Executor+(Editor+%231)+C%2B%2B',t:'0')),header:(),k:48.03767660910518,l:'4',m:100,n:'0',o:'',s:1,t:'0')),l:'2',m:100,n:'0',o:'',t:'0')),version:4"
>}}

### Parsing: A first attempt

So lets try to write a simple parser that works at compile-time.
For this we can use `<string_view>` for the heavy-lifting.

```cpp
template <typename T>
constexpr auto type_name() -> std::string_view
{
  constexpr auto prefix   = std::string_view{"[with T = "};
  constexpr auto suffix   = std::string_view{";"};
  constexpr auto function = std::string_view{__PRETTY_FUNCTION__};

  constexpr auto start = function.find(prefix) + prefix.size();
  constexpr auto end = function.rfind(suffix);

  static_assert(start < end);

  constexpr auto result = function.substr(start, (end - start));

  return result;
}
```

The algorithm is simple:

1. Find where the prefix starts, and get the index at the end of it
2. Find where the suffix ends, and get the index at the beginning of it
3. Create a substring between those two indices

Lets test if it works. A simple program of:

```cpp
std::cout << type_name<std::string>() << std::endl;
```

now prints:

```
std::__cxx11::basic_string<char>
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tENwAcpLegAyeWpgByxgEaZiIAJykADqgWE6rR6hiZmln4BanR2Ds5Gbh7eSirRtAwEzMQEIcamForKmKpBGVkEsU6u7l6Kmdm5YQUK9RX2VQk1ngCUiqgGxMgcAORSAMz2yIZYANTiYzrqLcSYzEbz2OIADACC45PTmHMLy/bAG9t7khO0Uwaz8zqnogD6AG54mADuF7uXBJgjD5hADjjoCABPHxaNZHAAqvx2aFoLUwAA8fMQZswDEQZpDoS9aLCIN0ZgBaDYzFroEAgZ7Ad6fH5/ADssl2MxmyNRGKxOLxmMwVDwaK5xwAItSCLT6QRiGcmd9xOypJJxABWaRfQgIGZwyVzSTq1kS%2BYcnZcnkAvnY3GoakGKgisVc%2BZSml0hlKn6q43mtUqs1jC1Wui8zF2vFUAy3NKGz1yhWvD7K9kvF4ABQAStg4XCAJovABiAFVHDo4QBJADyjgzQYDf053PDNsjAodLXKhpjcaCADoRdoIEKXWS5DMx6KBwEAF6YUlNy2tlHt/n2mbWXuxkp0AfEYfoCAKJ3j5eXLndtTIF7MBRKbIn1pgrfaboXlvW9EdzcrU%2BsAQO79vup4uMsz7lKQMwQNu5LSuU3QfiGlyXjMKwEAMtDoZgAEEBepqobsnYzEYzD2KSFJUvYQEqqG4qJmguJgo8%2BJQpgRKwo8iYMhslGPKxibWKwBFmrswy9KwIDDBqwykKYwxbHJqDSTociTgo/SDEc4ycHJBDSUpSGkAA1iAGqcAOngAGzXNcYyeFs5ieBYki8FJwzcHJClKaQKnDHJCggFspAGYpEmkHAsBIGgQJ4OwZAUBAsU%2BPFNTAOYnCkCKgHuEFEAuIZckuPYWQQtJemkLFRhaAQNa0Kw5XhaQWBkaI7BFS1eArCUby4Z16LFLiIyVTRyidaweAuMQZV6FgnXyngRgVRFND0EwbAcDw/CCMIogoOpMhCFNQWQL0qA%2BGkQXDOSNLuhIMhyJwrIUjWYyBUUe6mLB2iNKYWXWJU8SJIIkSBHQf2g/44O0ED1QeFlKTFGkZQNPoeSCEjX2o20cTw5jrSQ4jrRw50CO9JpAxDFwknSbJ8mdf5aLmNZ5LWdwMzAMgyAzOYA6cDBuCECQRpjFlMx6HFCWi5wZJqY9Mj6UVvQQEgg3IPaiWUFkwAKJmIjKAwCCoF8CmVSl8XMGk%2BsOKwRsmz5ckWwlIC68SPgKMbBBVagUvuDWuL26bA1osUOzELr0lyerGT4ApclrYwLAddtAhZXtYiHfIk3gfA52XUE123TK91Z89r3vX0VNbXUcc24bxvB8MlVfDNPgrbTMneYz0nM6z7Oc9zMHyrGJlktgoca0QWIQEL08y3LWdK%2BFxkIKsWAeKSpnhAOWxjKy1kH9ZWyeJwWwWZYHleaQy1nyFjt%2BVHijBaFyud5Icm31s989wFr8r70PqxAAgaG4EAA%3D"
>}}

Great!
Before we celebrate, we should check to make sure that the compiler isn't
embedding the entire function name -- otherwise we might bloat the executable
with unused strings. Otherwise, this would be quite the trade-off, and not
zero-overhead.

Checking the generated assembly, we can see that the string _does exist_ in its
complete form, even at `-O3`:

```assembly
type_name<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >()::__PRETTY_FUNCTION__:
        .string "constexpr std::string_view type_name() [with T = std::__cxx11::basic_string<char>; std::string_view = std::basic_string_view<char>]"
```

Which means that the compiler was unable to detect that the rest of the string
name was unused.

**Lets fix this.**

### Iteration: Optimize out unused string segments

In the first approach, the string is taken in its entirety -- which is likely
why its also seen in the assembly verbatim. The compiler sees its used, but
is unable to see that only *part* of it is used. So how can we make it see that?

The easiest way is to instead **build** a string, at compile-time, that *only*
contains the name of the type -- and not the rest of `__PRETTY_FUNC__`. This way
the compiler will not see any runtime uses of the function name, it will only
see runtime uses of the manually built string.

Unfortunately there is no way to build a `constexpr` string specifically in
C++17, and `basic_fixed_string` never made it into C++20; so
we will have to do this the old fashioned way: with a `char` array!

To build the array, we will need to extract each character independently at
compile-time at each specific index. This is a job for `std::index_sequence`,
and we can leverage C++17's CTAD of `std::array` along with `auto`-deduced
return types to make this even easier:

```cpp
// Creates a std::array<char> by building it from the string view
template <std::size_t...Idxs>
constexpr auto substring_as_array(std::string_view str, std::index_sequence<Idxs...>)
{
  return std::array{str[Idxs]..., '\n'};
}
```

And then we just need to update our `type_name()` function to make use of this

```cpp
template <typename T>
constexpr auto type_name() -> std::string_view
{
  ...

  static_assert(start < end);

  constexpr auto name = function.substr(start, (end - start));
  constexpr auto name_array = substring_as_array(name, std::make_index_sequence<name.size()>{});

  return std::string_view{name_array.data(), name_array.size()};
}
```

Lets test to see if it works! `type_name<std::string>()` now gives us:

```
�@�o�4P@
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEJMmkt6ADJ5amAHLGARpmJmAbKQAOqBYXVaPUMTMwtffzU6W3snI1d3SS8lFSjaBgJmYgJg41NzRWVMVUCMrIIYxxc3T0VM7NzQgoV6irsq%2BJqkgEpFVANiZA4AcikAZjtkQywAanExnXUW4kxmI3nscQAGAEFxyenMOYXlu2AN7b3JCdopg1n5nVPRAH0ANzxMAHcL3f3bw7HHRZYjMACeGxmUIA9NCZi10CAQCDwZd/ncHgsDGpWIQIWNsDNYfCCIiQHYsAAPF5KACOBi0QzRu2JOhWzAImAUM2YJLJKPxOmQCCykOcYJmzgMeFY%2BFEM0IMyoxFQRhmBAQR2ewBmH2%2Bl05Rm8wk5QIRSP8AC9MC8CAA6B0ASXQlIUvx2aFoLUwlO8xB52NQ8IMzm1L2YCnDxFBYIg5pAYb1XxJZD5SIpPppmHpjMwj2droddo23TRAHZZLsoSsCANaGnkdHURXluIAKzSAtutsAESLpBmYFGbZ0tCH4jLPfmlb2k%2BZO0Nxo5R0eBDB3i0ayOABV3Z7vb7/cxA%2Br1zbaFuIN0ZgBaSHxxOfH5/CuXKH7zmHgNEGZ%2BzBUPBKShY4ewbR99Vfcx22kL5CAQGZtxAuYoLnMYZ3fOgDz9b8gwUAwqAAoCoXmUCHwIYgzneJ8J1kKC0KkSQJyneiqxmD8fWw48fyoAxbjSJCyIo14kxol4XgABQAJWwbdtwATReAAxABVBwdG3R0AHkHDEpjp3nDCvU/TiTxacokJ4vjAjtADtAgP9COvORfxWQi7StTAr301j2K/LigysCzeJKOg7WIWz0DjfDHO8vZWLMtRkHDBQlGyONWiBGYrFLFi4p2QysKPE8LyMFcxlAyyQtodyQ2WdLygHCBApvElym6HL0LYzDjKKn8SptAUBNq8jKIjKMYwgfqB3jIxmAAaxtDNqTpBlbjzBZ%2BvcvBrS8gkaKYjqDJmGs6zAkbhOoit%2BvG8E7XQDlmCvAdroFLadtLVCZz0v5dn8mZZrsK9b0hOwCDmV9WKheM0GxIFHlPDcXn6x5BLODYgceeH4ysVhYu%2BnZhl6VgQGGNthlIUxhi2cnUBJnQ5GchR%2BkGFdrk4cmCBJ6n2tIOaQDbTg7QATg8a5rjGIWtgADiF7gpckXhieGbhycp6nSFp4ZyYUEAtlITmqcJ0g4FgJA0CNGU3HIShze8S33GAKXOFIADWE5YgdYgZwufJ5w7CyMESfZ0hzdK%2BhNNoVhA8N0gsFm0R2B92O8BWEo3i5JOfWKbERmD0HlCT3FnFBYgwT0LAk5Gowg6Nmh6CYNgOB4fhBGEUQUAZmQhDwUN4F6VBvDSHXhhvBESIkGQ5E4Mtb00sZtaKKrNG0RpTGdqxKjiBJBAiAI6FXne/D32hN%2BqdxnZSYo0jKBp9DyQRL6qm%2B2liM%2BH9aA%2BL9aU/OnP3omYGEMLgRMSZkwpknTWlIpYeBvB4bgMxgDIGQDMKWdpOAzAgLgQgJBkJjGdjMPQFt2D%2BnGJwa89NJ4yA5j7XoEAkBZ2QIGMgFAIBZGAAocSIhlAMAQKgL4lNg62xlByQInD7CsB4XwtW5MhHEJAOwi83gFC8IICHVUdtiGaWxJI/hmdKTFB2MQdhJNyYMIyHKExLs6CMBYInZuAhnZtzEJ3eQRcdaQH7oPQIw9R6knHi46es9559EAU3OocoxHcN4bo4Ywcvigm8DXEBpNVYQJJlAmBcCEFIIweRXic1rzYH0Ywog/pMH4FKbgshBCXHUMNjzTUzAsDuCenzOWdothjDLB4bpHgthC04FsAWUshAkxVqQaugy9bSI1pYnWzsDbc2SZIcmkytjTLSVrfWNDejpw9oEEA3AgA"
>}}

Un-oh, something definitely went wrong!

If we look closely at the previous code, we are actually returning a reference
to a local `constexpr` variable -- creating a dangling reference:

```cpp
  constexpr auto name_array = substring_as_array(name, std::make_index_sequence<name.size()>{});
  //        ^~~~~~~~~~~~~~~ <-- This is local to the function

  return std::string_view{name_array.data(), name_array.size()};
```

What we would *like* is for `name_array` to be `static`; though unfortunately
`constexpr` requirements in C++ prevents objects with `static` storage duration
from being defined within `constexpr` functions.

### Fixing our dangling pointer

Though we can't define a `static` storage duration object inside of a
`constexpr` function, we _can_ define a `constexpr` `static` storage duration
object _from_ a `constexpr` function -- as long as its a static class member.

```cpp
template <typename T>
struct type_name_holder {
  static inline constexpr auto value = ...;
};
```

If we rework our code from before a little bit, we can rearrange it so that
the parsing from the `type_name` function now returns the `array` of characters
at `constexpr` time to initialize the `type_name_holder<T>::value` object.

```cpp
// Note: This has been renamed to 'type_name_array', and now has the return
// type deduced to simplify getting the array's size.
template <typename T>
constexpr auto type_name_array()
{
  ...
  // Note: removing the return type changes the format to now require ']' not ';'
  constexpr auto suffix   = std::string_view{"]"};

  ...

  static_assert(start < end);

  constexpr auto name = function.substr(start, (end - start));
  // return the array now
  return substring_as_array(name, std::make_index_sequence<name.size()>{});
}

template <typename T>
struct type_name_holder {
  // Now the character array has static lifetime!
  static inline constexpr auto value = type_name_array<T>();
};

// The new 'type_name' function
template <typename T>
constexpr auto type_name() -> std::string_view
{
  constexpr auto& value = type_name_holder<T>::value;
  return std::string_view{value.data(), value.size()};
}
```

Trying this again now, we get the proper/expected output:

```
std::__cxx11::basic_string<char>
```

And checking the assembly, we now see a sequence of binary values representing
only the type name -- and not the whole `__PRETTY_FUNCTION__`

```assembly
type_name_holder<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >::value:
        .byte   115
        .byte   116
        .byte   100
        .byte   58
        .byte   58
        .byte   95
        .byte   95
        .byte   99
        .byte   120
        .byte   120
        .byte   49
        .byte   49
        .byte   58
        .byte   58
        .byte   98
        .byte   97
        .byte   115
        .byte   105
        .byte   99
        .byte   95
        .byte   115
        .byte   116
        .byte   114
        .byte   105
        .byte   110
        .byte   103
        .byte   60
        .byte   99
        .byte   104
        .byte   97
        .byte   114
        .byte   62
        .byte   10
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tENwAcpLegAyeWpgByxgEaZiXSwAdUCwuto9QxMzb19/OjsHZyM3D05LJRU1OgYCZmICIONTC0VlTFUAtIyCKKdXd09FdMzskLyFWrL7CtiqhIBKRVQDYmQOAHIpAGZ7ZEMsAGpxEZ11JuJMZiNZ7HEABgBBUfHJzBm5xftgNc2dyTHaCYNp2Z1j0QB9ADc8TAB3M%2B3d6/3DnQZYjMACeaymEIA9JCpk10CAQEDQedfjc7nMDGpWIQwSNsFNobCCPCQPYsAAPJ5KACOBi0AxR20JOiWzAImAUU2YRJJSNxOmQCAy4JcIKmLgMeFY%2BFEU0IUyoxFQRimBAQB0ewCmb0%2B53ZRi8wnZALhCL8AC9ME8CAA6O0ASXQ5IU3y2aFoTUw5K8xC5mNQsIMLk1T2YClDxGBIIgppAIZ1HyJZB5CLJXqpmFp9Mw90dzrtNrWnRRAHZZNsIUsCH1aCnEZHkWXFuIAKzSPMulsAEQLpCmYGGLZ0tAH4hLXdm5Z248ZW31hrZB3uBBBXi0KwOABVXe7Pd7fcx/arV1baBuI1GIMWfmXzhDd%2Bz936iFMfZgqHhyRDDl26/H3l8t6SJIrbSB8hAIFMm4/jMwFjhOIxTvedB7j6z4BgoBhUB%2BX4QrMv6xv%2BupASB3ZSCBM6IXeUwPl6aGHi%2BVAGNcKS1vhf4EMQJyvABY7SE8TwAAoAErYJum4AJpPAAYgAqo4OibvaADyjgCfBk6zshHqPvRR5NKUMFMSxAQ2h%2B2gQG%2BOGdLB0ivksOE2hamBXppFY0ShukHke1hGcxRR0DaxDmegMZYdZbk7O5BlqMgoYKEomQxs0AJTNYxZUT87m0U%2BDEBmeRhLiMv7GQFtBOUGizJaUfYQL5AC0RKlJ0GVIVMVY1oGwacdxYYXqCEAFZgfaxkYzAANZWmmlI0nS1w5nMQ1OXglquXifHwa1pYTllc6YAaRpFToK5rkNUGuosBiqMea5PENTwIPoWC%2Bht0XpLFcq0NiDgeTpdHeS%2BLxsHSMEnae558vc254mtU4aZlUV7Qdi4AmDZ3Q%2Bs2w5XpL5g3dG5XlM9XgoRPXPAmpZtdjAOoFIABs2rA0Vv54/dj3Su4UNrAiQOGAtbUdcQtak1x5O8WWvN0ja6BsswV59pLmDLatxaUXDM67eceVTGN9iE8TeKfQQMy3u5EKxmgmIAvcN3g4V9wiycayE/cNuxtYrCRRp2yDN0rAgIMLaDKQpiDBsweoAHOhyHIsK9P0S6XJwwcEAH4ctaQ40gC2nA2gAnHTlyXCMecbOYecWJIvD%2B4M3DB6H4ekJHgzBwoIAbKQqdh77pBwLASBoAaUruOQlCD14w8eMA5icKQH6sOyxBtxALhp8HLj2BkIIB8npCD4V9DKV92/d6QWBjaI7Br2feBLEULwctfXqFJiQy7/Y7I1432IuMCxAgnoLA18epGB3j3Gg9AmBsA4DwfgghhCiBQDHGQQg8DBngN0VAXhWJt0GPVOE%2BEJAyDkJwEsRNlIjFbgUMqmhtD1FMLPaw5QYhxEED4PwrF6FsPCKxZhlR4j5GSMUZoXDZ5JEKKxEomQ%2BHtAEQZOo%2BgciCHkS0aI/CuDdAUPHAYGihAByDiHa%2BzdyTmDpvVOm3ApjAGQMgKY5gbScCmBAXAhASCwRGLPKYegh7sBeknGy0diEyBTmvboEAkDP2QP6MgFAIAZGAAoQSIhlAMEeh8UOu9x5SjZAEJJDhWCpNQOk6%2BWTfEgASWeLwChHoED3sqCevjlKYkKcU0%2BkStjEASQHYOkS0gym6XPOgjAWBX1gQIWeCCxDIPkD/NukBMHYICLg/BxJCHTNIeQyhPQ%2Bg6LEcSeweSUlpIycHD4wIvBgL9vo%2BuRiA4mLMRYqxNinGcWYuNGy2ByQvyIL6Zx%2BAfnuM4AE6ZITu4Z3VMwZ6lBuhZwsDaDYIwSx0yRXTDYedOAbBzpYGuddSCgIxR3BuEcBlt1nl3dOVzBiSGDvijYhLbkt07qE7oD8l4BDMEAA%3D%3D"
>}}

So yay, we got it working for GCC!

### Supporting other compilers

Clang and MSVC can be handled similarly, and only require the `prefix`,
`suffix`, and `function` variables to be changed at compile-time
(e.g. through `#ifdef`s).

In the end, I came up with the following snippet to toggle the behavior:

```cpp
template <typename T>
constexpr auto type_name_array()
{
#if defined(__clang__)
  constexpr auto prefix   = std::string_view{"[T = "};
  constexpr auto suffix   = std::string_view{"]"};
  constexpr auto function = std::string_view{__PRETTY_FUNCTION__};
#elif defined(__GNUC__)
  constexpr auto prefix   = std::string_view{"with T = "};
  constexpr auto suffix   = std::string_view{"]"};
  constexpr auto function = std::string_view{__PRETTY_FUNCTION__};
#elif defined(_MSC_VER)
  constexpr auto prefix   = std::string_view{"type_name_array<"};
  constexpr auto suffix   = std::string_view{">(void)"};
  constexpr auto function = std::string_view{__FUNCSIG__};
#else
# error Unsupported compiler
#endif
  ...
}
```

These three variables are the only ones that would need to be changed to port to
a different compiler that also offers some form of `__PRETTY_FUNCTION__`-like
equivalent.

## A Working Solution

Putting it all together, our simple utility should look like:

```cpp
#include <string>
#include <string_view>
#include <array>   // std::array
#include <utility> // std::index_sequence

template <std::size_t...Idxs>
constexpr auto substring_as_array(std::string_view str, std::index_sequence<Idxs...>)
{
  return std::array{str[Idxs]..., '\n'};
}

template <typename T>
constexpr auto type_name_array()
{
#if defined(__clang__)
  constexpr auto prefix   = std::string_view{"[T = "};
  constexpr auto suffix   = std::string_view{"]"};
  constexpr auto function = std::string_view{__PRETTY_FUNCTION__};
#elif defined(__GNUC__)
  constexpr auto prefix   = std::string_view{"with T = "};
  constexpr auto suffix   = std::string_view{"]"};
  constexpr auto function = std::string_view{__PRETTY_FUNCTION__};
#elif defined(_MSC_VER)
  constexpr auto prefix   = std::string_view{"type_name_array<"};
  constexpr auto suffix   = std::string_view{">(void)"};
  constexpr auto function = std::string_view{__FUNCSIG__};
#else
# error Unsupported compiler
#endif

  constexpr auto start = function.find(prefix) + prefix.size();
  constexpr auto end = function.rfind(suffix);

  static_assert(start < end);

  constexpr auto name = function.substr(start, (end - start));
  return substring_as_array(name, std::make_index_sequence<name.size()>{});
}

template <typename T>
struct type_name_holder {
  static inline constexpr auto value = type_name_array<T>();
};

template <typename T>
constexpr auto type_name() -> std::string_view
{
  constexpr auto& value = type_name_holder<T>::value;
  return std::string_view{value.data(), value.size()};
}
```

{{<figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAGZJpAA6oFhdbT1DE0FvXzU6Kxt7IycXd0VlTFV/BgJmYgJA41NOBJVw2lT0gki7R2c3DwU0jKzg3Ori0ujYyoBKRVQDYmQOAHIpV2tkQywAanFXHXVq4kxmI0nscQAGAEFB4dHMCanZ62Al1Y3JIdoRg3HJnX3RAH0ANzxMAHcj9c3z7d2ddOJmACeSzGIIA9KCxtV0CAQH9AcdPhcrlMDGpWIQga5sGNwZCCNCQNYsAAPO5KACOBi0vQR61xOjmzAImAUY2YeIJcMxOmQCHSwIcALGDgMeFY%2BFEY0IYyoxFQRjGBAQO1uwDGT1ex2ZRk8wmZPyhMN8AC9MHcCAA6K0ASXQxIU7zWaFo1UwxM8xDZqNQkIMDlVd2YCkDxH%2BAIghpAAY1LzxZA5MKJbrJmEp1Mw11t9qtFqWbQRAHZZOsQXMCN1aAnYaH4UXZuIAKzSLMOhsAERzpDGYAGDZ0tB74gLbcmxY2w9pa21uqZO2uBABni0Cx2ABVHc7Xe7PcxvYrF2baCuQ2GIPmPkWEac8FQxlgqNZMOgIHc7iMRMBX%2Be1iDN8zt16RBjB6mAPsSIK7G2VbRs8byXpIkiNtIq6QRMCFDiOrhjr%2BdBbh6gE%2BgoBhUGBEGTFBkYwZq8GIe2UiIROWHHDhLr/vhu5AVQBjnAUqGUQQxAHI8sFDtIr4AAoAErYKuq4AJp3AAYgAqrYOirtaADytivhho5Xq4yg3neoGPs%2Br4AOK2MpOhfsxYx/m67F7iBpEguR0ECUJMaifRLyEAgYwoR59F6UxJYObhbE7nuREkXg4Hua4FH4kaXn3D5NGNiO6GMdhkWsU5MWcdxyR0HxqVRuln6ZWJdxSTJ8lKap6laTpdxhWOgxGbe95mS%2BACyDC2QAatgknfixeHFT6rkJWRyWeYJGUiTRC5LncR5GGaXLXKFeX2Y5AEcYRxFuRVBJUXBsjoViEAPKgeDoPmuWYflR3OSVPH%2BBdaXLTVq11SpakMNaFm6QdHynMoSgGWMzhyp6ykugYnjeBkT6RTqYrOAZ5g3pOU3RQReLFKhXHfXQFoPtoEBzcSbRodIwFzGBFompgZ76RFH0zfD2jk6VBQWsQNPPnFYH5uFGwRY0ajIIGChKBkEY1AQPz8893Myz%2BBXTSTW1zotFNlbQ7N%2BrMqvFF2EDmGMAC0pMZG0Uv5WWFa%2Bv61WKyegIQIbXaRkYzAANZmkmpIUlS5wZlMhvs3gppc1ivnDq7hYjh86zTnqRs6Oty7bUFjqzAYqj7htht3Ag%2BhYJ6vmy2k8tSrQ6I2HrxMneqbBUqhBebceu1TOud3pxeb2Tjns4/AXhvFyn6y8yT/eG2eDvAvx/3CZqF7vVFRUEVIABs3eGEbUEr8eNfirjw9LDCDw97HbuYOWxCVpv3mA4/Z8WugTLMDPF2H%2BVIE5J3zJDccmcdbHC7sHawa97bAmsOrBuusQSRjQKiH41wK6HhXNcT%2Boglhrz2lMKs5hWDaz0usPoHRWAgD6A2PopBTB9BWCw1AjCdByDkJCLoPQ5ynE4CwggjCOEu1ICHNw3ALSuB4NwAAnA2FYkgAAcBY1ErALA2BsQhGHcBYWwjhpAuF9BYQoEAKxSBiPYXQ0gcBYBIDdEkb0ZAKAQHSMABQ4kRDKAYDXF4bCRGkDQNjPU/hfE2FYAE1AQTxEsLCZ4HGLhvFHk8AoGuBBQnymSewYgmlUSxPiXY0gLjkBrGIN4xhLDympAlDU0gNB6BMDYBwHg/BBDCFECgXhMghB4H9PADoqBPAFEsX0e2UJyISBkHITgBYHaaVcBYgRvRBBQmsFE/xgTgksJeP8TwjCRH0MYcw1hCTTGMOJGoo%2B9sj7cDGMAZAyAxgQAEtxEOjNsDElcUQT0EBcCEBIGheRjMeFzJkKIhJkjlTMDrpQDo0j3ByIbJIBZnAj6uE0WohsijXD6L6IY0gRguArGscYzhjTLG5FsRIhxMBEAgHKW48glAvE%2BL8TE3ZxzEm5LFEySJXLimUpyeEio757gfNoCHMVeTnCFMYDy0p5TKnVPMWU35yB6nWEac0xgLB2BcF4AIDw3SxB9PkOiIZkARljP8BMqZ%2BIZmWoWUslZnRujrIaPiLZwrlUhIOcwI5fQTmEvOaKsxNy7kPIcuat50qvljB%2BX8kFgL8D/NBZwcFlroV2Mkci1wcjXAltLWW0thLiWRupVYmxMLTl9EkCw0lnByUXNKWYvNEiOgPGcL4DQ3AgA%3D"
>}}

## Closing Thoughts

Although there may not be any built-in facility to get the full type-name of an
object at compile-time; we can easily abuse other features of C++ to make this
possible in a reasonably portable way.

This is all possible thanks to the many reduced restrictions on `constexpr`
functions, and thanks to a rich feature-set of `constexpr` functionality
in the standard library (particularly `string_view`).
