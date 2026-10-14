+++
date = '2026-10-13T13:39:37+01:00'
draft = true
title = 'Session 4 (Workshop 2): Adding in Rust'
tags = ['Session', 'Operating Systems', 'Workshop']
summary = "Second workshop of Sem 1: Introducing Rust, VGA buffers and Frames"
+++

# Intro

**Session 4** ran on Thursday, 15th October in AT7.14. The session was a code-writing workshop that introduced Rust, explored the VGA Buffer further and covered frame allocation + page tables as part of the 'Write-Your-Own-OS' series of workshops that will be done throughout semester 1.
This session was ran by **Kacper** and **Archie**.

## Credit & Disclaimer:
This guide is intended for self-study / for people to follow along themselves. The sessions will cover the ideas in broad strokes and clarify parts where it gets confusing, but overall we aim to make each one of the workshop posts something that somebody could, ideally, do from home. Any slides that we present in workshop sessions will be available in the 'session content' section of this post.

We use a lot of material from Philipp Oppermann's blog *'Writing an OS in Rust (First Edition)'*; in fact, our workshop posts will effectively use his materials as a baseline that we will build on with theory, further ideas and may diverge in some cases where we thought it useful. Please check out his blog [here](https://os.phil-opp.com/edition-1/); our work would be much harder without his content!

**Furthermore, the majority of the content of these workshops will be written assuming a Linux host machine in mind. We can assist with some differences in distros, but we'd ask that if you follow this guide with a macOS or Windows system, you should use a Linux VM (like WSL).** For issues relating to compatibility and system-specific fixes, either ask us in-session or on the Discord to see if we know a fix!

## Session Content:
Empty at the moment - check back later!

# Rust set up & boilerplate

Continuing on with Phil Oppermann's amazing blog, we'll now be installing Rust and setting up all sorts of boilerplate code to make things work. We'll try to explain things along the way, but if there's something not working or confusing, don't hesitate to let us know in-session or on the discord.

## Installing Rust

Since we'll be using a lot of unstable features, we'll need to use **nightly** for its compiler. To do this, we'll managae our Rust installations with the use of [rustup](https://rustup.rs/). To tell rustup that we want to use nightly for our current directory, we run `rustup override nightly` in the terminal.

## Creating a Cargo project

