---
description: Bits and pieces of Rust
---

# Rust

* To override default toolchain for a particular project: `rustup override set nightly`. The directory is stored in `~/.rustup/settings.toml` \(in `overrides` section\) separately from the project itself.
* A [good article](https://stevedonovan.github.io/rustifications/2018/08/18/rust-closures-are-hard.html) about closures in Rust. It makes clear the connection between closures and structs, why `move |...|` is sometimes needed and why we have to add lifetimes annotations like `where F: Fn(i32) -> i32 + 'a>`

