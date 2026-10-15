+++
date = '2026-10-13T13:39:37+01:00'
draft = true
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

### SIMD Extensions
More formally, `sse` and `mmx` features determine support for SIMD (Single Instruction, Multiple Data) instructions which simultaneously perform an operation (like addition) on multiple data words. The `x86` architecture supports the following standards:
- **MMX**: *Multi Media Extension* instruction set defines eight 64 bit registers called `mm0` through `mm7`. These registers are just *aliases* for the registers of the x87 floating point unit.
- **SSE**: *Streaming SIMD Extensions* instruction set adds a completely new register set (instead of reusing floating point registers). The sixteen new registers are called `xmm0` through `xmm15` and are 128 bits each.
- **AVX**: *Advanced Vector Extensions* are extensions that further increase the size of the multi media registers. The new registers are called `ymm0` through `ymm15` and are 256 bits each. They extend the `xmm` registers, so `xmm0` is just the lower half of `ymm0` as an example.

By using such SIMD standards, programs can often speed up significantly. Good compilers are able to transform normal loops into such SIMD code automatically through a process called [auto-vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization).

However, the large SIMD registers lead to problems in OS kernels. The reason is that the kernel has to backup all registers that it uses on each hardware interrupt (recall what we were talking about with PCBs). So if the kernel uses SIMD registers, it has to backup a lot more data, which noticeably decreases performance. To avoid this performance loss, we disable the `sse` and `mmx` features (the `avx` feature is disabled by default).

### Compiling

To build our kernel for our new target, we pass the configuration file’s name as --target argument. You may have a bug where the `RUST_TARGET_PATH` variable isn't set properly, so we'll have to set it to the current working directory. Try running this command:

`RUST_TARGET_PATH=$(pwd) cargo build --target x86_64-blog_os`

You're likely to get the following error:

```
error[E0463]: can't find crate for `core`
  |
  = note: the `x86_64-blog_os` target may not be installed
```

The error tells us that the Rust compiler no longer finds the core library. The core library is implicitly linked to all `no_std` crates and contains things such as `Result`, `Option`, and iterators.

The problem is that the core library is distributed together with the Rust compiler as a *precompiled* library. So it is only valid for the underlying host triple (e.g., `x86_64-unknown-linux-gnu`) but not for our custom target. To compile code for other targets, we'll have to recompile code for them first as well!

That's where [xargo](https://github.com/japaric/xargo) comes in, which is a wrapper for cargo that makes cross compilation easier. Install it with (we'll need rust-src as well as it's a xargo dependency):

`cargo install xargo`
`rustup component add rust-src`

Xargo is *“a drop-in replacement for cargo”*, so every cargo command also works with xargo. However, the build command gains additional functionality: `xargo build` will automatically cross compile the core library when compiling for custom targets.

Let's try running it:

```bash
> RUST_TARGET_PATH=$(pwd) xargo build --target=x86_64-[your_os_name]
   Compiling core v0.0.0 (file:///…/rust/src/libcore)
    Finished release [optimized] target(s) in 22.87 secs
   Compiling your_os_name v0.1.0 (file:///…/your_os_name/tags)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29 secs
```

It worked! We were able to cross compile `core` for our custom target and then compiled our OS crate. After compilation, we can find a static library at `target/x86_64-[your_os_name]/debug/lib[your_os_name].a` which can be linked with our assembly kernel.

## Integrating Rust

Let’s try to integrate our Rust library into our assembly kernel so that we can call the rust_main function. For that we need to pass the `lib[your_os_name].a` file to the linker, together with the assembly object files. To do this, we'll extend our makefile:

```Makefile
# ...
target ?= $(arch)-your_os_name
rust_os := target/$(target)/debug/libyour_os_name.a
# ...
.PHONY: all clean run iso kernel
# ...
$(kernel): kernel $(rust_os) $(assembly_object_files) $(linker_script)
	@ld -n -T $(linker_script) -o $(kernel) \
		$(assembly_object_files) $(rust_os)

kernel:
	@RUST_TARGET_PATH=$(shell pwd) xargo build --target $(target)
```

