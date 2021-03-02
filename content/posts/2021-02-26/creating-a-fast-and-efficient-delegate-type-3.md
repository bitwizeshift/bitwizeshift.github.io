---
title: "Creating a Fast and Efficient Delegate Type (Part 3)"
subtitle: "Optimizing Delegate to have zero overhead"
date: 2021-02-26T22:19:51-05:00
draft: false
tags: [performance, optimizing, templates]
categories: [intermediate-templates, tutorial, c++17]
series: [delegate]
summary: "
In the previous post we reworked our `Delegate` object into something more
modern and easy-to-use by using `c++17` -- but is this utility zero-overhead?
In this post we will look at how to optimize this into a true zero-overhead
utility.
"
---

{{<info>}}
**Note:** This is part 3 of a 3 part series.
{{</info>}}


In the [previous post](/posts/2021-02-26/creating-a-fast-and-efficient-delegate-type-2)
we updated our `Delegate` object that we've been working on since
[the first post](/posts/2021-02-24/creating-a-fast-and-efficient-delegate-type)
to support covariance.

In this post, we will look at how to make this a true **zero-overhead** utility

{{<table-of-contents>}}

## Goal

To ensure this `Delegate` has exactly **zero-overhead**.
In particular, lets make sure that this `Delegate` is exactly as lightweight to
use as a raw pointer.

## Testing for zero overhead

The current `Delegate` implementation certainly has a very small memory
footprint, weighing in at one `const void*` and a raw function pointer.
But how does this perform? Does this do any extra work that it shouldn't?
Is it as _lightweight_ as just a function pointer?

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

