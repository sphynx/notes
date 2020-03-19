---
description: Correspondence of common combinators
---

# Cheatsheet: Rust Option vs Haskell Maybe

This is meant to be for people coming from Haskell to Rust or vice versa. For example, I keep forgetting the names of Rust combinators or which one I have to use in a particular situation \(is it `or_else` or `unwrap_or`? `unwrap_or_else`?\). I can imagine that people going into opposite direction may experience similar problems, hence the cheatsheet.

| Haskell | Rust | Purpose | Type \(Haskell style\) |
| :--- | :--- | :--- | :--- |
| Maybe | Option | type name |  |
| Just | Some | constructor for value |  |
| Nothing | None | constructor for no value |  |
| isJust | is\_some | check if has value | `Maybe a -> Bool` |
| isNothing | is\_none | check if has no value | `Maybe a -> Bool` |
| fmap | map |  apply function to value inside | `(a -> b) -> Maybe a -> Maybe a` |
| fromJust | unwrap | extract a value, fail if there is none | `Maybe a -> a` |
| fromMaybe | unwrap\_or | extract a value or return a given default | `a -> Maybe a -> a` |
| \(&gt;&gt;=\) | and\_then | propagate "no value", apply a function to a value, function can return no value too | `Maybe a -> (a -> Maybe b) -> Maybe b` |
| \(&lt;\|&gt;\) | or | return first value if present or ****second if not | `Maybe a -> Maybe a -> Maybe a` |
|  | and | return first value if none or second if not | `Maybe a -> Maybe a -> Maybe a` |
| maybe | map\_or | takes function and default. Apply function to the value or return default if there is no value | `b -> (a -> b) -> Maybe a -> b` |
|  | ok\_or | transforms optional value to possible error | `b -> Maybe a -> Either  b a` |
| mapMaybe | filter\_map from Iterator | applies filter and map simultaneously | `(a -> Maybe b) -> [a] -> [b]` |
| catMaybe |  | extracts only values from the list, drops no values | `[Maybe a] -> [a]` |
| join | flatten | squashes two layers of optionality into one | `Maybe (Maybe a) -> Maybe a` |

In Rust, all combinators with `or` at the end have a variant with `or_else` at the end: `unwrap_or_else` or `or_else` etc. Those variants take a closure for the default value and are lazily evaluated. They are recommended when you have a function call returning default value. In Haskell there is no need for this, since it is a lazy language by default.

In Rust there are several functions related to ownership of the value in the `Option`, like `take` and `replace`, they have no analogs in Haskell that does not have the ownership concept.

In Rust we sometimes want to copy or clone values, it's possible to do so on optional references \(`Option<&T>`\) to get `Option<T>` from those by cloning or copying the referenced value. There are `copied` and `cloned` functions for this.

It's possible to mutate optional values in Rust. For that we have `get_or_insert` and `get_or_insert_with` functions which allow to insert a new \(possibly computed\) value into `None` or just use the value which was there.

In Rust there are many different ways extracting the optional value:  

* `unwrap` which just fails if there is no value
* `unwrap_or` which provides a default for that
* `unwrap_or_else` where this default is calculated by a closure
* `unwrap_default` where default is taken from `Default` trait implemented on the type
* `unwrap_none` which fails if the value is **not** `None` and returns nothing
* `expect` which is the same as `unwrap`, but takes a custom error message
* `expect_none` which is the same as `unwrap_none` but takes a custom error message

Haskell has two functions `listToMaybe` and `maybeToList` that convert between trivial lists \(with 0 or 1 elements\) and `Maybe` values. Rust doesn't have those, since lists are not that ubiquitous.

