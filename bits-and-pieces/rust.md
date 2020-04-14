---
description: Bits and pieces of Rust
---

# Rust

### Useful commands

* To override default toolchain for a particular project: `rustup override set nightly`. The directory is stored in `~/.rustup/settings.toml` \(in `overrides` section\) separately from the project itself.

### Closures

* A [good post](https://stevedonovan.github.io/rustifications/2018/08/18/rust-closures-are-hard.html) by Steven Donovan about closures in Rust. It makes the connection between closures and structs, explains why `move |...|` is sometimes needed and why we have to add lifetimes annotations like `where F: Fn(i32) -> i32 + 'a>`
* All the nitty-gritty details are available in the Rust language reference \(sections ["Closure expressions"](https://doc.rust-lang.org/stable/reference/expressions/closure-expr.html) and ["Closure types"](https://doc.rust-lang.org/stable/reference/types/closure.html)\).

### Common/standard traits, conversion

* ["Rust's Built-in Traits, the When, How & Why"](https://llogiq.github.io/2015/07/30/traits.html) by Lloqiq \(2015\)
* ["The Common Rust Traits"](https://stevedonovan.github.io/rustifications/2018/09/08/common-rust-traits.html) by Steve Donovan \(2018\)
* ["Use conversion traits"](https://deterministic.space/elegant-apis-in-rust.html#use-conversion-traits) part from "Designing elegant APIs in Rust" by Pascal Hertleif \(a very insightful post!\) \(2016\)
* ["Conversion" ](https://doc.rust-lang.org/stable/rust-by-example/conversion.html)section in "Rust by Example"
* ["Convenient and idiomatic conversions in Rust"](https://ricardomartins.cc/2016/08/03/convenient_and_idiomatic_conversions_in_rust) by Ricardo Martins \(2016\)
* Rust reference: ["Special types and traits"](https://doc.rust-lang.org/reference/special-types-and-traits.html)

### Lifetimes, NLL

* [RFC-2094](https://github.com/rust-lang/rfcs/blob/master/text/2094-nll.md) about Non-Lexical Lifetimes: why and when they are needed, design
* ["Lifetimes"](https://doc.rust-lang.org/nomicon/lifetimes.html) and the following sections in Rustonomicon
* ["Understanding Rust Lifetimes"](https://medium.com/nearprotocol/understanding-rust-lifetimes-e813bcd405fa) by Maksym Zavershynskyi