The same will also need to be done for the member function versions.
Now we can see the `bind` call succeed.

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tQVvQAZPLUwA5YwCNMxEAA4AzKQAOqBYXW0eoYmgj5%2BanRWNvZGTi4eisqYqgEMBMzEBEHGppyJKhG0aRkEUXaOzm6eCumZ2SF5NSVlMXFVAJSKqAbEyBwA5FLu1siGWADU4u46mAAefV6FU9jiAAwAgkMjY5iT0ywKSpnLa5uSw7SjBhNTOlQGl4VsJxuno8yH49LM6AAiScBmARMDo2KxxiBxl4DA5WHhkOMaugQCA5gtCpMAOyyDbQ2HwkCncbE8bMAxEcYAdwQQIg7XGaFoNXGtFQaMwi3GAFplgy6MzkDTiAAqcaoABuzmIeBu2KJJIVxEwBB6tEmkhkP3GWHYgOBDLBUkkUxx6wV4kxv1OFqt7lNp2BRi8wn1twIAE8vFpmEZdgw8MBaECepgXut3p9/rqgaG7daNo7nTG9joPV6g77xgAlUjjNPe30AOmL62IwAUYYjCnGUcwetjOizEFL5eLhfaYYtprxcOQhI2xIA9IPxjolTHqyJxg8HN1tNqATH5bX63S9r8F1QyawCCb42bxsPxgB1XaUkQEPMIXYr5MUpx8rzu5iwzC5vyXXaEatw4AIAiUpgAb/sui7AhAjLMrewJSAAbGKBDXsQ9JTBuWBbgYO57gOpLkqgYpesQQIkKhEH8pe0GhpI8GoIhzj0jy7jYDWYFUfBqGbtuu5xq8OGJi6uy3GSFIAGIPCkdCkPKCoyXmnoFoJ7gbkiKLeq%2BAD6eBUOp3E6CpIB4Aomm0OKqAsBpxDqeKtw5guoz5hAYmPAEnQtgobYnExYbEsJ%2BEONY6BroxzGmTK8pdtJ4xGMZTSfuuLKYawizENhB7EtFNQwvFTRqMg6kHLpmUOOp9zOXQywQOIACsshVb8ZFMpeoXoMKuZuW2GTlgxvJZhFOGKsqqqIgQyIGSZqAANaYI54mFO%2BI0ojQxDnsQ6C3G5FWdQo7Rth2PFpVivx7aaxI2vuxL8cmQl4eMACymCxM4TkSbQub5hmuw6MIhxSf1skku9PqKcpC2okGGlaTptz6YZxmmeZ7DqZZ1nTLZWD2fJED3Y9xDPYUnSQZeX0fAorXjO1xaeSsXk4b54z%2BdoDXMsThyiqM23cryzXhXKf1Hj86BKocqBUNOSjoHm%2BFaAoIZUrswDKleuxeMQqCEVCqDWMCKV/dF1ixX08Xs6lkUZSqDjZekuX5R8hXmyVs0BBV1W1fVhPjM1opeG1ZbucWW3dUx2Z9QdQ4jqCzIOMwyATZLSt8sQSqqHJXqRcS7u%2BWzltAvCNs1Lc7ss6TFVeMdafjEqKrEGqMPjVNWMPXEeMBLmyDzaNS0rWt0wbUxEAB7tqXmpaZc4WdvEHpdrrTHT2NN47kkpwpo7fQov2h/9gOZhx%2BlqYjkN26NsPWPDL6I8jNm5ujO6Y3PT0L7QnRF2TFOFlT3m4feAUQM/DKsBzwUPaazWmPXmB09aNREIbDixt9oKjNllHeVtc4FWhvbUqL1nY1Wqm7ciQCZRex9q2f2vtA7MV6mA2SmcGRG3InnXSz8Ko5RQbbAueDGF91LqPDelchq11MvXO%2BuMH6t3botEgXd1q%2B02qQwecCSQ2m4adS051P74TVs4YixA6TNl9h1UhfJGqcyDr1UBJ0SRaXGBABBFtUIcVoIlZKKFKEyUQqrSkXwfiUVBKwVgEUVHyOUVaP6vDq5WOFDY9o1iYrpE/LmLacjzHj02LiaU4oYz9hSQeAwH5gDDRhA7MqaoOJNmFFE92nsiF%2B3bCbHCFTgGiggQbYGCVfHJRNIeYctxyYay1s4OOdFxj61iYbCAljCDjDmIZAg215RFUKS9KK6kirxQcW0ggKU7QkmHIObpZofB9OIAM684wMEYgpCfSaoZXgBPtK8c42xriKR0L6IwJB3Sdg2HTYENQID6QeHgAAjgYTA6l2nTC1ssMh%2BCJb%2BOCVkrWUVmDWDXCHYklFbjNT%2BaDAFwLQXgp0JCpie1mJrTlLc%2BU6BCwM27joOCKIfncWwHSE2lp%2BidFYCAfoVV%2BikFMP0VYvLUBcrpTIOQiJui9EEucTgvKCBcsFe0ToE0QBVVWEILl3BeX8sFaQYV/ReUKBAOq%2BVAr2WkDgLAJAcxkh4TIBQfuvsAAKIhlAMAQKgSk/LZWkDQE6PALoAgupsKwd1nqdW8r9V4ANlRgCcFWJIX1qB/XsGIAAeXJGGr1CreU2uQG5LlubZjJDSPgflvKaD0CYGwDgPB%2BCCGEKIFAcg5BCDwA4I1kBOhq0KEa/oXI03uG5EiVCEgxUyE4JiQ1kq%2BiCCRNYYNbqPXZv6D6ykREvBctlRyrlPK%2BU5r1Vy2YrhYJclgtwcYwBkAInjYWSQ4xsDFuQHaqxuBCAkHVO4Tg9JRWyBkHKnNSrSAqrVRq/oWr91msPQaxQxrSCmsVTu/okhtUHv1QBs1QHJTED8BobgQA%3D%3D%3D"
>}}

### Invoking functions with move-only parameters

Now what about invocations? Lets see what happens when we try to invoke the
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

