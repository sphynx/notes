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

A _binding mode_ determines how values are bound to identifiers in patterns. The default binding mode is to move, i.e. we just move those values: `y` is moved into `x` and we can't use it again. If `y` has `Copy` then we copy. Those two binding modes are normally called _binding by value_. There is also _binding by reference_, which is introduced with `ref` keyword attached to an identifier pattern like this: `let ref x = y`. This means that we want to borrow `y` instead of moving/copying it, so `x` will be a reference. It is exactly the same as `let x = &y`. For mutable reference there is `ref mut`, i.e `let ref mut x = y` which is the same as `let x = &mut ref y`.

> A thing to remember. Those two statements are identical:
>
> `let ref x = y;`
>
> `let x = &y;`

Now the question is: if `ref` is just a funky way of saying that you want to borrow, why is it needed? Is just using `&` not enough? `ref` becomes more useful when used in nested patterns.

When I first saw `ref` in Rust code, I thought that it was an antiquated way of borrowing, a remnant from previous Rust versions. It is indeed so, mostly because of the feature called "match ergonomics" which we look into in the next section. This feature mostly obsoleted the need for `ref` and so in modern Rust using `ref` is rarely needed, but no doubt you'll see it a lot in existing codebases. I think it's easy to understand it better by contrasting with other patterns. Let's say we pattern match on an Option with `match`

Then the difference between `Some(ref y) =>` and `Some(&y) =>` in patterns is that the first borrows whatever is in `Some` and the second dereferences the pointer in Some \(remember that we've already covered reference patterns in previous section\).

The difference between `Some(ref y) =>` and some `Some(y) =>` is that in the first case we borrow and in the second case we move/copy into `y`.

## Match ergonomics

In older versions of Rust \(up to [1.26](https://blog.rust-lang.org/2018/05/10/Rust-1.26.html) which was released in May 2018\) when you had a reference to Option and wanted to pattern match against it your life was hard. You had two choices:

1. Use reference patterns and `ref` because: a\) you needed to match references with reference patterns b\) you had to use `ref` because you couldn't move part of Option into `s`, since you only had a borrowed version of Option:

```rust
fn f(x: &Option<String>) {
    match x {
        &Some(ref s) => // do something with `s`
        &None => {}
    }
}
```

2. Dereference the Option pointer and still use `ref` because of the same reasons as in \(1\).

```rust
fn f(x: &Option<String>) {
    match *x {
        Some(ref s) => // do something with `s`
        None => {}
    }
}
```

Both of those choices were unsatisfactory and too verbose for such a common operation. Hence, RFC-2005 "Match Ergonomics" has been proposed implemented. It allowed one to just write this code, without `ref` and reference patterns:

```rust
fn f(x: &Option<String>) {
    match x {
        Some(s) => // do something with `s`
        None => {}
    }
}
```

It works as follows: it sees that `x` is a _reference_ which is matched by _non-reference_ patterns. Therefore it automatically dereferences `x` and uses appropriate binding modes for identifiers in that pattern, in this case bind by reference. In other words, it automatically inserts `ref` for you, since it's the only sensible thing to do in this situation.  

You can learn more details in the [original RFC](https://github.com/rust-lang/rfcs/blob/master/text/2005-match-ergonomics.md), it's amazingly readable, has nice motivating examples \(actually I've stolen my example from there\). Here is a memorable quote from that RFC:

> Match expressions are an area where programmers often end up playing 'type Tetris': adding operators until the compiler stops complaining, without understanding the underlying issues. This serves little benefit - we can make match expressions much more ergonomic without sacrificing safety or readability.

## Case study: tree traversal

Let's start with a case study which actually led me to writing this note. Say I want to implement a data structure for full binary trees \(i.e. each node has either 0 or 2 children\) and traverse it recursively. Of course I'll need pattern matching for that! Armed with our knowledge from previous sections it now seems like an easy task:

```rust
struct T {
    data: u8,
    children: Option<Box<(T, T)>>,
}

fn traverse(s: &T) {
    match &s.children {
        None => {}, // TODO: do something
        Some(ch) => {
            traverse(&ch.0);
            traverse(&ch.1);
        }
    }
}
```

I'll additionally comment on some relatively subtle points to make sure we understand what's going on in this example.

First, what is the type of `s.children`? It is a little bit non-obvious, since `s` is a reference and the we have a field accessor. Knowing that Rust likes to do things automatically for us, we may suspect that something "auto-" is happening here. Indeed, here we have an example of auto-dereferencing. If the left hand side of `.` operator is a pointer, it's dereferenced as many times as needed to get access to the field, as described [here](https://doc.rust-lang.org/reference/expressions/field-expr.html). So basically `s.children` is equivalent to `(*s).children`. And therefore its type is `Option<Box<(T, T)>>`.

Next, since we only have a reference to `T` and can't move out of it \(and don't want to\) we need to match a reference to `s.children` \(as opposed to `s.children` itself\) to kick-in the match ergonomics process. Note that we don't need `&` before `None` and `Some` or `ref` keyword.

Then what is the type of `ch` when it is bound? Remember that we bind by reference here, so it is `&Box<(T, T)>`. You can easily check it with `print_type_of` function listed above.

Now we want to recurse and to get access to our tuple which is hidden behind two references \(`&` and `Box`\). Auto-dereferencing to the rescue! We can just say `ch.0` and this will add  **automatically. Finally, we need to borrow that with &: this operator has lower precedence than field accessors, so &a.b is identical to &\(a.b\) In order to appreciate how much auto- is happening here let's look at the fully parenthesised unambiguous analog of &ch.0: it's the same as**