[Cargo](https://doc.crates.io/guide.html) is a Rust package manager used for just about any Rust project out there. Since our project folder already exists, we'll set it up as a Cargo project manually by adding a cargo config file named `Cargo.toml`:
```toml
[package]
name = "your_os_name"
version = "0.1.0"
authors = ["your_name_here"]

[lib]
crate-type = ["staticlib"]
```

- `package` contains required project metadata
- `lib` specifies that we want to build a static library (through `crate-type`), i.e. a library that contains all of its dependencies.

If you want additional details on `Cargo.toml`, we recommend checking out the Cargo reference in the rust-lang docs [here](https://doc.rust-lang.org/cargo/reference/index.html).

### Rust 101 - an aside
TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO

### Source File

Now, we want to place our root source file in `src/lib.rs`:
```rust
#![feature(lang_items)]
#![no_std]

#[no_mangle]
pub extern fn rust_main() {}

#[lang = "eh_personality"] #[no_mangle] pub extern fn eh_personality() {}
#[lang = "panic_fmt"] #[no_mangle] pub extern fn panic_fmt() -> ! {loop{}}
```

- `#!` defines an *attribute* of the current module. Since we're at the root module, the attributes apply to the [crate](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) itself (a crate being the smallest amount of code that the Rust compiler considers at a time).
- The `feature` attribute allows the specified *featured-gated* attributes in this crate. This can't be done in a stable/beta compiler, hence why we use Rust nightly.
- `no_std` attribute prevents the automatic linking of the standard library. This is because `std` relies on operating system features like files, syscalls, device drivers, IO and more which at the moment isn't handled by our OS!
- A `#` without a `!` defines an attribute for the *following* item (in this case, a function).
- `no_mangle` attribute disables the automatic *name mangling* that Rust uses to get unique function names. We want to call to `rust_main` from our assembly code, and if name mangling was enabled, `rust_main` wouldn't be the actual name of our function.
- We mark `rust_main()` as `extern` to make it compatible to the standard C [calling convention](https://doc.rust-lang.org/std/keyword.extern.html).
- The `lang` attribute defines a Rust [language item](https://doc.rust-lang.org/1.10.0/book/lang-items.html) (basically a way to make the `rustc` compiler know that certain operations exist)
- `eh_personality` function is used for us to tell Rust how to handle stack [unwinding](https://doc.rust-lang.org/nomicon/unwinding.html) on a `panic!`. At the moment, we leave this empty.
- `panic_fmt` function is the entry point on panic (i.e. what we do when a panic occurs). Again, we can't do much useful stuff here yet, so we just make sure that it doesn't return anything by looping (and use the `!` return type to specify a no-return type).

## Building Rust

We can now build our project by running `cargo build` which creates a static library at `target/debug/lib[your_os_name].a`. However, the resulting library will have been built specific to our **host operating system** (the machine that we're writing this on). If you want to run it solely on your own machine, this isn't much of an issue, but in the interests of standardisation, **let's define properties of our target system**:

- **x86_64**: our target CPU is an `x86_64` CPU.
- **No operating system**: Our target doesn't run any operating system (as that's what we're writing), so the compiler shouldn't assume any OS-specific functionality
- **Handles hardware interrupts**: Our kernel will need to handle asynchronous hardware interrupts at some point. This means that we'll have to disable a certain stack pointer optimization ([*the red zone*](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64#the-red-zone)), as it'd cause stack corruptions otherwise. 
- **No SSE**: Our target might not have **SSE** support (SSE = Streaming SIMD Extensions). Even if it does, we don't want to use SSE instructions in our kernel as it makes interrupt handling muchb slower. More on this later..
- **No hardware floats**: The `x86_64` architecture uses SSE instructions for floating point operations, which as said above, we're not gonna use. So we'll need to avoid hardware floating point operations in our kernel; instead, we'll use *soft floats*, which are basically software functions that emulate floating point operations using normal integers.

### What's the 'red zone'?
The red zone is an optimization of the [System V ABI](https://wiki.osdev.org/System_V_ABI) that allows functions to temporarily use **128 bytes below its stack frame** without adjusting the stack pointer.

![The red zone](https://os.phil-opp.com/set-up-rust/red-zone.svg)

The idea is that the function can use this 'red zone' area for temporary data that's not needed across function calls. This provides a nice optimization where we don't have to adjust the stack pointer in somde cases (e.g. leaf functions). Think of it as a 'messy-work' area that we don't necessarily care much for, but are about to see why its problematic.

Suppose an **exception** occurs while the red zone is in use by the function. The CPU and the exception handler will grow the stack downwards (moving the new stack pointer's location down) to accommodate the Exception Stack Frame, Register Backup & Handler Function Stack Frame. But hold on; we're using the red zone and then the exception has suddenly taken up space in the red zone, thus overwriting the data there, which will still be needed by the interrupted function (whenever the interrupt ceases). So the function will no longer work correctly upon the exception handler returning control and generally causes a big headache for anyone trying to debug this black magic ([especially for this guy in 2010](https://forum.osdev.org/viewtopic.php?t=21720)).

To avoid all this hassle and headache, we'll just disable the red zone so that we don't have to worry about it when implementing exception handling.

### Target Specifications

With the properties of our target system in mind, we can use a JSON configuration file to specify this custom target. A minimal target specification equal to `x86_64-unknown-linux-gnu` (default 64-bit Linux target) looks like this (don't use this):
```json
{
  "llvm-target": "x86_64-unknown-linux-gnu",
  "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128",
  "linker-flavor": "gcc",
  "target-endian": "little",
  "target-pointer-width": "64",
  "target-c-int-width": "32",
  "arch": "x86_64",
  "os": "linux"
}
```

- `llvm-target` specifies the target triple that is passed to LLVM. Target triples are a naming convention that define the CPU architecture, the vendor, the operating system and the ABI. For example, `arm_unknown_linux_gnu` is an ARM CPU with an unknown vendor, linux OS and a GNU ABI.
- `data-layout` is also passed to LLVM and specifies how data should be laid out in memory. It consists of various specifications separated by a `-` character. For example, the `e` means little endian and `S128` specifies that the stack should be 128 bits (= 16 byte) aligned. The format is described in detail in the LLVM documentation but there shouldn’t be a reason to change this string.
- `linker-flavor` allows us to control the interface and behaviour of the linker that the compiler will invoke when building our code. Different linkers use different syntax & flags, so they get grouped into 'flavors' by Rust so that we know what commands and arguments we need to pass accordingly.
- The other fields are specifically for conditional compilation, which we won't worry about just now.

### Our Kernel Target Specification

For our target system, we'll define the following JSON config in a file named `x86_64-[your_os_name].json`:
```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128",
  "linker-flavor": "gcc",
  "target-endian": "little",
  "target-pointer-width": "64",
  "target-c-int-width": "32",
  "arch": "x86_64",
  "os": "none",
  "disable-redzone": true,
  "features": "-mmx,-sse,+soft-float"
}
```

- Notice we don't include an ABI; it doesn't matter for us, so we just leave it off.
- Basically the same as the previous spec, but changes to OS (none) and additionally we have `disable_redzone` and `features` which are pretty self-explanatory (`-sse` in features means we don't want to include SSE features, `+soft-float` means we want to work with soft floats, etc.)



# VGA Text Buffer with Rust Macros

# Frame Allocation + Page Tables

# End of Post

If you have any questions, join our [discord](https://discord.gg/zKhj937xW2) and ask away!

`42 49 54 53 49 47 20 3C 33 20 59 4F 55`
