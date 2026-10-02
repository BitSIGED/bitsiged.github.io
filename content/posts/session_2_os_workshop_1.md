+++
date = '2026-10-01T12:46:55+01:00'
draft = false
title = 'Session 2 (Workshop 1): Minimal Kernel'
tags = ['Session', 'Operating Systems', 'Workshop']
summary = "First workshop of Sem 1: building a minimal kernel"
+++

# Intro

**Session 2** ran on Thursday, 1st October in AT7.14. The session was a code-writing workshop on writing a minimal kernel as part of the 'Write-Your-Own-OS' series of workshops that will be done throughout semester 1.
This session was ran by **Kacper** and **Archie**.

## Credit & Disclaimer:
This guide is intended for self-study / for people to follow along themselves. The sessions will cover the ideas in broad strokes and clarify parts where it gets confusing, but overall we aim to make each one of the workshop posts something that somebody could, ideally, do from home. Any slides that we present in workshop sessions will be available in the 'session content' section of this post.

We use a lot of material from Philipp Oppermann's blog *'Writing an OS in Rust (First Edition)'*; in fact, our workshop posts will effectively use his materials as a baseline that we will build on with theory, further ideas and may diverge in some cases where we thought it useful. Please check out his blog [here](https://os.phil-opp.com/edition-1/); our work would be much harder without his content!

We also use supplementary materials, like the CS216 guide to x86 assembly from the University of Virginia which you can find [here](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html). Thanks to David Evans, Adam Ferrari, Alan Batson, Mike Lack, Anita Jones + all other contributors to this guide!

**Furthermore, the majority of the content of these workshops will be written assuming a Linux host machine in mind. We can assist with some differences in distros, but we'd ask that if you follow this guide with a macOS or Windows system, you should use a Linux VM (like WSL).**

## Session Content:
Empty at the moment - check back later!

# A minimal multiboot Rust Kernel

We'll be going into how to create a minimal x86 OS kernel with the Multiboot standard. For now, all we want is to get `OK` to print to the screen. But before we can get into doing any of that, we need to understand how a computer boots.

## Booting sequence

When a computer switches on, it loads the **BIOS** (Basic Input/Output System) from some special flash memory. The BIOS will run self-test and hardware initializations and then looks for bootable devices (HDDs, USBs, etc.). *In the case of B.O.C., this is usually a USB since B.O.C currently has no HDD or SSD!* Once a bootable device is found, the **bootloader** takes control which has to determine the location of the kernel image on the bootable device and load it into memory. The bootloader also needs to switch the CPU into **'protected mode'** as all x86 CPUs start in the limited **'real mode'** by default (for old computers compatibility). To give you an idea, real mode generally has access to about 1MB of physical RAM because of how it does addressing, while protected mode can do up to 4GB in 32-bit addressing (and more in 64 bit via extension).

![Booting Diagram](img/booting.png)

So, our current task is to get an already-existing bootloader to boot our kernel, and that's where **Multiboot** and **GRUB 2** comes in!

## Multiboot
There exists a bootloader standard known as the **Multiboot Specification**. Our kernel will need to indicate that it supports Multiboot and then every Multiboot-compliant bootloader can boot it. We can pair Multiboot 2 in particular with the GRUB 2 bootloader to achieve all the booting needs we may have!

Our kernel needs to tell the bootloader that Multiboot 2 is supported, which we'll do with the *Multiboot Header*. This is effectively some boilerplate magic, but we'll try to explain it as much as we can. Create the x86 assembly file `multiboot_header.asm` below.

```nasm
section .multiboot_header
header_start:
    dd 0xe85250d6                ; magic number (multiboot 2)
    dd 0                         ; architecture 0 (protected mode i386)
    dd header_end - header_start ; header length
    ; checksum
    dd 0x100000000 - (0xe85250d6 + 0 + (header_end - header_start))

    ; insert optional multiboot tags here

    ; required end tag
    dw 0    ; type
    dw 0    ; flags
    dd 8    ; size
header_end:
```

Definitely scary for people who don't know x86 assembly, so we'll build on the quick guide from the Phil Opp blog here and go over the basics of x86 assembly.

## [x86 Assembly 101](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html) (useful guide from Uni of Virginia CS Dept - CS216)