{{< figure
  src="https://img.shields.io/badge/try-online-blue.svg"
  alt="Try Online"
  link="https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAM1QDsCBlZAQwBtMQBGAFlICsupVs1qhkAUgBMAISnTSAZ0ztkBPHUqZa6AMKpWAVwC2tEAGZJpLegAyeWpgByxgEaZiIABwB2UgAdUBUJ1Wj1DE3NLAKC1OjsHZyM3Dx9FZUxVEIYCZmICMONTCzSVWNps3IJ4p1d3L18FHLyCiOLGyurE5PqASkVUA2JkDgByKTN7ZEMsAGpxMx1MAA9hvzL57HEABgBBccnpzDmFlgUlPI3tvckJ2imDWfmdKgM7srZL3aup5jOZ6WY6AAIulgMwCJgdGxWDMQDM/AYXKw8MgZo10CAQMtVmU5t5ZLsEUiUSArjNyTNmAYiDMAO4IcEQHozNC0RozWiobGYNYzAC0GxZdHZyAZxAAVDNUAA3dzEPCPfFkikq4iYAiDWhzSQyQEzLDsMEQlnQqSSeYEnYq8TeIFXG12syWq4Qox%2BYTGp4EACefi0zCMRwYeGAtHBg0wnx2Pz%2BIMN4MjTvtu1d7oTxx0Pr9YcDMwASqQZln/YGAHTlnbEYAKKMxhQzOOYI2JnR5iCV6vl0s9KM2y1E5HIUm7ckAelHMx0aoT9ZEM1eLgG2n1oITysbzaZxyBK6oVNYBAtyatM3HMwA6kdaSICEWEEcN%2BmaW4hX5vcwkZhC0E7kdCPXkWABACFpTAQ2A9dVwhCBWXZR8ISkAA2KUCHvYhmXmHcsD3AwDyPEdKWpVApT9YhwRITCYOFW94MjSRkNQVD3GZAUzGwBsoLo5DMN3fdDyTL4CNTD0jieKkaQAMVeTI6FIZUVQUotfRLUSzB3dFMX9T8AH08CobT%2BJ0DSQDwBRdNoaVUBYHTiG06UngLFcpmLCApLeEI%2Bg7BQu0uNio3JcTiJcex0C3Vj2MshVlT7eSZiMcz2l/bcOVw1g1mIfCT3JeLGkRZL2jUZBtNOQzcpcbSXncugNggcQAFZZDqoEqLZW9IvQcVCy8rtcmrFjBTzGKCNVdVNTRAgMRMizUAAa0wVzpLKb8JsxGhiGvYh0CeLyat6hQei7HsBKyvEgSOy1yQdY9yWE9MxKImYAFlMCSdw3Jk2hC2LHMjh0YQzjk4bFIpb6A1U9SVqxMMdL0gynmM0zzMs6z2G02z7IWRysGc5SIGe17iHeso%2Blg28/t%2BBROpmbry18zY/IIwKZmC7QWvZcmzklKZ9v5QV2uipUgbPQF0DVM5UCoeclHQItiK0BQIzpI5gHVO8jj8YhUFI%2BFUHsCEMqB%2BL7ES4Zku5zLYpyjUXHynJCuK35SutirFpCGr6sa5rSZmdrJT8Lqq288s9v6tj8yGk6xwnKF2RcZhkBm2W1aFYg1VUJS/Vi8lvcCrnbfBFEHcaJ5vY5ymar8c6s5mNUNWILUEemua8Ze5IiZCQtkGWya1o2raFh2tiIBDw7MutW0q4Iq7BJPW7PQWJn8bb13ZIzlTJ3%2BhRAcj4HQdzHjjK01HYadybEfsZGP1R9GHMLbGD1xpe3pX2g%2BjLqmadLOn/MI58Qogd%2BLJWA83Cj7XWW0p6CxOkbVqIhTY8XNsdFUVs8oHztoXEq8NnaVQ%2Bu7Bq9UvbUTAQqP2AdOzB0DqHdig0oGKVziyM21Ei6GXfjVAqGDHYlyIawoeldJ471rmNRullm5P0Ji/Tu3dVokD7ttQOu1KGjyQRSB0/DLq2mukWF6aZ56ZmUj9LsABVLy29gYKT3uDcak0j6YF0vpU%2BmJz7TRRrYjGrZ2yB0OqQYxnjaZ%2BX8ZsRmD0tbuHIsQJkEAfHViQkhHqlDeZh0GpAi6FI9IzAgCgm2mEeK0FSulDCtDzEIE1rSf4gJaJQlYKwGKGjlHqLtEDQR9crGOKbvNTJhYYEmy/C0kAvdcj9x0FEmsQ8R7ljUadY8Gs8DSgTMOPYBEDA/mAONRELsqpah4m2cUPQ2ZtXAR/Xx3YLYEW9r7OKCUchJRyXkggGUnSnnHE8amOs9buCTkxGYxsrmmwgGkwgMxlimQIPtZUZV1kfQuWVZKuSqnpQtBScco5nlWgCG84gHz7wzBwbiGkF9ZqRi%2BLU50XwbgHAeKpHQgYjAkG9L2XYTMISNAgMZV4eAACOBhbHwoWHrDYVDiEyxqQ0hZOw9ZxWYPYLcEdyS0SeO1FlkM2Wcu5Xcp4fK2JHXYltJUxLlToFLCzQZSFMRMv4tgJkJyTyhWMkYZgc1tLKq5eq%2BgNVuCSB6Pwh0Iw%2BisBACMOqIxSCmBGFsINqB/U6DkHINEAwhiiRuJwINBB/Vhs9aQGaIA6pbCEP67gQaQ1htIBGkYQaFAgBzSm0NPrSBwFgEgZYGQiJkAoMPQOAAFEQygGAIFQLSENSbSBoDdHgD0IRO0OFYD2vthag3Dr8KOuowBOBbEsPOxdxAADy1Jp39tTUGxtyAvL%2BoPUsDI2R8AhqDTQegTA2AcB4PwQQwhRAoGjTIIQeAXDlsgH0LWZRy0jD5Jusw/J0SYQkDIOQnBvBlrjcMQQ6J7ATu7b2vdIxB20jIn4f1SbfX%2BsDcG/dxb/VLE8IhPkiFuAzGAMgVEK7SySBmNgM9yBm3pNwIQEg2ozCcGZFGqDMhk37vTZm7NuaRj5qI9WkjpbFAVtIFWtN%2BGRiSALcRktwnq3ptlMQIIGhuBAA%3D%3D"
>}}

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

