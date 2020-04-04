---
description: Bits and pieces of Rust
---

# Rust

### Useful commands

* To override default toolchain for a particular project: `rustup override set nightly`. The directory is stored in `~/.rustup/settings.toml` \(in `overrides` section\) separately from the project itself.

### Closures

* A [good post](https://stevedonovan.github.io/rustifications/2018/08/18/rust-closures-are-hard.html) by Steven Donovan about closures in Rust. It makes the connection between closures and structs, explains why `move |...|` is sometimes needed and why we have to add lifetimes annotations like `where F: Fn(i32) -> i32 + 'a>`
* All the nitty-gritty details are available in the Rust language reference \(sections ["Closure expressions"](https://doc.rust-lang.org/stable/reference/expressions/closure-expr.html) and ["Closure types"](https://doc.rust-lang.org/stable/reference/types/closure.html)\).

