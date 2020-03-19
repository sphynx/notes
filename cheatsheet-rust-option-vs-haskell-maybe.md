---
description: Correspondence of common combinators
---

# Cheatsheet: Option \(in Rust\) vs Maybe \(in Haskell\)

This is meant to be for people coming from Haskell to Rust or vice versa who want to quickly find the name of corresponding function on optional values. For example, I keep forgetting the names of Rust combinators. Which one I have to use in a particular situation? Is it `or_else`, `unwrap_or` or`unwrap_or_else`? I can imagine that other people may experience similar problems, hence the cheatsheet. You can find examples and more detailed description in the official documentation by clicking on function names.

### Cheatsheet

| Haskell | Rust | Purpose | Type \(Haskell style\) |
| :--- | :--- | :--- | :--- |
| [Maybe](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#t:Maybe) | [Option](https://doc.rust-lang.org/std/option/enum.Option.html) | type name |  |
| [Just](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:Just) | [Some](https://doc.rust-lang.org/std/option/enum.Option.html#variant.Some) | constructor for value |  |
| [Nothing](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:Nothing) | [None](https://doc.rust-lang.org/std/option/enum.Option.html#variant.None) | constructor for no value |  |
| [isJust](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:isJust) | [is\_some](https://doc.rust-lang.org/std/option/enum.Option.html#method.is_some) | check if has value | `Maybe a -> Bool` |
| [isNothing](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:isNothing) | [is\_none](https://doc.rust-lang.org/std/option/enum.Option.html#method.is_none) | check if has no value | `Maybe a -> Bool` |
| [fmap](https://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Functor.html#v:fmap) | [map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map) |  apply function to value inside | `(a -> b) -> Maybe a -> Maybe a` |
| [fromJust](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:fromJust) | [unwrap](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap) | extract a value, fail if there is none | `Maybe a -> a` |
| [fromMaybe](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:fromMaybe) | [unwrap\_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_or) | extract a value or return a given default | `a -> Maybe a -> a` |
| [\(&gt;&gt;=\)](https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Monad.html#v:-62--62--61-) | [and\_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then) | propagate "no value", apply a function to a value, function can return no value too | `Maybe a -> (a -> Maybe b) -> Maybe b` |
| [\(&lt;\|&gt;\)](https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Applicative.html#v:-60--124--62-) | [or](https://doc.rust-lang.org/std/option/enum.Option.html#method.or) | return first value if present or ****second if not | `Maybe a -> Maybe a -> Maybe a` |
|  | [and](https://doc.rust-lang.org/std/option/enum.Option.html#method.and) | return first value if none or second if not | `Maybe a -> Maybe a -> Maybe a` |
| [maybe](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:maybe) | [map\_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.map_or) | takes function and default. Apply function to the value or return default if there is no value | `b -> (a -> b) -> Maybe a -> b` |
|  | [ok\_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or) | transforms optional value to possible error | `b -> Maybe a -> Either b a` |
| [mapMaybe](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:mapMaybe) | [filter\_map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map) from Iterator | applies filter and map simultaneously | `(a -> Maybe b) -> [a] -> [b]` |
| [catMaybe](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:catMaybe) |  | extracts only values from the list, drops no values | `[Maybe a] -> [a]` |
| [join](https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Monad.html#v:join) | [flatten](https://doc.rust-lang.org/std/option/enum.Option.html#method.flatten) | squashes two layers of optionality into one | `Maybe (Maybe a) -> Maybe a` |

### Notes

In Rust, all combinators with `or` at the end have a variant with `or_else` at the end: `unwrap_or_else` or `or_else` etc. Those variants take a closure for the default value and are lazily evaluated. They are recommended when you have a function call returning default value. In Haskell there is no need for this, since it is a lazy language by default.

In Rust there are many different ways for extracting the optional value:  

* `unwrap` which just fails if there is no value
* `unwrap_or` which provides a default for that
* `unwrap_or_else` where this default is calculated by a closure
* `unwrap_default` where default is taken from `Default` trait implemented on the type
* `unwrap_none` which fails if the value is **not** `None` and returns nothing
* `expect` which is the same as `unwrap`, but takes a custom error message
* `expect_none` which is the same as `unwrap_none` but takes a custom error message

Another very useful method in Rust is `as_ref` which converts from `&Option<T>` to `Option<&T>` which is very handy for pattern matching. `as_mut` plays a similar role for mutable references.

In Rust there are several methods related to ownership of the value in the `Option`, like `take` and `replace`, they have no analogs in Haskell that does not have the ownership concept.

In Rust we sometimes want to copy or clone values, it's possible to do so on optional references \(`Option<&T>`\) to get `Option<T>` from those by cloning or copying the referenced value. There are `copied` and `cloned` methods for this.

It's possible to mutate optional values in Rust. For that we have `get_or_insert` and `get_or_insert_with` methods which allow to insert a new \(possibly computed\) value into `None` or just use the value which was there.

`transpose` method in Rust is interesting, since it reminds me of `sequence` from `Traversable` in Haskell. It basically transpose two layers: `Result` \(or `Either` in Haskell\) and `Option`. `sequence` in Haskell does approximately the same, but in a more generic fashion.

In addition to `and` and `or` Rust has a method `xor`, perhaps just for the completeness. You've probably guessed that it returns `Some` if and only if there is only one `Some` in its arguments.

Haskell has two functions `listToMaybe` and `maybeToList` that convert between trivial lists \(with 0 or 1 elements\) and `Maybe` values. Rust doesn't have those, since lists are not that ubiquitous.

### Summary

Rust has more functions to work with `Option` than Haskell because it has to support references, mutability and ownership. On the other hand Haskell outsources some of the combinators to its generic typeclasses: `Semigroup`, `Alternative`, `Monoid`etc. so its combinator library seems thinner.

