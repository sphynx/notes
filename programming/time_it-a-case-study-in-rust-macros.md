---
description: My adventures in Declarative Macros Land
---

# time\_it: a case study in Rust macros

### Problem setup

One day I wanted to quickly print for how long certain pieces of code run without setting up the whole profiling business. 

I used [`std::time::Instant`](https://doc.rust-lang.org/std/time/struct.Instant.html) for that, it's nice and easy and gives access to monotonic clock on your OS:

```rust
let timer = std::time::Instant::now();
let x = do_stuff();
println!("do_stuff: {:?}", timer.elapsed());

// Output:
// do_stuff: 2.12s
```

Then I thought that it would be convenient to have a macro `time_it` which I can use to wrap any statement, block of code or even many statements and measure their timings. I had in mind something like context managers in Python, where you can do this:

```python
with Timer("part 1"):
    do_a()
    do_b()
    do_c()
```

Context managers can "automatically" run some code before starting evaluation of that block \(e.g. start a timer\) and do something after leaving the block \(e.g. stop the timer and print elapsed time\). 

So, let's try to declare a macro in Rust which does something similar. My goal is to be able to annotate the code with `time_it!` macro in the least intrusive way \(ideally without changing the measured lines at all\).

### Macro for timing a single statement

```rust
macro_rules! time_it {
    ($context:literal, $s:stmt) => {
        let timer = std::time::Instant::now();
        $s
        println!("{}: {:?}", $context, timer.elapsed());
    };
}

// Let's test it:
fn main() {
    time_it!("println", println!("hello, world!"));
    
    let x = 1;
    time_it!("let", let _y = x + 2);
}

/* This will print:

hello, world!
println: 31.143µs
let: 43ns
*/
```

So far so good. I'm giving `time_it` a context: string literal to describe what we measure and one statement and it prints timings nicely \(even with fancy Greek letters in "µs" for microseconds!\).

However, even this simple macro contains several things which puzzled me and didn't work as expected.

### First nuance: \`stmt\` fragment specifier and semicolons

The first thing is that statement argument of the macro does not have a trailing semicolon, which is not ideal, since when I'm wrapping an existing statement terminated with a semicolon I have to remove it:

```rust
// before:
println!("x"); // must be terminated with a semicolon

// after wrapping:
time_it!("println",
  println!("x") // <- no semicolon here
)
```

What happens if I leave the semicolon in the macro argument? It fails with this error:

```rust
time_it!("println", 
  println!("hello, world!");  // with semi-colon
);


error: no rules expected the token `;`
  --> src/main.rs:31:50
   |
1  | macro_rules! time_it {
   | -------------------- when calling this macro
...
31 |     time_it!("println", println!("hello, world!"););
   |                                                  ^ no rules expected this token in macro call
```

What's going on here? Isn't semicolon supposed to be an integral part of the statement? If so, it should be consumed by `$s:stmt` without any problem... However, it turns out that no, in [Rust reference](https://doc.rust-lang.org/reference/macros-by-example.html) it's said that `stmt` fragment specifier works like this:

> `stmt`: a [_Statement_](https://doc.rust-lang.org/reference/statements.html) without the trailing semicolon \(except for item statements that require semicolons\)

Hm, ok. I'm not actually sure which item statements requiring semicolons are meant here, but it looks like `println!(...)` or `let _y = x + 1` is not one of them.

### Second nuance: no semicolon in the expansion: \`$s\`

Another thing about semicolons is that `$s` in the macro expansion part does **not** have a trailing semicolon. If I put a semicolon there \(i.e. write `$s;`\) the compiler will complain:

```rust
macro_rules! time_it {
    ($context:literal, $s:stmt) => {
        let timer = std::time::Instant::now();
        $s;  // <- now I'm adding a semicolon here
        println!("{}: {:?}", $context, timer.elapsed());
    };
}

warning: unnecessary trailing semicolon
  --> src/main.rs:19:11
   |
19 |         $s;
   |           ^ help: remove this semicolon
...
31 |     time_it!("println", println!("hello, world!"));
   |     ----------------------------------------------- in this macro invocation
   |
   = note: `#[warn(redundant_semicolon)]` on by default
```

This looks strange again: if `$s` matched our tokens without trailing semicolon, won't it be logical to add that semicolon in the expansion part to compensate?

The reason that it doesn't work like that is as follows. When we expand the macro, matched `$s` is not a token stream anymore! Since its tokens have been fully matched by `stmt` specifier, it's now a fully formed AST statement node! As you probably know, macro expansion in Rust happens not after the lexical stage \(as it does with `#define`in C\), but after the parsing stage, i.e. when we have a parsed abstract syntax tree of the program. That's the reason why we can use all those nice specifiers like `expr` , `stmt` and so on: we have access to the syntactic information. This is [very well explained](https://danielkeep.github.io/tlborm/book/mbe-syn-source-analysis.html) in "The Little Book of Rust Macros".

Therefore, when we plug `$s` it's already a fully formed statement and adding an extra semicolon feels redundant to the compiler, so it complains.

### Handling multiple statements

Ok, now let's say we want to wrap several statements in `time_it`. I've tried several approaches before making it work.

### With block specifier

My first naive attempt was to use `block` specifier \(since I didn't want to deal with all those nasty semicolons\):

```rust
($context:literal, $b:block) => {
    let timer = std::time::Instant::now();
    $b
    println!("{}: {:?}", $context, timer.elapsed());
}

fn main() {
    time_it!("two println", {
        println!("- hello, world!");
        println!("- hi, programmer!");
    });
}

- hello, world!
- hi, programmer!
two println: 35.549µs
```

Again, it looks good at first glance. But there is a problem with scoping. If you had `let` assignments in the original code and wrapped them in the block just to pass it to `time_it` - it's not possible to use them again, since the scope has changed:

```rust
// Before:
fn main() {
    let x = 1;
    let y = 2;
    let z = x + y;
    
    println!("z = {}", z);
}

// After:
fn main() {
    time_it!("let", {
        let x = 1;
        let y = 2;
        let z = x + y;
    });

    println!("z = {}", z);
}

// Error:
error[E0425]: cannot find value `z` in this scope
  --> src/main.rs:42:24
   |
42 |     println!("z = {}", z);
   |                        ^ not found in this scope

```

Not good. Let's try to use macro repetitions feature to actually say that we want multiple statements.

### With \`stmt\` repetitions

```rust
macro_rules! time_it {
    ($context:literal, $($s:stmt);+) => {
        let timer = std::time::Instant::now();
        $(
            $s
        )*
        println!("{}: {:?}", $context, timer.elapsed());
    }
}

fn main() {
    time_it!("let",
        let x = 1;
        let y = 2;
        let z = x + y
    );

    println!("z = {}", z);
}

// Output:
// let: 206ns
// z = 3

```

Now I know that `stmt` are parsed without trailing semicolon, so I'm just using this semicolon as a separator in a repetition: `$($s:stmt);+`. Here I say that I want one or more statements separated by semicolons. Also, I don't use any semicolons after `$s` in the expansion part due to the same reasons as before. 

It works almost perfectly, but note that the last statement `let z = x + y` still can't have a trailing semicolon when wrapped. That's expected, because it's the last one and repetitions syntax only allows separators **between** the items, not after the last one. One way to fix this is just to add that last semicolon explicitly: `$($s:stmt);+ ;` It works fine, because semicolons are allowed after `stmt` specifier \(not all symbols are, only `;` `,` and `=>` \), see ["Follow-set Ambiguity Restrictions"](https://doc.rust-lang.org/reference/macros-by-example.html#follow-set-ambiguity-restrictions) in Rust reference for details.

### With \`tt\` specifier

Another way of achieving this would be to use a sledgehammer of Rust declarative macros: `tt` specifier. This just matches any token tree. What is a token tree? It's either a single token or a group of any number of token trees wrapped in `()`, `[]` or `{}`. So we can just say: take many arbitrary token trees, prepend `Instant` creation to that, paste all the tokens and then print elapsed time:

```rust
($context:literal, $($tt:tt)+) => {
    let timer = std::time::Instant::now();
    $(
        $tt
    )+
    println!("{}: {:?}", $context, timer.elapsed());
}
```

This also works and doesn't require any special treatment of semicolons, but is probably overly generic, allowing more token types than we actually want to allow.

### Summary

I hope this exposed and clarified some nuances of working with Rust declarative macros. There are lots of other interesting nuances when one digs deeper. An interesting and illuminating reading about that is ["The Little Book of Rust Macros"](https://danielkeep.github.io/tlborm/book/README.html) which I recommended above. It's a little bit dated but still a very good read. And of course, Rust reference itself has an[ informative section](https://doc.rust-lang.org/reference/macros-by-example.html) on macros, which is an invaluable source of information for us, the ones trying to dig to the source of things.



