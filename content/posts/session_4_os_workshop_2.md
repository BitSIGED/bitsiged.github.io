+++
date = '2026-10-13T13:39:37+01:00'
draft = false
title = 'Session 4 (Workshop 2): Adding in Rust'
tags = ['Session', 'Operating Systems', 'Workshop']
summary = "Second workshop of Sem 1: Introducing Rust and making VGA Buffer printing easier"
+++

# Intro

**Session 4** ran on Thursday, 15th October in AT7.14. The session was a code-writing workshop that introduced Rust, figured out how to integrate it and showed how we can use it for VGA Buffer printing as part of the 'Write-Your-Own-OS' series of workshops that will be done throughout semester 1.
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

Since we'll be using a lot of unstable features, we'll need to use **nightly** for its compiler. To do this, we'll managae our Rust installations with the use of [rustup](https://rustup.rs/). To tell rustup that we want to use nightly for our current directory, we run `rustup override set nightly` in the terminal.

## Creating a Cargo project

[Cargo](https://doc.crates.io/guide.html) is a Rust package manager used for just about any Rust project out there. Since our project folder already exists, we'll set it up as a Cargo project manually by adding a cargo config file named `Cargo.toml`:
```toml
[package]
name = "bitos"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["staticlib"]
```

- `package` contains required project metadata
- `lib` specifies that we want to build a static library (through `crate-type`), i.e. a library that contains all of its dependencies.

If you want additional details on `Cargo.toml`, we recommend checking out the Cargo reference in the rust-lang docs [here](https://doc.rust-lang.org/cargo/reference/index.html).

### Adding .cargo

We'll also need to add another directory at the root of our project, titled `.cargo`. Inside of `.cargo`, create a `config.toml` file with the following contents:
```toml
[unstable]
json-target-spec = true
build-std-features = ["compiler-builtins-mem"]
#build-std = ["core", "compiler_builtins"]


# when we renable the allocator 
build-std = ["core", "compiler_builtins", "alloc"]

[build]
target = "x86_64-bitos.json"

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```



### Rust 101 - an aside
*Note:* This is not a guide that will teach you Rust. We will try to explain as much as possible as we go along, but there are parts where we expect you to use Google if you don't fully understand something; we can't explain every single snippet of code. If you have experience with langs like C, C++, you should find Rust not extremely different, but there are some fundamental differences you should know about:

1. **Memory Safety & Ownership.** Rust enforces memory safety at compile time through an *ownership and borrowing* model. You might've heard about Rust's **borrow checker** before, and Rust tracks who "owns" data and how long it lives (lifetimes). This is different to C, where the programmer sorta has free reign to do `malloc` and `free` as they please, often segfaulting in the process!

2. **Error Handling**. Rust uses the `Result` type for error handling, encouraging explicit error checking. C++ traditionally uses exceptions which, if not handled properly, can cause resource leaks and performance overheads.

3. **Metaprogramming**. Rust uses trait-based generics and macros for metaprogramming, which are less powerful than C's template metaprogramming approach for compile-time computations. Subjectively though, we think Rust's approach is more readable..

Here's a helpful table graphic for those interested:
![Rust vs C++](https://www.techtarget.com/rms/onlineimages/rust_vs_c++-f.png)

The main idea we want you to take away from Rust vs other low level langs is that the most confusing part of these workshops will probably be us trying to appease the borrow checker, so bear with us! If you want to, you can read more about the borrow checker here: [LogRocket on the Rust Borrow Checker](https://blog.logrocket.com/introducing-rust-borrow-checker/). You can also find Rust docs [here](https://doc.rust-lang.org/stable/).

### Source File

Now, we want to place our root source file in `src/lib.rs`:
```rust
#![feature(lang_items)]
#![allow(internal_features)]
#![no_std]

extern crate rlibc;

mod vga_buff;

#[unsafe(no_mangle)]
pub extern "C" fn rust_init() {
    println!("Hello World{}", "!");
}

#[lang = "eh_personality"]
pub extern "C" fn eh_personality() {}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

- `#!` defines an *attribute* of the current module. Since we're at the root module, the attributes apply to the [crate](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) itself (a crate being the smallest amount of code that the Rust compiler considers at a time).
- The `feature` attribute allows the specified *featured-gated* attributes in this crate. This can't be done in a stable/beta compiler, hence why we use Rust nightly.
- `no_std` attribute prevents the automatic linking of the standard library. This is because `std` relies on operating system features like files, syscalls, device drivers, IO and more which at the moment isn't handled by our OS!
- A `#` without a `!` defines an attribute for the *following* item (in this case, a function).
- `no_mangle` attribute disables the automatic *name mangling* that Rust uses to get unique function names. We want to call to `rust_init` from our assembly code, and if name mangling was enabled, `rust_init` wouldn't be the actual name of our function.
- We mark `rust_init()` as `extern` to make it compatible to the standard C [calling convention](https://doc.rust-lang.org/std/keyword.extern.html).
- The `lang` attribute defines a Rust [language item](https://doc.rust-lang.org/1.10.0/book/lang-items.html) (basically a way to make the `rustc` compiler know that certain operations exist)
- `eh_personality` function is used for us to tell Rust how to handle stack [unwinding](https://doc.rust-lang.org/nomicon/unwinding.html) on a `panic!`. At the moment, we leave this empty.
- `panic_handler` function is the entry point on panic (i.e. what we do when a panic occurs). Again, we can't do much useful stuff here yet, so we just make sure that it doesn't return anything by looping (and use the `!` return type to specify a no-return type).

## Building Rust

We can now build our project by running the BitOS build command, which creates a static library for the custom target. A normal `cargo build` would target the host operating system instead.

- **x86_64**: our target CPU is an `x86_64` CPU.
- **No operating system**: Our target doesn't run any operating system (as that's what we're writing), so the compiler shouldn't assume any OS-specific functionality
- **Handles hardware interrupts**: Our kernel will need to handle asynchronous hardware interrupts at some point. This means that we'll have to disable a certain stack pointer optimization ([*the red zone*](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64#the-red-zone)), as it'd cause stack corruptions otherwise. 
- **No SSE**: Our target might not have **SSE** support (SSE = Streaming SIMD Extensions). Even if it does, we don't want to use SSE instructions in our kernel as it makes interrupt handling muchb slower. More on this later..
- **No hardware floats**: The `x86_64` architecture uses SSE instructions for floating point operations, which as said above, we're not gonna use. So we'll need to avoid hardware floating point operations in our kernel; instead, we'll use *soft floats*, which are basically software functions that emulate floating point operations using normal integers.

### What's the 'red zone'?
The red zone is an optimization of the [System V ABI](https://wiki.osdev.org/System_V_ABI) that allows functions to temporarily use **128 bytes below its stack frame** without adjusting the stack pointer.

![The red zone](https://os.phil-opp.com/set-up-rust/red-zone.svg)

The idea is that the function can use this 'red zone' area for temporary data that's not needed across function calls. This provides a nice optimization where we don't have to adjust the stack pointer in somde cases (e.g. leaf functions). Think of it as a 'messy-work' area that we don't necessarily care much for, but are about to see why its problematic.

![Red zone exception](https://os.phil-opp.com/set-up-rust/red-zone-overwrite.svg)

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

For our target system, we'll define the following JSON config in a file named `x86_64-bitos.json`:
```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": 32,
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float",
    "rustc-abi": "softfloat"
}
```

- Notice we don't include an ABI; it doesn't matter for us, so we just leave it off.
- Basically the same as the previous spec, but changes to OS (none) and additionally we have `disable_redzone` and `features` which are pretty self-explanatory (`-sse` in features means we don't want to include SSE features, `+soft-float` means we want to work with soft floats, etc.)

### SIMD Extensions
More formally, `sse` and `mmx` features determine support for SIMD (Single Instruction, Multiple Data) instructions which simultaneously perform an operation (like addition) on multiple data words. The `x86` architecture supports the following standards:
- **MMX**: *Multi Media Extension* instruction set defines eight 64 bit registers called `mm0` through `mm7`. These registers are just *aliases* for the registers of the x87 floating point unit.
- **SSE**: *Streaming SIMD Extensions* instruction set adds a completely new register set (instead of reusing floating point registers). The sixteen new registers are called `xmm0` through `xmm15` and are 128 bits each.
- **AVX**: *Advanced Vector Extensions* are extensions that further increase the size of the multi media registers. The new registers are called `ymm0` through `ymm15` and are 256 bits each. They extend the `xmm` registers, so `xmm0` is just the lower half of `ymm0` as an example.

By using such SIMD standards, programs can often speed up significantly. Good compilers are able to transform normal loops into such SIMD code automatically through a process called [auto-vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization).

However, the large SIMD registers lead to problems in OS kernels. The reason is that the kernel has to backup all registers that it uses on each hardware interrupt (recall what we were talking about with PCBs). So if the kernel uses SIMD registers, it has to backup a lot more data, which noticeably decreases performance. To avoid this performance loss, we disable the `sse` and `mmx` features (the `avx` feature is disabled by default).

## Integrating Rust

Let’s try to integrate our Rust library into our assembly kernel so that we can call the `rust_init` function. BitOS links `libbitos.a` with the assembly object files:

```Makefile
arch ?= x86_64
kernel := build/kernel-$(arch).bin
iso := build/os-$(arch).iso

linker_script := src/arch/$(arch)/linker.ld
grub_cfg := src/arch/$(arch)/grub.cfg
# Some systems have grub-mkrescue2 instead of grub-mkrescue, so we check for both
grub_mkrescue := $(shell command -v grub-mkrescue 2>/dev/null || command -v grub2-mkrescue 2>/dev/null)
assembly_source_files := $(wildcard src/arch/$(arch)/*.asm)
assembly_object_files := $(patsubst src/arch/$(arch)/%.asm, \
    build/arch/$(arch)/%.o, $(assembly_source_files))
target ?= $(arch)-bitos
rust_os := target/$(target)/debug/libbitos.a

.PHONY: all clean run iso debug gdb

all: $(kernel)

clean:
    @cargo +nightly clean
    @rm -rf build
    @rm -f qemu.log

run: $(iso)
    @qemu-system-x86_64 -cdrom $(iso)

debug: $(iso)
    @qemu-system-x86_64 -cdrom $(iso) -s -S

gdb:
    @rust-gdb build/kernel-x86_64.bin -ex "target remote :1234"

iso: $(iso)

$(iso): $(kernel) $(grub_cfg)
    @mkdir -p build/isofiles/boot/grub
    @cp $(kernel) build/isofiles/boot/kernel.bin
    @cp $(grub_cfg) build/isofiles/boot/grub
    @if [ -z "$(grub_mkrescue)" ]; then \
        echo "error: grub2 mkrescue is required to build the ISO"; \
        exit 127; \
    fi
    @$(grub_mkrescue) -o $(iso) build/isofiles 2> /dev/null
    @rm -r build/isofiles

$(kernel): $(rust_os) $(assembly_object_files) $(linker_script)
    @ld -n -T $(linker_script) -o $(kernel) \
        $(assembly_object_files) $(rust_os)

$(rust_os): Cargo.toml src/lib.rs x86_64-bitos.json .cargo/config.toml
    @cargo +nightly build -Zbuild-std=core,compiler_builtins,alloc --target $(target).json

build/arch/$(arch)/%.o: src/arch/$(arch)/%.asm
    @mkdir -p $(shell dirname $@)
    @nasm -felf64 $< -o $@
```

### Calling Rust

Let's go back to our `longmode.asm` file (`long_mode_start`) and add the following:
```nasm
extern rust_init

bits 64
long_mode_start:
    ...

    ; call the Rust initialisation code
    call rust_init
    hlt
```

By defining `rust_init` as `extern` we tell nasm that the function is defined in another file. As the linker takes care of linking them together, we’ll get a linker error if we have a typo in the name or forget to mark the rust function as `pub extern`.

If we’ve done everything right, `rust_init` is called when executing `make run`. The current implementation prints through the Rust VGA writer and then returns to the assembly code, which halts.

### Linker Errors of doom and despair

Try adding the following rust code to our `rust_init()` function in our `src/lib.rs` file:
```rust
pub extern "C" fn rust_init() {
    let x = ["Hello", "World", "!"];
    let y = x;
}
```

When we test it using `make run`, it fails with `undefined reference to 'memcpy'`. The `memcpy` function is one of the basic functions of the C library (`libc`). Usually the libc crate is linked to every Rust program together with the standard library, but we opted out through `#![no_std]`. We could try to fix this by adding the *libc* crate as extern crate. But `libc` is just a wrapper for the **system** `libc`, for example `glibc` on Linux, so this won’t work for us. Instead we need to recreate the basic `libc` functions such as `memcpy`, `memmove`, `memset`, and `memcmp` in Rust.

**NOTE:** It might not fail anymore, in fact the nightly compiler might be optimizing memcpy away. We WILL STILL NEED `rlibc`, so keep following these steps!

**rlibc - a convenient saviour**

Fortunately there already is a crate for that: `rlibc`. When we look at its [source code](https://github.com/alexcrichton/rlibc/blob/defb486e765846417a8e73329e8c5196f1dca49a/src/lib.rs) we see that it contains no magic, just some *raw pointer* operations in a while loop. To add `rlibc` as a dependency we just need to add two lines to the `Cargo.toml`:

```toml
...
[dependencies]
rlibc = "1.0"
```

and an `extern crate` definition in our `src/lib.rs`:
```rust
...
extern crate rlibc;

#[unsafe(no_mangle)]
pub extern "C" fn rust_init() {
...
```

The target specification already sets this in `x86_64-bitos.json`:

```json
{
  "...",
  "panic-strategy": "abort"
}
```

By setting the `panic` strategy to `abort` instead of the default `unwind`, we disable all unwinding in our kernel. By default, the destructors of all stack variables are run when a `panic` occurs. This is called **unwinding** and allows parent threads to recover from panics. However, it requires a platform specific gcc library, which isn’t available in our kernel. Hence, replacing the panic strategy with `abort` tells Rust not to use unwinding, which will save us from more linker errors if our OS ever panics.

Now, run `cargo clean` to force a recompile (to get rid of any unwind code left over from our OS). Then, we'll try `make run`:

```
> cargo clean
> make run
   Compiling rlibc v1.0.0
   Compiling bitos v0.1.0
warning: unused variable: `test` […]

    Finished dev [unoptimized + debuginfo] target(s) in 0.60 secs
```

It worked! We no longer see linker errors, and our kernel can call Rust code.

## Hello World at long last!

Let's get Hello World printing into our VGA buffer from Rust:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_init() {
    println!("Hello World{}", "!");
}
```

- `println!` is the macro exported by `vga_buff` and writes through the global, mutex-protected writer.
- The writer accesses the VGA buffer at `0xb8000` and writes to the bottom row, scrolling when a newline is emitted.

### Stack Overflows
*yes, like the website*

Since we still use the small 64 byte stack, we must be careful not to overflow it (SIGINT reference!!!). Normally, Rust tries to avoid stack overflows through **guard pages**: The page below the stack isn’t mapped and such a stack overflow triggers a page fault (instead of silently overwriting random memory). But we can’t unmap the page below our stack right now since we currently use only a single big page. Fortunately the stack is located just above the page tables. So some important page table entry would probably get overwritten on stack overflow and then a page fault occurs, too.

![Guard pages in Linux](https://miro.medium.com/1*MwjMXwwoxlbotZpSeRUA1g.png)

### Let's make VGA easier!

So far, we've been printing to our screen by writing a bunch of magic bits to some magic memory location and doing all sorts of finickity 'translations' manually to get our VGA buffer printing. The second half of this workshop will now be about creating an abstraction for the VGA text buffer so that we can print strings in different colours and do so via a simple interface, in, you guessed it, Rust!

# VGA Text Buffer with Rust

## The VGA Text Buffer

The text buffer starts at physical address `0xb8000`, has 25 rows and 80 columns and each screen character has the following format:

| Bit(s) | Value             |
|--------|-------------------|
| 0-7    | ASCII Code        |
| 8-11   | Foreground Colour |
| 12-14  | Background Colour |
| 15     | Blink             |

We can use any of the following colours for the foreground and/or background:

| Number | Colour     | Number + Bright Bit | Bright Colour |
|--------|------------|---------------------|---------------|
| 0x0    | Black      | 0x8                 | Dark Gray     |
| 0x1    | Blue       | 0x9                 | Light Blue    |
| 0x2    | Green      | 0xa                 | Light Green   |
| 0x3    | Cyan       | 0xb                 | Light Cyan    |
| 0x4    | Red        | 0xc                 | Light Red     |
| 0x5    | Magenta    | 0xd                 | Pink          |
| 0x6    | Brown      | 0xe                 | Yellow        |
| 0x7    | Light Gray | 0xf                 | White         |

Bit 4 is the *bright Bit*, which turns blue into light blue, green into light green, etc. It cannot be used in background colour as the bit there is used to control if the text should blink. If you want to use a light background (e.g. white), you'll have to disable blinking through a [BIOS function](http://www.ctyme.com/intr/rb-0117.htm).

## A basic Rust Module

Now that we're familiar with how the VGA buffer works, we can create a Rust module to handle printing:

```rust
// in src/lib.rs
mod vga_buffer;
```

The content of this module can live in either `src/vga_buffer.rs` or `src/vga_buffer/mod.rs`. As we'll want to divide up our code into submodules, we'll want to create the `src/vga_buffer` directory.

### Colours

Create a new file at `src/vga_buffer/colour.rs`, and paste the following code inside:

```rust
#[allow(dead_code)]
#[repr(u8)]
pub enum Colour {
    Black      = 0,
    Blue       = 1,
    Green      = 2,
    Cyan       = 3,
    Red        = 4,
    Magenta    = 5,
    Brown      = 6,
    LightGray  = 7,
    DarkGray   = 8,
    LightBlue  = 9,
    LightGreen = 10,
    LightCyan  = 11,
    LightRed   = 12,
    Pink       = 13,
    Yellow     = 14,
    White      = 15,
}
```

- We use a C-like enum to specify the number for each colour
- `repr(u8)` attribute will make each enum variant be stored as a `u8` (no `u4` type in Rust)
- `allow(dead_code)` attribute disables warnings from the compiler about unused variants

Let's create a struct titled `VgaAttribute` that we'll use to store a full colour code that represents foreground + background colour, and whether or not our text is blinking.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)] // <-- don't worry about this for now
#[repr(u8)]
#[allow(dead_code)]
pub enum Colour {
    Black = 0,
    Blue = 1,
    Green = 2,
    Cyan = 3,
    Red = 4,
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    LightBlue = 9,
    LightGreen = 10,
    LightCyan = 11,
    LightRed = 12,
    Pink = 13,
    Yellow = 14,
    White = 15,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)] // <-- don't worry about this for now
#[repr(transparent)]
pub struct VgaAttribute(u8);

impl VgaAttribute {
    #[allow(dead_code)]
    pub const fn new(blink: bool, fg: Colour, bg: Colour) -> Self {
        Self(((blink as u8) << 7) | (((bg as u8) & 0x07) << 4) | ((fg as u8) & 0x0F))
    }

    pub const fn from_colour(fg: Colour, bg: Colour) -> Self {
        Self((((bg as u8) & 0x07) << 4) | ((fg as u8) & 0x0F))
    }
}
```

The `VgaAttribute` is a new type built on top of `u8`, and contains the full colour byte, containing foreground + background color and blink status. The new function is a [const](https://github.com/rust-lang/rfcs/blob/master/text/0911-const-fn.md) function to allow it in static initializers.

`from_colour` is just an alternative `new` function, where we don't worry about the blink bit at all. This is in case we want light background colours.

### The Text Buffer

We'll now want to add structs that can represent a screen character and the text buffer. Create a new file at `src/vga_buffer/buffer.rs` with the following code:

```rust
use super::VgaAttribute;
use volatile::Volatile; // <-- don't worry about this for now

pub(super) const BUFFER_HEIGHT: usize = 25;
pub(super) const BUFFER_WIDTH: usize = 80;

#[derive(Debug, Clone, Copy, PartialEq, Eq)] // <-- don't worry about this for now
#[repr(C)]
pub(super) struct ScreenChar {
    pub(super) ascii_character: u8,
    pub(super) colour_code: VgaAttribute,
}

#[repr(transparent)]
pub(super) struct Buffer {
    pub(super) chars: [[Volatile<ScreenChar>; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

Some details:
- `repr(C)` means we want our struct's to have a defined field ordering (like they do in C). Rust doesn't have field orderings by default.
- `pub(super)` is used to make items visible to the parent module; since we're splitting our VGA buffer code across multiple files, it'd be good to reuse the definitions we use here instead of redefining them in each file..
- `ScreenChar` contains two structs, one for which ASCII character we're using, and one for the colour code (using `VgaAttribute` defined in `colour.rs`)
- `BUFFER_HEIGHT` and `BUFFER_WIDTH` are the maximum lengths of the height and width of the VGA text buffer. This means we can print in 25 rows, and 80 columns.
- Don't worry about things like `Volatile` and the `derive` for now; we'll cover those later.

Now, once more, we'll make a new file at `/src/vga_buffer/writer.rs` with the following code to start with:

```rust
use core::fmt; // <-- don't worry about this for now

use lazy_static::lazy_static; // <-- don't worry about this for now
use spin::Mutex; // <-- don't worry about this for now

use super::buffer::{BUFFER_HEIGHT, BUFFER_WIDTH, Buffer, ScreenChar}; // import stuff from buffer.rs
use super::{Colour, VgaAttribute}; // import stuff from colour.rs

pub struct Writer {
    pub(super) column_position: usize, // which column we're in
    pub(super) colour_code: VgaAttribute,
    pub(super) buffer: &'static mut Buffer,
    pub(super) do_line_wrapping: bool, // line wrapping at end-of-width
}
```

We'll be using the Writer type to actually write to the screen. Most of the struct is pretty self-explanatory, with the exception of `buffer: &'static mut Buffer`. Let's break that down:
- `&` is a reference to some thing
- `'` is what we prepend to any object to indicate lifetime - how *long* can this buffer be used for?
- `'static`: when you pair the `'` with `static`, we are telling Rust that we want this buffer's lifetime to be the lifetime of the program itself; as long as the program runs, so does the buffer exist & can be used.
- `mut` just marks the Buffer as mutable; i.e. we can change what's inside of it. This is pretty important, since an immutable buffer would be uneditable!

## Printing Characters

Now we'll want to implement behaviour for the `Writer` type. Let's create a method to write a single ASCII byte:

```rust
impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(), // new line chars
            b'\t' => { // tab chars
                let delta = 4 - (self.column_position % 4);
                for _ in 0..delta {
                    self.write_byte(b' ');
                }
            }
            b'\r' => { // carriage returns
                self.column_position = 0;
            }
            byte => {
                if self.column_position >= BUFFER_WIDTH { // check if we're about to go over the edge
                    if self.do_line_wrapping { self.new_line() } 
                    else { return; } // Advanced buffering with scrolling can be implemented later..
                                     // For now an overflow without linewrapping can be dropped
                }
                let row = BUFFER_HEIGHT - 1; // first line
                let col = self.column_position;
                let colour_code = self.colour_code;
                self.buffer.chars[row][col].write(ScreenChar {
                    ascii_character: byte,
                    colour_code,
                });
                self.column_position += 1;
            }
        }
    }
    fn new_line(&mut self) {/* TODO */}
}
```

Hopefully the code above isn't very hard to understand. When we want to write a byte (i.e. a character), we first try to match into a newline, tab or carriage return character to handle those separately. Otherwise, we treat it as any other character. We call a newline if we're about to reach over the end of our BUFFER_WIDTH. Otherwise, for all chars, we get their row, column and colour_code and write this as a `ScreenChar` to the buffer. Afterwards, we increment the column (as we're moving onto the next char). Also, `.write()` is a method brought in by the `volatile` library, more on this later.

### Why `derive Debug, Clone, Copy`?

The code snippets where you've seen `#[derive(Debug, Clone, Copy)]` are necessary to implement debugging, cloning and copying traits for our custom structs. This is because Rust *moves* values by default instead of copying them, which would give us a bunch of borrow-checker errors if we left it unhandled. By using `Clone`, we're telling Rust to clone the struct when its being moved about, we add `Copy` in for [good practice](https://doc.rust-lang.org/core/marker/trait.Copy.html#when-should-my-type-be-copy) and `Debug` for debugging purposes (if we ever need them).

### What's up with Volatile?

Ok, so we've explained the `Debug, Clone, Copy` stuff, but what about the mysterious `Volatile?`. The TLDR is this: compilers like to optimize things, some more aggressively than others. We can't guarantee that for future Rust compilers, their optimization which is meant to be helpful, doesn't break things! Consider that we write to `Buffer` and never read from it again; an aggressive optimization that a compiler might implement is to get rid of this entirely, as it isn't aware that writing to the `Buffer` prints characters onto the screen. That's why we use `Volatile`: we're telling Rust that our *writes* are `Volatile`, that meaning they have side effects that the compiler may not be aware of, and hence should NOT optimize away. You'll need to add the following lines to make `Volatile` work properly:

```toml
# in Cargo.toml

[dependencies]
volatile = "0.2.6"
```

and...

```rust
// in src/lib.rs

extern crate volatile;
```

Furthermore,  this crate (this is how packages are called in the Rust world) provides a Volatile wrapper type with `read` and `write` methods. These methods internally use the read_volatile and write_volatile functions of the standard library and thus guarantee that the reads/writes are not optimized away. That's why we use `.write()` instead of a simple `=` assignment in our `write_byte` function; to make sure the writing isn't optimized away.

## Printing Strings

Surely it's a pain to just be printing each byte individually. To print whole strings, we can write a string, convert it to bytes, and then call `write_byte` repeatedly until we exhaust the string.

```rust
impl Writer {
    ...
    pub fn write_string(&mut self, s: &str) {
        for byte in s.bytes() {
            match byte {
                0x20..=0x7e | b'\n' | b'\t' | b'\r' => self.write_byte(byte),
                _ => self.write_byte(0xfe), // non-valid chars
            }
        }
    }
}
```

### Supporting formatting macros

It would be nice to support Rust’s formatting macros, too. That way, we can easily print different types like integers or floats. To support them, we need to implement the `core::fmt::Write` trait. The only required method of this trait is `write_str` that looks quite similar to our `write_str` method. To implement the trait, we just need to make a copy of it and place it into an `impl fmt::Write for Writer` block and add a return type:

```rust
// in writer.rs

impl fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.write_string(s);
        Ok(())
    }
}
```
The `Ok(())` is just a `Ok` Result containing the `()` type. We can drop the pub because trait methods are always public.

Now we can use Rust's built-in `write!` / `writeln!` formatting macros (no need to copy this):

```rust
use core::fmt::Write;
let mut writer = Writer {...};
writer.write_byte(b'H');
writer.write_str("ello! ");
write!(writer, "The numbers are {} and {}", 42, 1.0/3.0);
```

This would produce a `Hello! The numbers are 42 and 0.3333333333333333` at the bottom of the screen.

### Newlines

Right now, we just ignore newlines and characters that don’t fit into the line anymore. Instead we want to move every character one line up (the top line gets deleted) and start at the beginning of the last line again. To do this, we add an implementation for the `new_line` and `clear_row` methods of `Writer`:

```rust
// in `impl Writer`
    fn new_line(&mut self) {
        for row in 1..BUFFER_HEIGHT {
            for col in 0..BUFFER_WIDTH {
                self.buffer.chars[row - 1][col].write(self.buffer.chars[row][col].read())
            }
        }
        self.clear_row(BUFFER_HEIGHT - 1);
        self.column_position = 0;
    }

    fn clear_row(&mut self, row: usize) {
        let blank = ScreenChar {
            ascii_character: b' ',
            colour_code: self.colour_code,
        };
        for col in 0..BUFFER_WIDTH {
            self.buffer.chars[row][col].write(blank);
        }
    }
}
```

We iterate over all screen characters and move each characters one row up. Note that the range notation `(..)` is exclusive the upper bound. We also omit the 0th row (the first range starts at `1`) because it’s the row that is shifted off screen.

Then, the `clear_row` method clears a row by overwriting all of its characters with a space character.

## Providing an Interface

BitOS initializes the global writer with `lazy_static` and protects it with a spin-lock `Mutex`:

```toml
[dependencies.lazy_static]
version = "1.0"
features = ["spin_no_std"]
```
and using it in the code 
```rust
// src/vga_buff/writer.rs
use lazy_static::lazy_static;
use spin::Mutex;

lazy_static! {
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer {
        column_position: 0,
        colour_code: VgaAttribute::from_colour(Colour::White, Colour::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
        do_line_wrapping: true,
    });
}
```

The `Mutex` allows only one caller to access the writer at a time. The current implementation uses a spin lock, which repeatedly checks whether the resource is free.

To use it we add the spin crate (to avoid us implemnting it now, we will have to later)
```toml
[dependencies]
rlibc = "1.0"
spin = "0.5.2"
```
Now we can lock it before writing:
```rust
use spin::Mutex;

WRITER.lock().write_str("Hello again");
```

To use our writer, we need to export it from `src/vga_buffer/mod.rs`:
```rust
mod buffer;
mod colour;
mod writer;

pub use writer::WRITER;
```


And we can now see it working in our main function
```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_init() {
    println!("Hello World{}", "!");
}
```

## A println macro
Although we *can* now use the `WRITER` in a thread safe way it's in no way to ergonomicly use it.

To start we can create a nice print function that does all of the locking and formatting.

```rust
// src/vga_buff/mod.rs

use core::fmt;

#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    WRITER.lock().write_fmt(args).unwrap();
}
```

```rust
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::vga_buffer::_print(format_args!($($arg)*)));
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}
```


### Clearing the screen

### Line wrapping 

### Hello World using println

### Deadlocks

### Fixing deadlocks


# End of Post

If you have any questions, join our [discord](https://discord.gg/zKhj937xW2) and ask away!

`42 49 54 53 49 47 20 3C 33 20 59 4F 55`
