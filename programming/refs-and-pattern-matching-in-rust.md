---
description: >-
  `ref` keyword, match ergonomics, box patterns, references patterns and other
  interesting stuff
---

# References and pattern matching in Rust

## Introduction

In Haskell pattern matching is mostly nice and easy. It might be made more complicated by strictness annotations \(i.e. whether to evaluate sub-patterns lazily or strictly\) or irrefutability annotations, but it doesn't have any special interactions with ownership, borrowing and mutability which are bread and butter of Rust. So, naturally, pattern matching in Rust has additional complexity which I wanted to investigate here. Interestingly enough, most of this material is not covered by the Rust Book.

The plan of attack is as follows:

* `ref` keyword: what's the difference between using `Some(ref x) =>` and `Some(&x) =>` in patterns? Maybe no difference?
* Reference patterns: things like `let &x = &1`
* So called "match ergonomics": what Rust compiler does under the hood to make our lives easier? \(and maybe less predictable?\)
* Box patterns: things like `box (x, y) =>` in patterns
* Other useful auxiliary notes: binding modes, patterns \(ir-\)refutability, auto dereferencing, printing types for debugging purposes, etc.

## Reference patterns. Refutability.

As the Rust Book tells us, `match` is not the only place where pattern matching occurs. It also happens in many other places: `let`, `if let`, `while let`, function arguments, etc. So, even a simple `let x = 1` exhibits pattern matching: we match 1 against pattern `x`. This is called _identifier pattern_ and it never fails, hence it is called _irrefutable pattern_. We can use only irrefutable patterns with `let`. If we try something like `let Some(x) = Some(1)` it will fail, because we don't cover all the cases. A pattern which may fail is called _refutable pattern_. I am reminding you about those `let` patterns here just because it's easier to use them in examples rather than fully fledged `match` even tough such examples may feel more contrived.

Let's move to the next class of patterns called _reference patterns_. It's a pattern starting with one or two `&` like this: `let &y = &1`. After that we can use `y` and its value would be 1, no surprises. 

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

A quick aside: here I am using a nice function for printing types which uses `std::any::type_name::<T>()` function from stable Rust. This is useful for investigating pattern matching, auto-dereferencing and other things when compiler does something weird under the hood and the type might not be clear from context:

```rust
fn print_type_of<T>(msg: &str, _: &T) {
    println!("type of {}: {}", msg, std::any::type_name::<T>())
}
```

Ok, back to reference patterns. Why would we need this kind of a pattern? It's used for dereferencing pointers on the right hand side. For example a common idiom for mapping a function with iterator: `xs.iter().map(|&x| x + 1).collect()` Note how we use `&x` pattern here to dereference the element returned by iterator. We can also use `map(|x| *x + 1)` for that but I like the first variant because one can immediately see from `&x` pattern that a parameter is a reference and is borrowed. 