**Note: you can skip this section if you are pretty familiar with x86 assembly!**

### Registers
Modern x86 processors have eight **32-bit** general purpose registers, called `EAX, EBX, ECX, EDX, ESI, EDI, ESP, EBP`. `ESP` and `EBP` are both reserved for special purposes; they correspond to the Stack Pointer and the Base Pointer. We can also subdivide the first 16-bits of `EAX, EBX, ECX, EDX` into two 8-bit registers, meaning that technically `EAX` can, for example, contain 3 registers (one 16-bit, two 8 bits).

### Declaring data

We can declare static data regions which are analogous to global variables. In our case, we are looking at directives such as `dd` (declare double, 32-bit), `dw` (declare word, 16-bit). These directives will output whatever we specify, so for example `dd 0xe85250d6` will output the double `0xe85250d6` (this is a magic number used to identify that our header is a multiboot 2 header).

There's also `db` (declare byte), which is 8-bits.

Note that in our `multiboot_header.asm`, we don't use any variables, meaning that all of these declarations will store the values next to each other in memory. If we were to use variables (aka locations), our lines would look like this for example:
```
var dd 0 ; declare a byte, referred to as var, containing the value 0
```

The reason we don't use location names is because we want all parts of our multiboot header to be in the same memory location so that the bootloader can read the full header in one place.

We can also declare arrays, which are just stored contiguously in memory. For example: `B DD 1,2,3` would store three 4-byte values (1,2,3) at `B`.

Perhaps not entirely in the 'declaring data' section but useful to mention regardless, the `global` keyword is used to export a label (i.e. make it public). In x86 (and other assembly langs), labels are used to label sections of code. By making a label `global`, we can reference it from outside the file!

### Addressing Memory
Modern x86-compatible processors are capable of addressing up to 2^32 bytes of memory: memory addresses are 32-bits wide. We can also use two 32-bit registers and a 32-bit signed constant added together to compute a memory address (e.g. `0x100000 + 0x000004`).

Let's look at some examples of `mov` instructions using address computations:

```nasm
mov eax,[ebx] ; move the 4 bytes in mem at EBX into EAX

mov [var], ebx ; move contents of EBX into 4 bytes at var (var is a 32-bit constant)

mov eax, [esi-4] ; move 4 bytes at memory address ESI + (-4) into EAX

mov [esi+eax], cl ; move the contents of CL into the byte at address ESI+EAX
```

### Data Movement + Arith and Logic Instructions
Here we'll just list the instructions and what they generally do; if you want to read more about them, click on the link attached to the **x86 Assembly 101** section title.

- `mov`: move. Copy a data item referred to by its second operand into the location referred to by its first operand.
- `push`: push stack. Place an operand onto the top of the stack in memory.
- `pop`: pop stack. Pop an operand off the top of the stack in memory.
- `lea`: load effective address. Places the address specified by its second operand into the register specified by its first operand (not the contents of the mem location).

- `add`: add.
- `sub`: subtract.
- `inc, dec`: increment or decrement by one.
- `imul`: integer multiplication
- `idiv`: integer division
- `and,or,xor`: Bitwise logical ANDs, ORs and XOR operations.
- `not`: Bitwise logical NOT.
- `neg`: negate. Performs the two's complement negation of the operand contents.
- `shl, shr`: bit shift left and bit shift right.

### Control Flow Instructions

Similarly to the last section, we'll just list the instructions and a small text on what they do. Control Flow instructions are quite important to understand in general, so get used to them:

- `jmp <label>`: jump. Transfer program control flow to the instruction at the memory location indicated by the operand.
- `jcondition <label>`: conditional jump. Same as jump but **on a condition**. We replace `condition` with the condition we'd actually like to check for, e.g.:
    - `je`: jump when equal
    - `jne`: jump when not equal
    - `jge:` jump when greater than
    - ... etc.

- `cmp`: compare. Compare the values of the two specified operands, setting the condition codes in the machine status word appropriately. We could pair this with a subsequent instr like `jeq` to determine what happens depending on result of `cmp`.

- `call, ret`: subroutine call & return - don't necessarily need to know this at the moment, but handy to know if you want to do subroutines (functions).

