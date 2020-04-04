# Rust closures: combining \`move\` and \`Fn\`

## Problem setup

Recently I had to write some `nom` code \(it is a parser combinator library for Rust\). To my surprise I discovered that there is no combinator for creating a parser which always succeeds returning a certain given value \(at least not without using macros which is discouraged in nom v5\). That would be something like `pure 3` in Haskell Parsec. It's not very useful on its own, but can be used as part of other combinators, say providing a default alternative for `alt`.

So I decided to add `success` to nom. However, when I started looking at the library code, it was not 100% clear for me what's going on there. It uses closures quite heavily and I didn't use them much in Rust. Here is my version of `success` inspired by similar combinator `value` :

```rust
pub fn success<I: Clone + Slice<RangeTo<usize>>, O: Clone, E: ParseError<I>>(val: O) -> impl Fn(I) -> IResult<I, O, E>
{
  move |input: I| {
    Ok((input, val.clone()))
  }
}
```

That type signature looks somewhat scary, eh? However, since we are not going to focus on `I` or `E` arguments here \(input and error types\), we can just rewrite it like this, omitting irrelevant details:

```rust
pub fn success<I, O: Clone, E>(val: O) -> impl Fn(I) -> IResult<I, O, E>
{
  move |input: I| {
    Ok((input, val.clone()))
  }
}
```

I had three questions here:

1. Why do we need to clone `val`? After all I have a value and just want to pass the ownership to the parser, no need to clone anything.
2. Why we have `move` closure, but return type of the function is `impl Fn(something)` and not `impl FnOnce(something)` I thought that when we use `move` then we move the captured environment into the closure and `FnOnce` trait matches that behaviour.
3. Can we omit `move` or change the type to `FnOnce` or remove `Clone` i.e. to remove all those things which I didn't understand and still make it work? Are they actually necessary?