We add a new kernel target that just executes `xargo build` and modify the `$(kernel)` target to link the created static lib. We also add the new `kernel` target to the `.PHONY` list, since it does not belong to a file with that name.

### Calling Rust

Let's go back to our `longmode.asm` file (`long_mode_start`) and add the following:
```rust
bits 64
long_mode_start:
    ...

    ; call the rust main
    extern rust_main     ; new
    call rust_main       ; new

    ; print `OKAY` to screen
    mov rax, 0x2f592f412f4b2f4f
    mov qword [0xb8000], rax
    hlt
```

By defining `rust_main` as `extern` we tell nasm that the function is defined in another file. As the linker takes care of linking them together, we’ll get a linker error if we have a typo in the name or forget to mark the rust function as `pub extern`.

If we’ve done everything right, we should still see the green `OKAY` when executing `make run`. That means that we successfully called the Rust function and returned back to assembly.

### Linker Errors of doom and despair

Try adding the following rust code to our `rust_main()` function in our `src/lib.rs` file:
```rust
pub extern fn rust_main() {
    let x = ["Hello", "World", "!"];
    let y = x;
}
```

When we test it using `make run`, it fails with `undefined reference to 'memcpy'`. The `memcpy` function is one of the basic functions of the C library (`libc`). Usually the libc crate is linked to every Rust program together with the standard library, but we opted out through `#![no_std]`. We could try to fix this by adding the *libc* crate as extern crate. But `libc` is just a wrapper for the **system** `libc`, for example `glibc` on Linux, so this won’t work for us. Instead we need to recreate the basic `libc` functions such as `memcpy`, `memmove`, `memset`, and `memcmp` in Rust.

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

