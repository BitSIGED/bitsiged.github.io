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

We use a lot of material from Philipp Oppermann's blog *'Writing an OS in Rust (First Edition)'*; in fact, our workshop posts will effectively use a bunch of his materials as a baseline that we will build on with theory, further ideas and may diverge in some cases where we thought it useful. Please check out his blog [here](https://os.phil-opp.com/edition-1/); our work would be much harder without his content!

**Furthermore, the majority of the content of these workshops will be written assuming a Linux host machine in mind. We can assist with some differences in distros, but we'd ask that if you follow this guide with a macOS or Windows system, you should use a Linux VM (like WSL).**

# A minimal multiboot Rust Kernel (Phil Opp)

We'll be going into how to create a minimal x86 OS kernel with the Multiboot standard. For now, all we want is to get `OK` to print to the screen. But before we can get into doing any of that, we need to understand how a computer boots.

## Booting sequence

When a computer switches on, it loads the **BIOS** (Basic Input/Output System) from some special flash memory. The BIOS will run self-test and hardware initializations and then looks for bootable devices (HDDs, USBs, etc.). *In the case of B.O.C., this is usually a USB since B.O.C currently has no HDD or SSD!*. Once a bootbale device is found, the **bootloader** takes control which has to determine the location of the kernel image on the bootable device and load it into memory. The bootloader also needs to switch the CPU into **'protected mode'** as all x86 CPUs start in the limited 'real mode' by default (for old computers compatibility).
