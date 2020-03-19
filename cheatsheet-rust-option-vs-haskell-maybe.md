---
description: Correspondence of common combinators/functions
---

# Cheatsheet: Rust Option vs Haskell Maybe

This is meant to be for people coming from Haskell to Rust or vice versa. For example, I keep forgetting the names of Rust combinators or which one I have to use in a particular situation \(is it `or_else` or `unwrap_or`? `unwrap_or_else`?\). I can imagine that people going into opposite direction may experience similar problems, hence the cheatsheet.

| Haskell | Rust | Purpose | Type \(Haskell style\) |
| :--- | :--- | :--- | :--- |
| Maybe | Option | type name |  |
| Just | Some | constructor for value |  |
| Nothing | None | constructor for no value |  |
| isJust | is\_some | check if has value | Maybe a -&gt; Bool |
| isNothing | is\_none | check if has no value | Maybe a -&gt; Bool |
| fmap | map |  apply function to value inside | \(a -&gt; b\) -&gt; Maybe a -&gt; Maybe a |
| fromJust | unwrap | extract a value, fail if there is none | Maybe a -&gt; a |
| fromMaybe | unwrap\_or | extract a value or return a given default | a -&gt; Maybe a -&gt; a |
| \(&gt;&gt;=\) | and\_then | propagate "no value", apply a function to a value, function can return no value too | Maybe a -&gt; \(a -&gt; Maybe b\) -&gt; Maybe b |
|  |  |  |  |