{{<
  figure src="/img/2021-02-24/benchmark-1.png"
  title="Our current `Delegate` implementation compared to a raw function pointer"
  link="https://quick-bench.com/q/7Ft0Q82qSOTvmx1lE1nx6MOA5jA"
>}}

We can see here that `Delegate` performs _ever so slightly_ slower than
a raw function pointer does. This appears to be a consistent, though tiny,
overhead.

Is there any way that we could potentially address this?

## Optimizing our `Delegate`

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

{{< figure
  src="/img/2021-02-24/benchmark-2.png"
  title="Our optimized `Delegate` implementation compared to a raw function pointer"
  link="https://quick-bench.com/q/1XppGG1P12SjBfu1sZdj1qLWJVw"
>}}

This is what we wanted to see!

We have successfully optimized our `Delegate` class to be exactly as fast as a
raw-function pointer, while also allowing binding class member functions as
well.

## Closing Thoughts

So there we have it.

We managed to check off all of the criteria from the goals laid out at the
start. This is just one such example where a few templates and some creative
problem solving can lead to a high-quality, 0-overhead solution.

This works well in `signal`-patterns, or in class delegation callbacks where the
lifetime of the callback is never exceed by the object being called back.

There are certainly other improvements that could be added, such as:

* Adding support for non-owning viewing callable objects
* Storing small callable objects in the small-buffer storage
* Support for empty default-constructible callable objects (such as `c++20`
  non-capturing lambdas)
* Factory functions to produce lambdas and deduce the return type

These suggestions are left as an exercise to the reader.
