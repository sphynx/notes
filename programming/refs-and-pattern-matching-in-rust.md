---
description: >-
  `ref` keyword, match ergonomics, box patterns, references patterns and other
  interesting stuff
---

# References and pattern matching in Rust

## Introduction

In Haskell pattern matching is mostly nice and easy. It might be made more complicated by strictness annotations \(i.e. whether to evaluate sub-patterns lazily or strictly\) or irrefutability annotations, but it doesn't have any special interactions with ownership, borrowing and mutability which are bread and butter of Rust. So, naturally, pattern matching in Rust has additional complexity which I wanted to investigate here. Interestingly enough, some of this material is not covered by current edition of the Rust Book.

The plan of attack is as follows:

* Reference patterns: things like `let &x = &1`
* `ref` keyword: what's the difference between using `Some(ref x) =>` and `Some(&x) =>` in patterns? Maybe no difference?
* So called "match ergonomics": what Rust compiler does under the hood to make our lives easier? \(and maybe less predictable?\)
* Box patterns: things like `box (x, y) =>` in patterns
* Other useful auxiliary notes: binding modes, patterns \(ir-\)refutability, auto dereferencing, printing types for debugging purposes, etc.

## Reference patterns

As the Rust Book tells us, `match` is not the only place where pattern matching occurs. It also happens in many other places: `let`, `if let`, `while let`, function arguments, etc. So, even a simple `let x = 1` exhibits pattern matching: we match 1 against pattern `x`. This is called _identifier pattern_ and it never fails, hence it is called _irrefutable pattern_. We can use only irrefutable patterns with `let`. If we try something like `let Some(x) = Some(1)` it will fail, because we don't cover all the cases. A pattern which may fail is called _refutable pattern_. I am reminding you about those `let` patterns here just because it's easier to use them in examples rather than fully fledged `match` even tough such examples may feel more contrived.

Let's move to the next class of patterns called _reference patterns_. It's a pattern starting with one or two `&` like this: `let &y = &1`. After that we can use `y` and its value would be 1, no surprises:

```rust
fn test() {
    let x = 1;
    print_type_of("x", &x);
    println!("x: {}", x);

    let &y = &1;
    print_type_of("y", &y);
    println!("y: {}", y);
}
```

When run it prints:

```text
type of x: i32
x: 1
type of y: i32
y: 1
```

A quick aside: here I am using a little helper for printing types which uses `std::any::type_name::<T>()` function from stable Rust. This is useful for investigating pattern matching, auto-dereferencing and other situations when compiler may do something strange and the type might not be clear from context:

```rust
fn print_type_of<T>(msg: &str, _: &T) {
    println!("type of {}: {}", msg, std::any::type_name::<T>())
}
```

Ok, back to reference patterns. When do we use this kind of a pattern? It's used for dereferencing the pointers which are being matched. Or you can say that they are used to "destructure" the pointer. For example let's look at a common idiom for mapping a function with an iterator: `xs.iter().map(|&x| x + 1).collect()` Note how we use `&x` pattern here to dereference \(deconstruct\) the pointer returned by the iterator. We can also use `map(|x| *x + 1)` for that, making dereference explicit, but I'd prefer the first variant because one can immediately see from `&x` pattern that a parameter is a reference without looking at its usage. I've run a quick `grep` over Rust standard library looking for iter/map combination and it feels like the first variant is more popular when we need to dereference, but it's probably a matter of taste.

An exercise for the reader: what error message do you expect if you try `let &x = 1`?

## Binding modes and \`ref\` keyword

When we pattern match some expressions against patterns, for example `let x = y` we have several options with respect to binding modes. 

A _binding mode_ determines how values are bound to identifiers in patterns. The default binding mode is to move, i.e. we just move those values: `y` is moved into `x` and we can't use it again. If `y` has `Copy` then we copy. Those two binding modes are normally called _binding by value_. There is also _binding by reference_, which is introduced with `ref` keyword attached to the pattern like this: `let ref x = y`. This means that we want to borrow `y` instead of moving/copying it, so `x` will be a reference. It is exactly the same as `let x = &y`. For mutable reference there is `ref mut`, i.e `let ref mut x = y` which is the same as `let x = &mut ref y`.

> A thing to remember. Those two statements are identical:
>
> `let ref x = y;`
>
> `let x = &y;`

Now the question is: if `ref` is just a funky way of saying that you want to borrow, why is it needed? Is just using `&` not enough? When I first saw `ref` in Rust code, I thought that it was an antiquated way of borrowing, a remnant from previous Rust versions. It is indeed so, mostly because of the feature called "match ergonomics" which we look into in the next chapter. But first, let's try to understand how `ref` is/was used.

Let's say you have an Option with some data inside and you want to pattern match against that data:

```rust
fn analyse(x: &Option<Node>) -> bool {
    match x {
        None => false,
        Some(node) => ...
    };
}
```

TBC