### Additional useful bits (for this workshop)
- `.text` section is the default section for executable code
- `bits 32` specifies that the following lines are 32-bit instructions. `bits 64` would specify that they're 64-bit, etc.
- `hlt` halts the CPU.

While not an indepth guide to x86 assembly, this should give you enough of an idea of how things work to understand the x86 we use!

## Endianness

Anyhow, we can use the `nasm` command to assemble `multiboot_header.asm`. If we hexdump the assembled flat binary, we can actually spot a good-to-be-aware-of feature of x86: **Little-Endian**.

```
> nasm multiboot_header.asm
> hexdump -x multiboot_header
0000000    50d6    e852    0000    0000    0018    0000    af12   17ad
0000010    0000    0000    0008    0000
0000018
```

You probably have heard of Endianness before; it is the order of bytes stored in computer memory for data types that consist of multiple bytes (e.g. strings). There's two conventions, **Big-Endian** and **Little-Endian**.

- **Big-Endian**: stores the most significant byte ('big end') at the lowest memory address. This matches how we naturally read numbers from left to right.
- **Little-Endian**: stores the least significant byte ('little end') at the lowest memory address.

So how does that hex dump demonstrate Little-Endian? If you take a look at `50d6` and `e852`, it's clear that `50d6` appears first (e.g. at a lower area in memory) and then `e852` follows. However, when we wrote our x86 assembly above, we declared the double `0xe85250d6`, where clearly `e852` is the most significant byte and `50d6` is the least significant byte. So clearly, since this order appears 'swapped' in our hex dump, we're using Little-Endian.

You can save yourself this entire investigation by just searching up that x86 uses Little-Endian as standard, but it's fun to spot little things like that to cement your understanding.

## Boot Code

To boot our kernel, we need some code that the bootloader can call. Create a file called `boot.asm` and write the following:

```nasm
global start

section .text
bits 32
start:
    ; print `OK` to screen
    mov dword [0xb8000], 0x2f4b2f4f
    hlt
```

### VGA Buffer: How do we "print 'OK'"?

Ok, so we can see that we are declaring a word `0x2f4b2f4f` and moving it to the memory address `0xb8000`. How is this enough to print 'OK' to the screen? There's no mention of 'OK' at all like you'd see in `print('OK')` in Python or some other higher-level language. And what is this `0xb8000` address?

`0xb8000` is where the **VGA text buffer** begins. It's an array of screen characters that are displayed by the graphics card. Nowadays some people can call VGA displays retro, but they're quite useful for our minimal kernel because we can get stuff printing to the screen!

So then it seems that if we move a word to `0xb8000`, the VGA buffer will render it. Let's dissect `0x2f4b2f4f` - what is it actually doing?

Characters that we're sending to the VGA buffer come in two parts; 8 bits for a color code, and 8 bits for an ASCII character. `2f` is the color code for white text on a green background, and `4b` and `4f` are ASCII `K` and `O` respectively. The reason we get `OK` and not `KO` is because of Little-Endian (see Endianness section above). So in its entirety, it's like we're saying "okay, I'm sending stuff to the VGA buffer. I'm sending `2f4b` which means white text on green background, letter K, and `2f4f`, which has the same colour (`2f`) but the letter O this time".


## Building the executable
Okay, great, we've got the boot code and the multiboot header which will tell GRUB that our kernel supports Multiboot 2. But GRUB will need something called an **ELF** (Executable and Linkable Format)  Executable, and our files to be **ELF** object files, which we can get `nasm` to create by passing the `-f elf64` flag when using `nasm` to assemble our code.

Then, the **ELF** executable is created by *linking* the **ELF** object files together. This is done by something called a *linker*, which in summary 'links' multiple files / objects together into one main executable. To do this, we create a custom linking script called `linker.ld`:

```ld
ENTRY(start)

SECTIONS {
    . = 1M;

    .boot :
    {
        /* ensure that the multiboot header is at the beginning */
        *(.multiboot_header)
    }

    .text :
    {
        *(.text)
    }
}
```

