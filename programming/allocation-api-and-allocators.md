---
description: Exploring allocators in general and their Rust API in particular.
---

# Allocation API and allocators

## Introduction

I like to look at standard libraries of programming languages! It feels like being in a candy store: there are so many interesting functions and nice tools one can use! Some of them may feel useless at a particular moment, but may come handy later to save on some typing, to avoid reinventing the wheel, or even to guide a design choice.

In Rust, [standard library documentation](https://doc.rust-lang.org/std/) is extremely readable and even has a section "How to read this documentation" for people like me. Let me quote from it:

> If this is your first time, the documentation for the standard library is written to be casually perused. Clicking on interesting things should generally lead you to interesting places.

So, I started reading and here we go. The very first module in the documentation is:

| [alloc](https://doc.rust-lang.org/std/alloc/index.html) | Memory allocation APIs |
| :--- | :--- |


This definitely sounded interesting, so I decided to take a closer look and this post summarizes what I've learnt about allocators so far.

## Registering dummy allocator as global

`std::alloc` module provides us with means to write our own memory allocators and register them as the default global allocator for our programs. Moreover, this is easy to do: you just have to implement a trait with two functions: `alloc` and `dealloc`and register it with `#[global_allocator]` annotation, that's it. Of course, writing the allocator and making it do something reasonable is much less easier.

After that, every time you allocate anything \(say, use a `Box` or `Vec` or anything else which allocates on heap\) you'll be using your own allocator, how cool is that.

Let's try to break the system and write a `NullAllocator` \(which just pretends that there was no memory and immediately gives up\), register it gobally and see what happens.

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

`Layout` specifies size and alignment of the memory you want to get. I.e. you can say: "I want 16 bytes with alignment of 2". This means that you'll get 16 bytes at an address divisible by 2. Or you can use "turbofish" and say `Layout::new::<u32>()` which means just give me memory which is good enough to store `u32`. 

Note that we also get `layout` as a parameter for deallocation, not only a pointer \(as is the case with `free(ptr)` function in C\).

Of course if you are implementing your own allocator, there isn't much Rust can do to protect you in terms of memory safety, so not only the trait functions are `unsafe`, but even the trait impl itself is `unsafe`. Unsafe impl means that you have to provide certain guarantees on the whole allocator logic, while `alloc` function is unsafe because it will result in Undefined Behaviour if you request zero bytes.

We returned null pointer from our `alloc`, which is a way to tell that we are out of memory. Let's try to run our program and see what happens.

```text
> cargo run
Finished dev [unoptimized + debuginfo] target(s) in 0.00s
Running `target/debug/alloc-blog`
memory allocation of 4 bytes failed[1] 8113 abort      cargo run
```

Wow, and we didn't even allocate anything, note that my `main` function was **empty**: `fn main() { }`

So, what's going on? Who requested those 4 bytes, is it that minimalist Rust runtime? I thought it should only allocate arguments to `main` on stack and the stack itself is provided by OS... Let's investigate.

## Allocation before \`main\`

For this we can write one more allocator which wraps the system allocator and additionally counts number of bytes it allocated! I've shamelessly stolen this example from `std::alloc::System` documentation:

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

Here we use atomics, recently there was [a post](https://www.nickwilcox.com/blog/arm_vs_x86_memory_model/) in "This Week in Rust" about atomics, memory model and different implementation on ARM and x86 processors, you may want to check it out. But for now, we just need to know that atomics represent shared memory for threads to communicate. And `Ordering::SeqCst` represent the highest barrier between those threads, so that they definitely see the previous value before updating it and everything  works well for our example.

Also note that we use `std::alloc::System` which is a system allocator. What's that, say, on macOS? In the libstd [source code](https://github.com/rust-lang/rust/blob/f844ea1e561475e6023282ef167e76bc973773ef/src/libstd/sys/unix/alloc.rs#L14) we can see that it basically amounts to calling `malloc` library function from `libc` on UNIX systems \(including macOS\).

So what's the answer? How many bytes, do you think have been allocated before `main`? Click on this black-box to reveal the answer: ⬛. Hm, I need some JavaScript to make it happen and it doesn't work on Gitbook, so ok, I'll tell you without clicking: 237 bytes on my macOS machine. And 241 if we don't subtract for deallocating \(so apparently 4 bytes have even been deallocated\). Just for fun I've also asked my friend to test it on Windows and Linux:

| OS | RT Allocated, bytes | Remaining in use, bytes |
| :--- | :--- | :--- |
| macOS | 241 | 237 |
| Windows | 441 | 173 |
| Linux | 177 | 173 |

Again, there recently was an [interesting post](https://blog.mgattozzi.dev/rusts-runtime/) by Michael Gattozzi in "This Week in Rust" about all the details of Rust runtime. It guides you through the code and it can be seen that at least the runtime uses `Vec` to allocate `std::env::args` from `argv`pointer. Also have `"main".to_owned()` to name the main thread there. And `String` allocates just the same as `Vec`. Presumably there are other structures which allocate and all of this is of course OS-dependent.

I've also tried to just search for that error message in Rust codebase:

```text
> rg "memory allocation of"
src/libstd/alloc.rs
270:    dumb_print(format_args!("memory allocation of {} bytes failed", layout.size()));
```

The error messages comes from `default_alloc_error_hook` function which gets called when there is an allocation error. Apparently we can also override the error hook and install our own.

## Tracing in allocators

Now, we played enough with the dummy and system allocators and want to write our own. However, it would be nice to have some tracing ability to be able to monitor the allocation.

Just naively using `println!("allocated {} bytes", layout.size())` in `alloc` function doesn't work and leads to this scary error: `illegal hardware instruction`. Oh well, joys of unsafe code.

As you may have guessed this happens because `println!` allocates. And allocating within the allocator itself presumably leads to some kind of stack overflow masking as that scary error above.

But where exactly it allocates? First, I thought that it happens while building the formatted string, but no. Let me show you the tracing code which works fine inside `alloc`:

```rust
// This works fine like this: 
// print(format_args!("hello {}", 1));
fn print(args: std::fmt::Arguments<'_>) {
    std::io::stderr().write_fmt(args).unwrap()
}

// so I can use it inside allocator's `alloc`:
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ret = System.alloc(layout);
        if !ret.is_null() {
            ALLOCATED.fetch_add(layout.size(), SeqCst);
            print(format_args!(
                "allocated {} bytes with align {}\n",
                layout.size(),
                layout.align()
            ));
        }
        return ret;
    }
// ...
```

This code uses `format_args!` and still it doesn't allocate. Both `print!` and `eprint!` also use `format_args!` 

Apparently, both `print!` and `eprint!` \(and their -ln variants\) allocate when they create their local versions of stderr/stdout: judging from the [source code](https://github.com/rust-lang/rust/blob/f844ea1e561475e6023282ef167e76bc973773ef/src/libstd/io/stdio.rs#L16) there is a `Box` there and it alone allocates eight bytes. If we just use `std::io::stderr()` directly, this doesn't happen. 

Note also that using `stderr` \(as opposed to `stdout`\) is also important, because using `std::io::stdout()` also allocates!

I've used this code to actually measure the number of bytes allocated by different constructs on macOS:

```rust
fn main() {
    let before_main = ALLOCATED.load(SeqCst);
    print(format_args!(
        "allocated bytes before main: {}\n",
        before_main
    ));
    
    std::io::stderr().write(b"hello").unwrap();
    
    print(format_args!(
        "allocated bytes for the construct: {}\n",
        ALLOCATED.load(SeqCst) - before_main
    ));
}
```

| Code | Bytes allocated |
| :--- | :--- |
| `println!()` | 1240 |
| `eprintln!()` | 32 |
| `std::io::stdout().write(b"")` | 1208 |
| `std::io::stderr().write(b"hello")` | 0 |
| `std::io::stderr().write_fmt(format_args!("{}", 1)` | 0 |

I think that the difference between `stderr` and `stdout` in terms of allocations can be explained by [this comment](https://github.com/rust-lang/rust/blob/f844ea1e561475e6023282ef167e76bc973773ef/src/libstd/io/stdio.rs#L762) from libstd:

> Note that unlike `stdout()` we don't use `Lazy` here which registers a destructor. Stderr is not buffered nor does the `stderr_raw` type consume any owned resources, so there's no need to run any destructors at some point in the future.
>
> This has the added benefit of allowing `stderr` to be usable during process shutdown as well!

Now we also know that one more benefit is that it can be used in `alloc`along with `format_args!`

## Using custom allocators for standard collections

Of course it's entirely optional to register allocator with `#[global_allocator]`, you can continue using the standard allocator and only use custom allocators for specific goals by directly calling their `alloc` and `dealloc`methods. 

This raises an interesting question: if you want to allocate a Box, Vec or HashMap using a custom allocator, how would you do this? I was surprised to discover that there is no such API in `Vec`, even on nightly. However, the work and discussion on this are ongoing, I think the latest relevant places to check is the [repository](https://github.com/rust-lang/wg-allocators) of "Rust Allocators Working Group". It looks like the proposed version of API adds `new_in` [function](https://timdiekmann.github.io/alloc-wg/alloc_wg/vec/struct.Vec.html#method.new_in) which will receive an allocator \(i.e. something implementing `AllocRef`, another allocator trait\) as a parameter, so you can have `Vec::new_in(my_allocator)` or `Box::new_in(2, my_allocator)`.

## How to get memory from inside the allocator

If we want to write a more realistic allocator which actually does some allocation and not just returns `null_ptr` or counts whatever system allocator returns us, we need to face another serious question: what's the proper way to actually get memory in our allocator then? If we just call `libc::malloc` doesn't it defeat the point? I.e. don't we just delegate the work to another allocator this way? In order to answer this question, we probably need some overview of memory management first.

## General overview of memory management

As you probably know, memory management conceptually happens on two levels.

1. Operating system manages physical memory of the whole machine giving it to processes and using it for its own purposes. 
2. Programs manage their dynamic memory for their own purposes \(for example to create dynamic data structures\).

We can manage memory in fixed size chunks or in variable size chunks. 

On the first level, operating systems normally use _fixed_ size approach called _paging:_ memory is given out to processes in fixed size pages, usually 4k in size \(you can run `pagesize` on macOS to check the page size\). OS and relevant hardware also has to manage for each process the mapping between virtual _pages_ \(which are fixed size parts of the process virtual address space\) and physical _page frames_ which are parts of actual physical memory. The mapping itself may be rather large, so in order to resolve lookups efficiently OS uses help of the hardware which basically caches part of the pages mapping in Memory Management Unit of the CPU. Without hardware support, paging would be prohibitively slow due to extra indirections involved in the address translation.

On the second level, programs normally request memory in chunks of _variable_ sizes, which makes it harder to manage than using _fixed_ size chunks. Variable size leads to _fragmentation_ and requires certain algorithms to handle it. This also entails additional performance costs associated with such algorithms. Hence, there is a need for an allocator library which provides API for allocating and deallocating memory of any size, so that the complexity can be abstracted away and hidden from programmers.

 If it's abstacted away, then we need some API to work with it.`malloc` and `free` from C is an example of such API, `allocate` and `deallocate` from `std::alloc` in Rust we are looking at is another. 

Now it should make some sense that  `malloc` is _not_ an OS system call, but a library function. And allocators implementing `malloc` are part of C standard library \(i.e. [GNU libc](https://www.gnu.org/software/libc/manual/html_node/The-GNU-Allocator.html) or [musl libc](https://git.musl-libc.org/cgit/musl/tree/src/malloc/mallocng/malloc.c)\). When you link to that library, you get a clever allocator for free.

## \`malloc\` and getting new memory

How does `malloc` gets new memory under its management? It has two options: to adjust the program break with `brk` or to map new pages of memory with `mmap`. Let's look at those in more details.

![Process memory layout](../.gitbook/assets/memory-layout.jpg)

Above you can see a typical process memory layout, which describes which areas of the process virtual address space are assigned to which segments. The _program break_ is basically the address of the end of data segments of the program. Past program break there are unallocated addresses and if we try to access them we will get "segmentation violation" error. When the process is created, its program break is set to the top of uninitialised data segment \(BSS\), so the heap size is effectively 0.

### brk/sbrk

`malloc` then can increase the available heap area by moving program break further up. This is done by `brk` or `sbrk` system calls on UNIX. [Those calls](https://linux.die.net/man/2/sbrk) just update the address, they don't do any actual memory mapping. Their effect in essence is that they make new virtual memory addresses available for the process to use \(so we won't get segmentation violation anymore\). The actual pages mapping is made by OS kernel when we try to access that memory for the first time. Since OS virtual memory operates on pages, actual changes in program break are multiples of the system page size.

Allocators can also decrease the amount of mapped memory by moving the program break down \(say, when all the blocks at the top were freed\).

By the way, I was always puzzled why those calls had such strange names for something heap-related. Now I know that that's because they manipulate that "program break", which is some ancient programming term.

### mmap

Another option for `malloc` is to use `mmap` which is a system call which can be used for many purposes: to implement memory-mapped file I/O, to allocate memory or to organize shared memory for inter-process communication. If we use `mmap` like this:

```c
mmap(
  NULL, // let OS decide where to place it in virtual address space 
  size, // size of the memory we need, in bytes
  PROT_READ | PROT_WRITE, // read/write memory 
  MAP_ANON | MAP_PRIVATE, // zero-filled, used only by this process 
  -1, // unused for anonymous memory
  0   // unused for anonymous memory
);
```

We are basically asking OS to give us a zero-initialised chunk of memory with size `size` so that we can read/write it for any private needs in our process. This memory will be mapped to an virtual address of the process at OS discretion.

`mmap` is normally used by `malloc` for allocating larger blocks of memory, it's controlled by certain threshold which can be adjusted.

I think the principal differences between `mmap` and `brk` are as follows:

*  with `mmap` we can have many memory blocks \(for example one for each thread\), while with `brk` and program break - we maintain a large monolitic block of memory - the heap.
* with `mmap` we can give back the memory of a particular block back to the OS \(with `munmap`\) when everything in that block has been freed. It's trickier to return memory with `sbrk` approach, since we can't return freed memory from the middle of the monolitic heap \(we can only decrease the program break when end of the monolitic block is freed\).



## Back to the "getting memory in our allocator" question

So, now we've seen how `malloc` does it, we have some ideas how to do it in our own allocator program.

The easiest approach is probably just to use the system allocator `std::alloc::System` and ask it for some large memory area which we can then micro-manage with our allocator \(which presumably has some specific goals\).

If we want to get lower than that we can emulate what `malloc` is doing and use OS-based system calls \(most likely `mmap` on UNIX-based systems, [VirtualAlloc](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc) on Windows or something else on WebAssembly\). For example you can see how `wee_alloc` [does it ](https://github.com/rustwasm/wee_alloc/blob/f26c431df6fb6c7df0d6f8e0675471b9c56d8787/wee_alloc/src/imp_unix.rs#L11)on UNIX with `mmap`.

Finally, if we don't have any OS at all and are running on some bare-metal embedded device, we can just use a statically allocated larged array to manage:

```rust
static mut HEAP: [u8; 8192] = [0; 8192];
```

## Summary

This post is already getting rather long, so I think I need to wrap up, even though we've just scratched the surface of the allocators and didn't even start to implement one. 

However, now we are equipped with important tools and ideas: we know the basics about virtual memory and paging, how to get memory from the OS, how to register our allocator within Rust and how to add tracing which should help with the debugging.

There seem to be a ton of interesting algorithms and decisions to be made when implementing an allocator. Also, the staggering amount of different approaches suggests that there is no silver bullet and it's impossible to write an allocator which will scale well and satisfy all the possible needs, so we need to specialise.

If you want to read more about virtual memory and operating systems in general I highly recommend a [free book](http://pages.cs.wisc.edu/~remzi/OSTEP/) "Operating Systems: Three Easy Pieces". It's an amazing textbook which is quite entertaining to read just for fun. Some of the relevant chapters are ["Address Translation"](http://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf), ["Free-Space Management"](http://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf) and ["Paging: Introduction"](http://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf).

If you want to see more Rust code examples and some actual implementations of educational allocators you can read those two pages from the wonderful blog "Writing OS in Rust" by Philipp Oppermann: ["Heap Allocation"](https://os.phil-opp.com/heap-allocation/) and ["Allocator Designs"](https://os.phil-opp.com/allocator-designs/), it's very very good.

If you want to take a look at some popular real-world allocators written in Rust, you can start with the following:

* [wee\_alloc](https://github.com/rustwasm/wee_alloc) - "The **W**asm-**E**nabled, **E**lfin Allocator". 333 ⭐. This allocator specialises for WASM, trying to be simple \(so that the generated WASM size will be smaller\). It can also be run on UNIX, Windows or just with static array as a heap.
* [bumpalo](https://github.com/fitzgen/bumpalo) - "A fast bump allocation arena for Rust". 339 ⭐. Bump allocation is a good place to start, since this is one of the simplest allocation strategies you can implement, while still being useful. Also the code is very well commented.

That's it, I hope you've learnt something useful. If you want to comment: you can just create an issue on Gihub. Or if you want to send me some pull request with any changes or errors you've found, I'd be extremely grateful.

