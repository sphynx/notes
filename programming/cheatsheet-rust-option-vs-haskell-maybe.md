---
description: Correspondence of common combinators
---

# Cheatsheet: Option \(in Rust\) vs Maybe \(in Haskell\)

This is meant to be for people coming from Haskell to Rust or vice versa who want to quickly find the name of corresponding function on optional values. For example, I keep forgetting the names of Rust combinators. Which one I have to use in a particular situation? Is it `or_else`, `unwrap_or` or`unwrap_or_else`? I can imagine that other people may experience similar problems, hence the cheatsheet. You can find examples and more detailed description in the official documentation by clicking on function names.

**Update 1**: as [/u/jkachmar](https://www.reddit.com/user/jkachmar) points out on Reddit that there is a [`note`](https://hackage.haskell.org/package/errors-2.3.0/docs/Control-Error-Util.html#v:note) function in some of alternative Preludes in Haskell \(or in `errors` package\), which is an analog of `ok_or` in Rust. Added to the cheatsheet.

**Update 2:** [/u/masklinn](https://www.reddit.com/user/masklinn) says that Haskell `listToMaybe` is akin to calling `next` on an `Iterator` and `maybeToList` is an `Option` implementing `IntoIterator`. I agree with that, since lists in Haskell, being lazy, more or less correspond to Rust iterators. Also, you can iterate over `Option` in Rust by calling `iter`.

**Update 3:** fixed several typos and errors spotted by Reddit readers. Thank you [/u/jroller](https://www.reddit.com/user/jroller/), [/u/jlombera](https://www.reddit.com/user/jlombera/), [/u/gabedamien](https://www.reddit.com/user/gabedamien/), [/u/george\_\_\_\_t](https://www.reddit.com/user/george_____t/)

**Update 4:** use `flatten` from Iterator to implement `catMaybe`, thanks to [/u/MysteryManEusine](https://www.reddit.com/user/MysteryManEusine/). 

### Cheatsheet

<table>
  <thead>
    <tr>
      <th style="text-align:left">Haskell</th>
      <th style="text-align:left">Rust</th>
      <th style="text-align:left">Purpose</th>
      <th style="text-align:left">Type (Haskell style)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#t:Maybe">Maybe</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html">Option</a>
      </td>
      <td style="text-align:left">type name</td>
      <td style="text-align:left"></td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:Just">Just</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#variant.Some">Some</a>
      </td>
      <td style="text-align:left">constructor for value</td>
      <td style="text-align:left"><code>a -&gt; Maybe a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:Nothing">Nothing</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#variant.None">None</a>
      </td>
      <td style="text-align:left">constructor for no value</td>
      <td style="text-align:left"><code>Maybe a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:isJust">isJust</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.is_some">is_some</a>
      </td>
      <td style="text-align:left">check if has value</td>
      <td style="text-align:left"><code>Maybe a -&gt; Bool</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:isNothing">isNothing</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.is_none">is_none</a>
      </td>
      <td style="text-align:left">check if has no value</td>
      <td style="text-align:left"><code>Maybe a -&gt; Bool</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="https://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Functor.html#v:fmap">fmap</a> from
        Functor</td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.map">map</a>
      </td>
      <td style="text-align:left">apply function to value inside</td>
      <td style="text-align:left"><code>(a -&gt; b) -&gt; Maybe a -&gt; Maybe b</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:fromJust">fromJust</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap">unwrap</a>
      </td>
      <td style="text-align:left">extract a value, fail if there is none</td>
      <td style="text-align:left"><code>Maybe a -&gt; a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:fromMaybe">fromMaybe</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_or">unwrap_or</a>
      </td>
      <td style="text-align:left">extract a value or return a given default</td>
      <td style="text-align:left"><code>a -&gt; Maybe a -&gt; a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Monad.html#v:-62--62--61-">(&gt;&gt;=)</a> from
        Monad</td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then">and_then</a>
      </td>
      <td style="text-align:left">propagate &quot;no value&quot;, apply a function to a value, function
        can return no value too</td>
      <td style="text-align:left"><code>Maybe a -&gt; (a -&gt; Maybe b) -&gt; Maybe b</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Applicative.html#v:-60--124--62-">(&lt;|&gt;)</a> from
        Alternative</td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.or">or</a>
      </td>
      <td style="text-align:left">return first value if present or<b> </b>second if not</td>
      <td style="text-align:left"><code>Maybe a -&gt; Maybe a -&gt; Maybe a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Monad.html#v:-62--62-">(&gt;&gt;)</a> from
        Monad</td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.and">and</a>
      </td>
      <td style="text-align:left">return first value if none or second if not</td>
      <td style="text-align:left"><code>Maybe a -&gt; Maybe b -&gt; Maybe b</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:maybe">maybe</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.map_or">map_or</a>
      </td>
      <td style="text-align:left">takes function and default. Apply function to the value or return default
        if there is no value</td>
      <td style="text-align:left"><code>b -&gt; (a -&gt; b) -&gt; Maybe a -&gt; b</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">
        <p><a href="https://hackage.haskell.org/package/errors-2.3.0/docs/Control-Error-Util.html#v:note">note</a>,
          <a
          href="https://hackage.haskell.org/package/either-5.0.1.1/docs/Data-Either-Combinators.html#v:maybeToRight">maybeToRight</a>(both non-</p>
        <p>standard)</p>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or">ok_or</a>
      </td>
      <td style="text-align:left">transforms optional value to possible error</td>
      <td style="text-align:left"><code>b -&gt; Maybe a -&gt; Either b a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:mapMaybe">mapMaybe</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map">filter_map</a> from
        Iterator</td>
      <td style="text-align:left">applies filter and map simultaneously</td>
      <td style="text-align:left"><code>(a -&gt; Maybe b) -&gt; [a] -&gt; [b]</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:catMaybe">catMaybe</a>
      </td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flatten">flatten</a> from
        Iterator</td>
      <td style="text-align:left">extracts only values from the list, drops no values</td>
      <td style="text-align:left"><code>[Maybe a] -&gt; [a]</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="https://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Monad.html#v:join">join</a> from
        Monad</td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.flatten">flatten</a>
      </td>
      <td style="text-align:left">squashes two layers of optionality into one</td>
      <td style="text-align:left"><code>Maybe (Maybe a) -&gt; Maybe a</code>
      </td>
    </tr>
    <tr>
      <td style="text-align:left"><a href="http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Traversable.html#v:sequence">sequence</a> from
        Traversable</td>
      <td style="text-align:left"><a href="https://doc.rust-lang.org/std/option/enum.Option.html#method.transpose">transpose</a>
      </td>
      <td style="text-align:left">transposes Option and Result layers (or Either and Maybe in Haskell terms)</td>
      <td
      style="text-align:left"><code>Maybe (Either e a) -&gt; Either e (Maybe a)</code>
        </td>
    </tr>
  </tbody>
</table>

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

Haskell has two functions `listToMaybe` and `maybeToList` that convert between trivial lists \(with 0 or 1 elements\) and `Maybe` values. Rust doesn't have those, since lists are not that ubiquitous, but see the Update 2 above.

### Summary

Rust has more functions to work with `Option` than Haskell because it has to support references, mutability and ownership. On the other hand Haskell outsources some of the combinators to its generic typeclasses: `Semigroup`, `Alternative`, `Monoid`etc. so its combinator library seems thinner.

