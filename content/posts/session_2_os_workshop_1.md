+++
date = '2026-10-01T12:46:55+01:00'
draft = true
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

**Furthermore, the majority of the content of these workshops will be written assuming a Linux host machine in mind. We can assist with some differences in distros, but we'd ask that if you follow this guide with a macOS or Windows system, you should use a Linux VM (like WSL).**

# A minimal multiboot Rust Kernel (Phil Opp)

We'll be going into how to create a minimal x86 OS kernel with the Multiboot standard. For now, all we want is to get `OK` to print to the screen. But before we can get into doing any of that, we need to understand how a computer boots.

## Booting sequence

When a computer switches on, it loads the **BIOS** (Basic Input/Output System) from some special flash memory. The BIOS will run self-test and hardware initializations and then looks for bootable devices (HDDs, USBs, etc.). *In the case of B.O.C., this is usually a USB since B.O.C currently has no HDD or SSD!*. Once a bootbale device is found, the **bootloader** takes control which has to determine the location of the kernel image on the bootable device and load it into memory. The bootloader also needs to switch the CPU into **'protected mode'** as all x86 CPUs start in the limited **'real mode'** by default (for old computers compatibility). To give you an idea, real mode generally has access to about 1MB of physical RAM because of how it does addressing, while protected mode can do up to 4GB in 32-bit addressing (and more in 64 bit via extension).

So, our current task is to get an already-existing bootloader to boot our kernel, and that's where **Multiboot** and **GRUB 2** comes in!

## Multiboot
There exists a bootloader standard known as the **Multiboot Specification**. Our kernel will need to indicate that it supports Multiboot and then every Multiboot-compliant bootloader can boot it. We can pair Multiboot 2 in particular with the GRUB 2 bootloader to achieve all the booting needs we may have!

Our kernel needs to tell the bootloader that Multiboot 2 is supported, which we'll do with the *Multiboot Header*. This is effectively some boilerplate magic, but we'll try to explain it as much as we can. Take a look at the x86 assembly file `multiboot_header.asm` below.

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

Definitely scary for people who don't know x86 assembly, so we'll build on the quick guide from the Phil Opp blog here and go over the basics of assembly.

**ASSEMBLY SECTION**

Anyhow, we can use the `nasm` command to assemble `multiboot_header.asm`. If we hexdump the assembled flat binary, we can actually spot a good-to-be-aware-of feature of x86: **Little Endian**.

**Endianness**

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

## Building the executable
Okay, great, we've got the boot code and the multiboot header which will tell GRUB that our kernel supports Multiboot 2. But GRUB will need something called an **ELF** (Executable and Linkable Format)  Executable, and our files to be **ELF** object files, which we can get `nasm` to create by passing the `-f elf64` flag when using `nasm` to assemble our code.

Then, the **ELF** executable is created by *linking* the **ELF** object files together. To do this, we use a custom linking script called `linker.ld`:

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

Then save the Makefile with this content:
```make
arch ?= x86_64
kernel := build/kernel-$(arch).bin
iso := build/os-$(arch).iso

linker_script := src/arch/$(arch)/linker.ld
grub_cfg := src/arch/$(arch)/grub.cfg
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
	@grub-mkrescue -o $(iso) build/isofiles 2> /dev/null
	@rm -r build/isofiles

$(kernel): $(assembly_object_files) $(linker_script)
	@ld -n -T $(linker_script) -o $(kernel) $(assembly_object_files)

# compile assembly files
build/arch/$(arch)/%.o: src/arch/$(arch)/%.asm
	@mkdir -p $(shell dirname $@)
	@nasm -felf64 $< -o $@
```

Note that you'll have to make changes to this default Makefile depending on your system (for example, recall that Fedora uses `grub2-mkrescue`, so you'll have to update any mentions of `grub-mkrescue` in the Makefile).