- `start` is the entry point which the bootloader will jump to after loading the kernel
- `. = 1M;` sets the load address of the first section to 1 MiB (convention)
- the executable will be made of two sections: `.boot` first and `.text` afterwards. `.text` output section contains all input sections named `.text`.
- Sections named `.multiboot_header` are added to the first output section; necessary as GRUB expects to find the Multiboot header very early in the file.

With the linker now ready, we can create the ELF object files and link them together into the ELF executable that we're after:

```bash
> nasm -f elf64 multiboot_header.asm
> nasm -f elf64 boot.asm
> ld -n -o kernel.bin -T linker.ld multiboot_header.o boot.o
```

It's important that we pass the `-n` flag to the linker to disable the automatic section alignment in the executable. Otherwise the linker may page align the `.boot` section in the executable file, thus moving it away from the beginning. GRUB needs `.boot` to be at the beginning, as thats where it can find the Multiboot header.

## Creating the ISO

All PC BIOSes know how to boot from a CD-ROM, so we'll want to create a bootable CD-ROM image which contains our kernel and the GRUB bootloader's files in an **ISO** (Optical Disc Image). Create the following directory structure and make sure `kernel.bin` is copied to the right place:
```
isofiles
└── boot
    ├── grub
    │   └── grub.cfg
    └── kernel.bin
```

The `grub.cfg` file will specify the file name of our kernel and its Multiboot 2 compliance to GRUB:

```
set timeout=0
set default=0

menuentry "my os" {
    multiboot2 /boot/kernel.bin
    boot
}
```

We can now create a bootable image:
```
grub-mkrescue -o os.iso isofiles
```

Note that `grub-mkrescue` doesn't work on all platforms. On Fedora Linux, for example, we need to use `grub2-mkrescue`. Other steps you can try include:
- try running it with `--verbose`
- make sure `xorriso` is installed
- if you're using an EFI system, `grub-mkrescue` will try to create an EFI image by default. You can either pass `-d /usr/lib/grub/i386-pc` to avoid EFI or install the `mtools` package to get a working EFI image.

## Booting & Build Automation
We can boot our OS using **QEMU**:
```
qemu-system-x86_64 -cdrom os.iso
```

... which should show you a green `OK` in the upper left corner. If this didn't work for you, see the comments on the original [Phil Oppermann post](https://os.phil-opp.com/multiboot-kernel/) for potential bug fixes.

So, in summary:
1. BIOS loads the bootloader (GRUB) from the virtual CD-ROM (the ISO)
2. the bootloader reads the kernel executable and finds the multiboot header
3. it copies the `.boot` and `.text` sections to memory (`0x100000` and `0x100020` respectively)
4. it jumps to the entry point (`0x100020`)
5. our kernel prints the green `OK` and stops the CPU.

**Important:** We've also found at our sessions some people being unable to boot their custom OS in QEMU because their underlying system is EFI (and `grub-mkrescue` will attempt to produce EFI boot images instead of a BIOS boot image for QEMU). Thanks to *emk1024* from Phil Opp's blog, the solution is to install `grub-pc-bin` and run `grub-mkrescue /usr/lib/grub/i386-pc -o myos.iso isodir`.

Congrats, you've got your kernel to boot! At this stage, you could even put it on a USB stick and boot it onto B.O.C!

However, let's do some build automation first as typing out all those commands each time you want to build your ISO is gonna get real tedious real fast. We'll use `Makefile` for this. Firstly, create the following directory structure:
```
├── Makefile
└── src
    └── arch
        └── x86_64
            ├── multiboot_header.asm
            ├── boot.asm
            ├── linker.ld
            └── grub.cfg
```

Then save the Makefile with this content (this is a custom Makefile written by **Archie** that specifically checks for systems that may need to use `grub2-mkrescue` vs `grub-mkrescue`):

```Make
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

.PHONY: all clean run iso

all: $(kernel)

clean:
	@rm -r build

run: $(iso)
	@qemu-system-x86_64 -cdrom $(iso)

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

$(kernel): $(assembly_object_files) $(linker_script)
	@ld -n -T $(linker_script) -o $(kernel) $(assembly_object_files)

# compile assembly files
build/arch/$(arch)/%.o: src/arch/$(arch)/%.asm
	@mkdir -p $(shell dirname $@)
	@nasm -felf64 $< -o $@
```

