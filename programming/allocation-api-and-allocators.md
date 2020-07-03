---
description: Exploring allocators in general and their Rust API in particular.
---

# Allocation API and allocators

## Intro

I like to look at standard libraries of programming languages! It feels like being in a candy store: there are some many interesting functions and nice tools one can use! Some of them may feel useless at a particular moment, but may come handy later to save on some typing, to avoid reinventing the wheel, or even to guide a design choice.

In Rust, [standard library documentation](https://doc.rust-lang.org/std/) is extremely readable and even has a section "How to read this documentation" for people like me. To quote from it:

> If this is your first time, the documentation for the standard library is written to be casually perused. Clicking on interesting things should generally lead you to interesting places.

So, I started reading and here we go. The very first module in the documentation is:

| [alloc](https://doc.rust-lang.org/std/alloc/index.html) | Memory allocation APIs |
| :--- | :--- |


This definitely sounded interesting, so I decided to take a closer look and this post summarizes what I've learnt about allocators.

## Registering dummy allocator as global

`std::alloc` module provides us with means to write our own memory allocator and register it as a default global allocator for our program! And moreover this is easy to do: you just have to implement a trait with two functions: `alloc` and `dealloc`and register it with `[global_allocator]` annotation, that's it. Of course, writing the allocator and making it do something reasonable is much less easier.

After that, every time you allocate anything \(say, use a `Box` or `Vec` or anything else which allocates on heap\) you'll be using your own allocator, how cool is that.

Let's try to break the system and write a `NullAllocator` which immediately gives up as there was no memory, register it gobally and see what happens.

```rust
use std::alloc::{GlobalAlloc, Layout};

struct NullAllocator;

unsafe impl GlobalAlloc for NullAllocator {
    unsafe fn alloc(&self, _lt: Layout) -> *mut u8 {
        std::ptr::null_mut()
    }
    unsafe fn dealloc(&self, _ptr: *mut u8, _lt: Layout) {
        panic!("won't deallocate: we never allocated!");
    }
}

#[global_allocator]
static A: NullAllocator = NullAllocator;

fn main() {}
```

Several comments along the way.

`Layout` specifies size and alignment of the memory you want to access. I.e. you can say I want 16 bytes with alignment of 2. This means that you'll get 16 bytes at an address divisible by 2. Or you can use turbofish and say `Layout::new::<u32>()` which means just give me memory which is good enough to store `u32`. There are certain rules specified about data layout of Rust types in the [reference](https://doc.rust-lang.org/reference/type-layout.html).

Also note that we also get `layout` as a parameter while deallocating, not only a pointer \(as is the case with `free(ptr)` function in C\).

Of course if you are implementing your own allocator, there isn't much Rust can do to protect you, so not only the trait functions are `unsafe`, but even the trait impl itself is `unsafe`. Unsafe impl signifies that you have to provide certain guarantees on the whole allocator logic, while `alloc` function is unsafe because it will result in Undefined Behaviour if you request zero bytes.

We returned null pointer from our `alloc`, this signifies that we are out of memory. Let's try to run our program and see what happens.

```text
> cargo run
Finished dev [unoptimized + debuginfo] target(s) in 0.00s
Running `target/debug/alloc-blog`
memory allocation of 4 bytes failed[1]    8113 abort      cargo run
```

Wow, and we didn't even allocate anything, note that my `main` function was empty.

So, what's going on? Who requested those 4 bytes, is it that minimalist Rust runtime? I thought it should only allocate arguments to `main` on stack and the stack itself is provided by OS... Let's investigate.

## Allocation before \`main\`

For this we can write an allocator which wrap system allocator, but counts number of bytes which it allocated! I've shamelessly stolen this example from `std::alloc::System` documentation:

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::SeqCst};

static ALLOCATED: AtomicUsize = AtomicUsize::new(0);

struct Counter;
unsafe impl GlobalAlloc for Counter {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ret = System.alloc(layout);
        if !ret.is_null() {
            ALLOCATED.fetch_add(layout.size(), SeqCst);
        }
        return ret;
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout);
        ALLOCATED.fetch_sub(layout.size(), SeqCst);
    }
}

#[global_allocator]
static A: Counter = Counter;

fn main() {
    println!("allocated bytes before main: {}", ALLOCATED.load(SeqCst));
}

```

Here we use atomics, recently there was [a post](https://www.nickwilcox.com/blog/arm_vs_x86_memory_model/) on "This Week in Rust" about atomics, memory model and different implementation on ARM and x86 processors, you may want to check it out. But for now, we just need to know that atomics represent shared memory for threads to communicate. And `Ordering::SeqCst` represent the highest barrier between those threads, so that they definitely see the previous value before updating it and everything definitely works well for our counting example.

Also note that we use `std::alloc::System` which is a system allocator. What's that, say on macOS? In the libstd [source code](https://github.com/rust-lang/rust/blob/f844ea1e561475e6023282ef167e76bc973773ef/src/libstd/sys/unix/alloc.rs#L14) we can see that it basically amounts to calling `malloc` library function from `libc` on UNIX systems \(including macOS\).

So what's the answer? How many bytes, do you think have been allocated before `main`? Tap on this black-box to reveal the answer: ⬛ - hm, I need some JavaScript to make it happen and it doesn't work on Gitbook, so ok, I'll tell you without clicking: 237 bytes on my macOS machine. And 241 if we don't subtract for deallocating \(so apparently 4 bytes have even been deallocated\). Just for fun I've also asked my friend to test it on Windows and Linux:

| OS | Allocated, bytes | Remaining in use, bytes |
| :--- | :--- | :--- |
| macOS | 241 | 237 |
| Windows | 441 | 173 |
| Linux | 177 | 173 |

Again, there recently was an [interesting post](https://blog.mgattozzi.dev/rusts-runtime/) by Michael Gattozzi in "This Week in Rust" about all the details of Rust runtime. It guides you through the code and it can be seen that at least the runtime uses `Vec` to allocate `std::env::args` from `argv`pointer. Also have `"main".to_owned()` to name the main thread there. And `String` allocates just the same as `Vec`. Presumably there are other structures which allocate and all this is of course OS dependent.

I've also tried to just search for that error message in Rust codebase:

```text
> rg "memory allocation of"
src/libstd/alloc.rs
270:    dumb_print(format_args!("memory allocation of {} bytes failed", layout.size()));
```

This is defined in `default_alloc_error_hook` function which gets called when there is allocation error. Apparently we can also override error hooks to install our own.

