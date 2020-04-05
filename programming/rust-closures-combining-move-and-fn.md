# Rust Closures: Returning \`impl Fn\` for \`move\` closures

## Problem setup

Recently I've had to write some `nom` code \(it is a parser combinator [library](https://docs.rs/nom/5.1.1/nom/) for Rust\). To my surprise I discovered that there is no combinator for creating a parser which always succeeds returning a certain given value. At least not without using macros which is [discouraged](https://github.com/Geal/nom/blob/master/doc/upgrading_to_nom_5.md#from-macros-to-functions) in nom v5. That combinator would be something like `pure` in Haskell Parsec. It's not very useful on its own, but can be used as part of other combinators, say providing a default alternative for `alt`.

So I decided to add `success` to nom. After looking at the library code, I realised that it uses closures quite heavily and I didn't use them much in Rust, so I had some questions. Here is my version of `success` basically copy-pasted from a similar combinator `value`:

```rust
pub fn success<I: Clone + Slice<RangeTo<usize>>, O: Clone, E: ParseError<I>>(val: O) -> impl Fn(I) -> IResult<I, O, E>
{
  move |input: I| {
    Ok((input, val.clone()))
  }
}
```

That type signature looks a little scary, eh? However, since we are not going to focus on `I` or `E` arguments here \(input and error types\), we can just rewrite it like this, omitting irrelevant details:

```rust
pub fn success<I, O: Clone, E>(val: O) -> impl Fn(I) -> IResult<I, O, E>
{
  move |input: I| {
    Ok((input, val.clone()))
  }
}
```

## Questions

I had three questions here:

1. Why do we need to clone `val`? After all it looks like l I have a value and just want to pass the ownership to the parser, no need to clone anything.
2. Why we have `move` closure, but return type of the function is `impl Fn(something)` and not `impl FnOnce(something)` I thought that when we use `move` then we move the captured environment into the closure and `FnOnce` trait matches that behaviour.
3. Can we omit `move` or change the type to `FnOnce` or remove `Clone` i.e. to remove any of those things which I didn't understand and still make it work? Are they actually necessary?

**TL;DR** `move` ****determines how captured variables are **moved** into the returned closure. Then returned `impl Fn/FnMut/FnOnce` puts restrictions on how they are **used** inside that closure \(which in turn defines whether the closure can be used once or more\). We can `move` into closure but still only use the captured values by reference and return `impl Fn` to allow multiple calls of the returned closure. And yes, everything in the code above was necessary :\)

## More detailed answers

I assume here that you know the basics about closures. If not, you can read a corresponding chapter in the Rust book. Also, on top of that I would recommend reading Steven Donovan's post ["Why Rust Closures are \(Somewhat\) Hard"](https://stevedonovan.github.io/rustifications/2018/08/18/rust-closures-are-hard.html). 

That post \(and Rust reference\) tells you that a closure basically corresponds to an anonymous structure of some unknown type which has fields corresponding to the captured variables. The capture mode of those fields \(i.e. whether they are `&T`, `&mut T` or `T`\) is determined by the usage of the captured variables inside the closure. Or it can be forced to be `T`, i.e. to passing the ownership to the closure, by using `move` keyword.

I'll repeat the code above for your convenience:

```rust
pub fn success<I, O: Clone, E>(val: O) -> impl Fn(I) -> IResult<I, O, E>
{
  move |input: I| {
    Ok((input, val.clone()))
  }
}
```

So, in our case we implicitly have the following structure for our `move` closure, it only captures a single variable `val` of generic type `O`:

```rust
struct ClosureType<O> {
    // where it's O, &O or &mut O is defined by 
    // `val` usages in the closure and presence of `move` keyword 
    val: O; 
}
```

Then we know that this closure should implement `Fn` trait, since this is what is returned from `success` function. As described in the [documentation](https://doc.rust-lang.org/std/ops/trait.Fn.html), it will look like this \(note the helpful comment\):

```rust
impl<I, O, E> Fn<I> for ClosureType<O> {
  type Output = IResult<I, O, E>;

  // This `&self` here is because we implement Fn, not FnOnce!
  // In FnOnce it would be `self`, owning the structure fields.
  fn call(&self, input: I) -> Output {    
      Ok((input, self.val.clone()));
  }
}
```

Now we can answer our **question 1**: why do we have to clone? If we didn't clone, we would be moving out of `self.val` which is behind a reference. So we can't do that for `Fn`. 

Another way of resolving this issue \(i.e. if we don't want to clone\) would be to use FnOnce as the result type which would in effect give us `self` instead of `&self` in the call method, so we can pass the ownership further with `Ok((input, self.val))`. However, using FnOnce means that we can use the resulting closure just once, perhaps that's not we want in a parser. Why? I don't know for sure, but I suspect that we may want to do certain lookaheads while parsing and this means invoking the closure and then backtracking if it doesn't match. But this mean we would have spoiled the parser closure and can't use it again for another run. So let's assume that our parsers should be `Fn`s which is indeed Nom's API.

The **question 2:** how `move` can coexist with returning `Fn` is also clear now. `move` determines how values are **captured** into that closure-structure, i.e. which type they have there \(refs, mutable refs or owned values\) while `Fn/FnOnce/FnMut` trait is determined by the way they are **used** in that closure \(in our `call` method above\). For example, we can `move` into a closure but still only use the captured values by reference and return `impl Fn` to allow multiple calls of the returned closure.

The remaining part of **question 3** is why we want to use `move` closure here if we only access the variable by reference anyway. The explanation is that we are **returning** this closure, but `val` will be dropped immediately on return and the closure can't outlive it. In other words, if we didn't use `move`, we would have `val: &'a O` in that closure structure field. But that reference would immediately become invalid since when we return our closure, `val` is dropped, so no references to that are allowed, including inside our returned closure. So that won't work and we would get "val does not live long enough" error message.

Therefore, it looks like all the bells and whistles in that `success` combinator were actually needed.

## Summary

I think the main conclusion of this investigation is that explicitly writing out implicit structures and Fn/FnMut/FnOnce implementations is very useful for making sense of compiler errors and understanding what's going on under the hood in the world of closures. At least for the first time.

## Further reading

* Rust reference: ["Closure expressions"](https://doc.rust-lang.org/stable/reference/expressions/closure-expr.html), ["Closure types"](https://doc.rust-lang.org/stable/reference/types/closure.html) \(it even has a special note about combination of `move` closures and `Fn` [here](https://doc.rust-lang.org/stable/reference/types/closure.html#call-traits-and-coercions)\).
* ["Why Rust Closure are \(Somewhat\) Hard"](https://stevedonovan.github.io/rustifications/2018/08/18/rust-closures-are-hard.html) blog post
* [Fn](https://doc.rust-lang.org/std/ops/trait.Fn.html), [FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html) and [FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) in standard documentation

**Update**: as [/u/CUViper](https://www.reddit.com/user/CUViper/) has suggested on Reddit, there is also a very nice blog post ["Closures: Magic Functions"](https://krishnasannasi.github.io/rust/syntactic/sugar/2019/01/17/Closures-Magic-Functions.html) which meticulously demonstrates the desugaring of various types of closures.