#[no_mangle]
pub extern fn rust_main() {
...
```

Now `make run` doesn’t complain about `memcpy` anymore. Instead it will show a pile of new ugly linker errors (WE LOVE LOW-LEVEL DEV!!!):
```
target/x86_64-your_os_name/debug/libyour_os_name.a(core-92335f822fa6c9a6.0.o):
    In function `_$LT$f32$u20$as$u20$core..num..dec2flt..
        rawfp..RawFloat$GT$::from_int::h50f7952efac3fdca':
    core.cgu-0.rs:(.text._ZN59_$LT$f32$u20$as$u20$core..num..dec2flt..
        rawfp..RawFloat$GT$8from_int17h50f7952efac3fdcaE+0x2):
    undefined reference to `__floatundisf'
target/x86_64-your_os_name/debug/libyour_os_name.a(core-92335f822fa6c9a6.0.o):
    In function `_$LT$f64$u20$as$u20$core..num..dec2flt..rawfp..
        RawFloat$GT$::from_int::h12a81f175246914a':
    core.cgu-0.rs:(.text._ZN59_$LT$f64$u20$as$u20$core..num..dec2flt..rawfp..
        RawFloat$GT$8from_int17h12a81f175246914aE+0x2):
    undefined reference to `__floatundidf'
target/x86_64-your_os_name/debug/libyour_os_name.a(core-92335f822fa6c9a6.0.o):
    In function `core::num::from_str_radix::h09b12650704e0508':
    core.cgu-0.rs:(.text._ZN4core3num14from_str_radix
        17h09b12650704e0508E+0xcf):
    undefined reference to `__muloti4'
...
```

**-gc-sections**

The new errors are linker errors about various missing functions such as `__floatundisf` or `__muloti4`. These functions are part of *LLVM’s `compiler-rt` builtins* and are normally linked by the standard library. For `no_std` crates like ours, one has to link the `compiler-rt` library manually. Unfortunately, this library is implemented in C and the build process is a bit cumbersome. Alternatively, there is the [compiler-builtins](https://github.com/rust-lang-nursery/compiler-builtins) crate that tries to port the library to Rust, but we're not sure if it's ready-for-use yet.

So instead, we opt for a simpler solution. Our kernel doesn’t really need any of those functions yet. So we can just tell the linker to remove unused program sections and hopefully all references to these functions will disappear. Removing unused sections is generally a good idea as it reduces kernel size. The magic linker flag for this is `--gc-sections`, which stands for “garbage collect sections”. Let’s add it to the `$(kernel)` target in our `Makefile`:

```Makefile
$(kernel): xargo $(rust_os) $(assembly_object_files) $(linker_script)
	@ld -n --gc-sections -T $(linker_script) -o $(kernel) \
		$(assembly_object_files) $(rust_os)
```

Now we can do a `make run` again and it compiles without errors again. However, it doesn’t boot anymore:
`GRUB error: no multiboot header found.`

What? Our multiboot header is right there? What happened? Well, the linker removed unused sections, and since we don’t use the Multiboot section anywhere, `ld` removes it too. So we need to tell the linker explicitly that it should keep the multiboot header. The `KEEP` command does exactly that, so we add it to the linker script (`linker.ld`):
```
.boot :
{
    /* ensure that the multiboot header is at the beginning */
    KEEP(*(.multiboot_header))
}
``` 

And for sanity's sake, you should also add this line to your `x86_64_your_os_name.json` file:

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
   Compiling your_os_name v0.1.0 (file://[path_to_your_os_name]/tags)
warning: unused variable: `test` […]

    Finished dev [unoptimized + debuginfo] target(s) in 0.60 secs
```

It worked! We no longer see linker errors, and our kernel prints `OKAY` again!

## Hello World at long last!

Let's get Hello World printing into our VGA buffer from Rust:

```rust
#[no_mangle]
pub extern fn rust_main() {
    // ATTENTION: we have a very small stack and no guard page

    let hello = b"Hello World!";
    let color_byte = 0x1f; // white foreground, blue background

    let mut hello_colored = [color_byte; 24];
    for (i, char_byte) in hello.into_iter().enumerate() {
        hello_colored[i*2] = *char_byte;
    }

    // write `Hello World!` to the center of the VGA text buffer
    let buffer_ptr = (0xb8000 + 1988) as *mut _;
    unsafe { *buffer_ptr = hello_colored };

    loop{}
}
```

- `b` prefix creates a *byte string*, which is just an array of `u8`s (unsigned 8 bit integers)
- *enumerate* is an `Iterator` method that adds the current index `i` to elements
- `buffer_ptr` is a raw pointer that points to the center of the VGA text buffer
- Rust doesn't know the VGA buffer and so cannot guarantee that writing to the `buffer_ptr` is safe, so we tell Rust that we know what we're doing by wrapping it in an `unsafe{}` block.

### Stack Overflows
*yes, like the website*

Since we still use the small 64 byte stack, we must be careful not to overflow it (SIGINT reference!!!). Normally, Rust tries to avoid stack overflows through **guard pages**: The page below the stack isn’t mapped and such a stack overflow triggers a page fault (instead of silently overwriting random memory). But we can’t unmap the page below our stack right now since we currently use only a single big page. Fortunately the stack is located just above the page tables. So some important page table entry would probably get overwritten on stack overflow and then a page fault occurs, too.

![Guard pages in Linux](https://miro.medium.com/1*MwjMXwwoxlbotZpSeRUA1g.png)

### Let's make VGA easier!

So far, we've been printing to our screen by writing a bunch of magic bits to some magic memory location and doing all sorts of finickity 'translations' manually to get our VGA buffer printing. The second half of this workshop will now be about creating an abstraction for the VGA text buffer so that we can print strings in different colours and do so via a simple interface, in, you guessed it, Rust!

# VGA Text Buffer with Rust Macros



# End of Post

If you have any questions, join our [discord](https://discord.gg/zKhj937xW2) and ask away!

`42 49 54 53 49 47 20 3C 33 20 59 4F 55`