If your build doesn't work with this Makefile, there may be other cross-platform problems / incompatibilities so please let us know in-session or ask on our Discord.

# Foreword on Long Mode & Paging

## What is Long mode 

At the moment our CPU has started up in 32-bit mode for backwards compatibility reasons even if our CPU is 64-bit, Because of this we have to do all the preparation and validation to switch us to 64 bit mode (referred to as **long mode**) ourself. This includes 
- Validating the CPU supports Long Mode 
- Enabling paging 
- Configuring the [Global Descriptor Table](https://wiki.osdev.org/Global_Descriptor_Table)

## Paging

You might have heard of virtual memory before, and should definitely know about physical memory. After all, one of these things we can see physically, and the other thing is, well, virtual. **Paging** is a memory management scheme that links physical memory and virtual memory. The logical / virtual address space is split into equal sized *pages* and a *page table* specifies which **virtual page** points to which **physical page**. 

In long mode, x86 uses a page size of 4096 bytes and a 4 level page table that consists of:
- the Page-Map Level-4 Table (PML4)
- the Page-Directory Pointer Table (PDP)
- the Page-Directory Table (PD)
- the Page Table (PT)

To simplify things, let's call them P4,P3,P2,P1 from now on. Each page table contains 512 entries and one entry is 8 bytes, so they fit exactly in one 'page' (`512*8 = 4096`, so 512 entries on one page). Let's think about what happens when we want the CPU to translate a virtual address to a physical address:

![Paging Explained](https://os.phil-opp.com/entering-longmode/X86_Paging_64bit.svg)

1. Get the address of the `P4` table from the `CR3` register
2. Use bits 39-47 (9 bits) as an index into `P4` (`2^9 = 512 = num of entries`)
3. Use the following 9 bits as an index into `P3`
4. Use the following 9 bits as an index into `P2`
5. Use the following 9 bits as an index into `P1`
6. Use the last **12** bits as page offset (`2^12 = 4096 = page size`)

The eagle-eyed might notice that we don't do anything to bits 48-63 of the 64-bit virtual address. They can't be used; the '64-bit' long mode is in fact just a 48-bit mode. The 48-63 bits must be copies of bit 47, so each valid virtual address is still unique. Wikipedia seems to suggest that the main reason for the '48-bit mode' is that we don't need full 64-bit addressing as that would drastically raise complexity / the number of potential virtual addresses, much more than we'd ever need really.

An entry in the P4, P3, P2 and P1 tables consist of the page aligned 52-bit *physical address* of the frame or the next page table and the following bits that can be `OR`-ed in:

| Bits  | Name                  | Meaning                                                                                      |
|-------|-----------------------|----------------------------------------------------------------------------------------------|
| 0     | present               | the page is currently in memory                                                              |
| 1     | writable              | it's allowed to write to this page                                                           |
| 2     | user accessible       | if not set, only kernel mode code can access this page                                       |
| 3     | write through caching | writes go directly to memory (write-through policy)                                          |
| 4     | disable cache         | no cache is used for this page                                                               |
| 5     | accessed              | the CPU sets this bit when this page is used                                                 |
| 6     | dirty                 | the CPU sets this bit when a write to this page occurs                                       |
| 7     | huge page / null      | must be 0 in P1 & P4, creates a 1 GiB page in P3, creates a a 2 MiB page in P2               |
| 8     | global                | page isn't flushed from caches on address space switch (PGE bit of CR4 register must be set) |
| 9-11  | available             | can be used freely by the OS                                                                 |
| 52-62 | available             | can be used freely by the OS                                                                 |
| 63    | no execute            | forbid executing code on this page (the NXE bit in the EFER register must be set)            |

# Enabling long mode

We will need to check the CPU we're running on has all the features we need, if not we should exit with a clear error message explaining the issue. To handle this we create a little error handler stub that accepts a error code in `al`

```nasm
; parameter `al` contains the error code 
error:
  mov dword [0xb8000], 0x4f524f45 ; `ER`
  mov dword [0xb8004], 0x4f3a4f52 ; `R:`
  mov dword [0xb8008], 0x4f204f20 ; `  `
  mov byte  [0xb800a], al ; the error code 
  hlt  ; stop the operating system
```

We will also need some stack space to call functions and store variables, however, as we've not set up memory yet we will need to allocate some temporary scratch space in the `bss` section of the kernel binary.
The `.bss` section is a section of memory that we can both *read* and *write* to (different to `.text` which is *read* and *execute* only) and it is automatically loaded by grub letting us use a stack to validate the memory features without having to juggle registers.

```nasm
section .bss
boot_stack_bottom:
    resb 64 ; reserve 64 bytes for the stack
boot_stack_top:
```
We then want to update our start method to use the new stack
```nasm
section .text
bits 32
start: 
    mov esp, stack_top ; add the top of the stack to the esp reg (remember stack grows down)
    ; ... 
```

## Checking for multiboot2

We will use multiboot2 to initialize our kernel hence we will need to make sure we're using a bootloader that uses it. If we are using it `eax` will hold the magic bytes `0x36d76289`.
Hence we can take advantage of our new stack and error handler and write a basic checker function.

```nasm
check_multiboot:
  cmp eax, 0x36d76289 ; multiboot2 magic
  jne .no_multiboot
  ret
.no_multiboot:
  mov al, "0"
  jmp error
```

## Checking for CPUID

[CPUID](https://revers.engineering/x86/cpuid.pdf) is an instruction that lets us query information about the CPU we're running on (however some older CPU's don't support it so we need to check we've got it).
If we are running on a CPU with `CPUID` support then the "flags" register will let us flip the `ID` bit otherwise it'd remain the same no matter what we do.

To do this we:
1. Copy the `flags` into a register and try flip the `ID` bit
2. Set `flags` to the new value with the flipped bit. 
3. Re-read `flags` and see if the bit flip saved.
4. Reset `flags` back to its initial state.

```nasm
check_cpuid:
  ; Step 1, Store the flags into `eax`
  pushfd; pushes the value of the `flags` register onto the stack
  pop eax
  ; store the initial flags for restoration later
  mov ecx, eax
  ; flip the `CPUID` bit
  xor eax, 1 << 21
  
  ; Step 2, copy the changed flags into FLAGS
  push eax
  popfd

  ; Step 3, re read flags to check if we were able to set CPUID
  pushfd
  pop eax

  ; Step 4, restore original flags (done now to prevent duplicated cleanup code)
  push ecx
  popfd

  ; Validate the `CPUID` bit flip worked
  cmp eax, ecx
  je .no_cpuid
  ret

.no_cpuid:
  mov al, "1"
  jmp error
```

## Checking for long mode

Now we've a way to validate we have the `CPUID` instruction we can use it to detect long mode support on the CPU. 

To Do this we query the `CPUID` to check if we have support for long mode 
```nasm
check_long_mode:
  ; eax is the argument and return to CPUID 
  mov eax, 0x80000000 ; query what the max CPUID querys we have (as long mode query is in the extended) 
  cpuid
  cmp eax, 0x80000001 ; if were not greater than 0x80000001 then we've definitely no long mode  
  jb .no_long_mode 
  
  mov eax, 0x80000001 
  cpuid ; get extended processor info (including long mode support query)
  test edx, 1 << 29 ; check for the long mode bit 
  jz .no_long_mode
  ret 

.no_long_mode:
  mov al, "2" 
  jmp error
```

Have a read of [Wikipedia](https://en.wikipedia.org/wiki/CPUID#EAX=8000'0001h:_Extended_Processor_Info_and_Feature_Bits) to find out more

## Enabling Paging
### Setting up page tables 

Now we know we can enter long mode we will need to set up an identity map so we can access ram. 

*For those who've previously done the phil-opp blog this is where we start differing they enable huge page but we will not.*

We start by setting aside some space for our page tables
```nasm

P1_COUNT equ 4

section .text
; ...

section .bss 
align 4096
p4_table:
    resb 4096
p3_table:
    resb 4096
p2_table:
    resb 4096
p1_tables:
    resb 4096 * P1_COUNT
stack_bottom:
```

We then setup the first p4_table entry to point to the p3_table, and the first p3_table entry to point to p2_table

```nasm
set_up_page_tables: 
  ; map first P4 entry to P3 table 
  mov eax, p3_table
  or eax, 0b11 ; present + writable 
  mov [p4_table], eax
  ; map P3 to P2
  mov eax, p2_table
  or eax, 0b11
  mov [p3_table], eax
  ; ...
```

We then fill out the first 4 values of the second page table with pointers to our p1 tables (we do this instead of just doing one entry in p2 to give us some more space as we're not doing large tables)

*we use ecx as our `i` and have this act as a for loop, feel free to ask for help if you don't understand whats happening*

```nasm
  ; ...
  mov ecx, 0
.map_p2:
  mov eax, ecx 
  shl eax, 12 ; ecx * 4096 aka side of one P1 table 
  add eax, p1_tables
  or eax, 0b11 ; present and writable
  mov [p2_table + ecx * 8], eax 
  inc ecx 
  cmp ecx, P1_COUNT
  jne .map_p2

  ;...
```

Finally we map all of the p1 tables, critically not mapping the first one as this ensures the pointer to 0 (AKA a null pointer) remains unmapped which will prevent undefined behaviour down the line

```nasm
  ;...

  mov ecx, 1 ; leave null page unmapped to prevent null pointer ref
.map_p1:
  mov eax, ecx
  shl eax, 12
  or eax, 0b11 
  mov [p1_tables + ecx * 8], eax 
  inc ecx
  cmp ecx, P1_COUNT * 512 
  jne .map_p1
  ret 
```

### Turning on paging

To get the cpu to use our created page tables we must 

1. Store the P4 address into CR3 (a special control register)
2. Enable Physical Address Extension (PAE) so we can use the full 64 bit address space 
3. Set long mode to enabled in EFER 
4. Enable the paging flag

```nasm
enable_paging:
  mov eax, p4_table ; laod P4 into cr3
  mov cr3, eax 
  
  mov eax, cr4 ; enable PAE flag physical address extention
  or eax, 1 << 5 
  mov cr4, eax 

  mov ecx, 0xC0000080 ; set long mode EFER MSR 
  rdmsr 
  or eax, 1 << 8 
  wrmsr
  
  mov eax, cr0 ; enable paging in the cr0 
  or eax, 1 << 31 
  mov cr0, eax 
  
  ret
```

## The Global Descriptor Table 

Finally the last requirement of long mode is the GDT which was a old segmentation register however paging is used instead of it nowadays, but it is still a requirement to enable 64 bit mode for ..... Legacy reasons ! 

Don't worry to much about it now we will explain it later on, all we are doing for now is creating one code segment to allow us to execute code. We use `.rodata` as it is readonly 

```nasm
section .rodata
gdt64: ; long mode gdt
    dq 0 ; zero entry
.code: equ $ - gdt64 ; new
    dq (1<<43) | (1<<44) | (1<<47) | (1<<53) ; code segm
.pointer:
    dw $ - gdt64 - 1
    dq gdt64
```

## Putting it all together

```nasm
global start
extern long_mode_start

P1_COUNT equ 4

section .text
bits 32
start:
    ; Setup temporary stack
    mov esp, boot_stack_top

    ; check required features 
    call check_multiboot
    call check_cpuid
    call check_long_mode

    ; configure long mode
    call set_up_page_tables
    call enable_paging

    ; load the GDT
    lgdt [gdt64.pointer]
    jmp gdt64.code:long_mode_start
    
    ; if something went wrong report it as L
    mov al, "L"
    jmp error
```

Now we create another file `longmode.asm` and define the `long_mode_start` method and have it print `OKAY` (which will require quad words only available in long mode)
```nasm
global long_mode_start

section .text
bits 64
long_mode_start:
    ; Clear legacy regs 
    mov ax, 0
    mov ss, ax
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    ; print `OKAY` to screen
    mov rax, 0x2f592f412f4b2f4f
    mov qword [0xb8000], rax
    hlt
```
Now you should be able to run the os and get OKAY printed to the screen 

CONGRATS WE ARE NOW IN LONG MODE!
 
In the next session we will be setting up rust and defining an interface for interacting with the VGA in rust 

# End of Post

If you have any questions, join our [discord](https://discord.gg/zKhj937xW2) and ask away!

`42 49 54 53 49 47 20 3C 33 20 59 4F 55`
