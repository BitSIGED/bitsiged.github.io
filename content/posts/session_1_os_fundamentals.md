+++
date = '2026-09-25T11:09:55+01:00'
draft = false
title = 'OS Fundamentals / Intro (Session 1)'
tags = ['Session', 'Operating Systems']
summary = "Content & Summary from Session 1 (24/9/2026)"
+++

# Intro

**Session 1** ran on Thursday, 24th September in AT7.14. The session covered the fundamentals of Operating Systems.
This session was ran by **Kacper** and **Archie**.

# Session Content + Resources
[Session Slides (Click Me!)](https://drive.google.com/file/d/1H28tDhoKsigcj7A427sFz2-y9x6v7J2v/view?usp=sharing)
Session slides were made with content generally available on the internet + some content from **Operating System Concepts 10th Edition (2018)** by **A.Silberschatz**.

### Call for Committee members, speakers & presentations
If you're someone who would like to help us run BitSIG, please let us know by pinging us in our Discord server (linked at the bottom of this post). Similarly, if you're
already very experienced in OS, low-level programming and other areas that BitSIG involves itself in and would like to give a talk or presentation about your research
and/or experience, please let us know! We wanna hear from you!

### Semester 1 Schedule
Meetings take place **every Thursday, 18:00-19:00 in AT 7.14**. We may sometimes run until 8pm if people are very interested, and otherwise we might go out to Teviot or some other social after the session.

We have an alternating schedule each week, where one week we'll run a discussion / theory session and then the next week will be a more practical workshop (e.g. code-writing).
Since this week is a theory / discussion session week, the next session will be a practical workshop. This scheduling gives us time to prepare workshop materials which require more effort than the theory sessions.

### What are Operating Systems?
- Many different definitions. People disagree on what
features should be included / packaged as part of an
‘OS’ and what features are actually user applications /
shipped separately [(see Microsoft vs US DoJ in 1998)](https://en.wikipedia.org/wiki/United_States_v._Microsoft_Corp.)

- Loose definition can be “...the one program running at
all times on the computer – usually called the **kernel**.”
[A.Silberschatz in OS Concepts 10e]

- The Operating System can be viewed as a **resource
allocator** (CPU, memory, I/O devices) or **control
program** (which manages the execution of user
programs)

Some examples of Operating Systems include MSDOS, macOS, Windows, Linux, iOS, Android. [See this xkcd comic](https://xkcd.com/1508/).

The structure of operating systems tends to revolve around a 'stack' approach, where applications communicate with the Kernel, and the Kernel acts as the bridge
between the applications and hardware like the CPU, Memory, I/O and other devices.

### Key components of a Computer System
- One or more CPUs
- Multiple device controllers connected through a shared system bus (Memory, CPU, USB, Graphics adapters -> monitors, etc.)
- Device controllers interface with the OS through 'device drivers' (which you'll know tons about if your GPU drivers ever broke). The drivers
provide the OS with a common interface to interact with device controllers.

### Interaction with Hardware
Let's dive a bit deeper:
- How does a resource let the CPU know when it needs computation?
- How do we know when said computation is finished, or, for example, when a user presses a key?

### Interrupts
The solution: Interrupts. They're called that because they 'interrupt' the CPU's work to tell it that another resource needs the CPU. For example,
a device controller for a keyboard may signal to the CPU that it requires processing, thus taking the CPU out of the work it was currently doing and
switching it to 'interrupt handling'. When the interrupt is handled, the CPU no longer needs to work on the 'interrupt' and the past state of the CPU is usually
restored.

### Context Switching & PCBs
Working with interrupts, we have the idea of **context switching**. This is where the OS switches which process is currently being executed / worked on by the CPU(s).
Note that context switching can also refer to when the CPU switches from operating in user mode to kernel mode (i.e. from restricted app code to privileged OS code).

When the context switches, we need a way of storing the *state* of the registers of the previous context so we can restore them once we want to switch back. Otherwise we'd
get lost or have to start over with processes that got interrupted, and that's not very efficient. We can store this information by using **PCBs** (Process Control Blocks, not Printed Circuit Boards) which are maintained by the Operating System.

### Kernel Space (don't touch)
Kernel space is the most highly-privileged region of memory in a Computer. This is where the Kernel (OS) manages system resources, CPU scheduling, memory
mapping + more.

User apps do not run in Kernel space and instead run in **User Space**; a crash in Kernel space can really screw things up (kernel panics: if the kernel
can’t trust its own state, it stops as that is usually the safer thing to do than still going).

### Kernel design & tradeoffs (monolithic, microkernels and hybrids)
**Monolithic Kernels:**
- More OS services in Kernel space (shared memory space) = faster (don't have to context switch often)
- A bug or crash in a driver or program in kernel space can crash the entire system
- Examples include LINUX & UNIX Operating Systems

**Microkernels:**
- More stuff in user space; a crash won't be as damaging and not cause a kernel panic.
- Slower because of frequent context switches + IPC (Inter-process communication)
- Examples include QNX, MINIX & GNU Hurd

**Hybrid Kernels:**
- Examples include Windows (NT) & macOS (XNU)
- These are more common and combine features of both monolithic kernels and microkernels - it's quite rare to see a purely monolithic kernel or purely microkernel.
- Usually a single address space (like monolithic)
- Subsystems organised in a modular fashion (i.e. messaging / communication between processes with IPC, similar to microkernels)

## Next time:
Practical workshop (bring laptop) on writing a minimal Rust Kernel (with Phil Oppermann’s blog).

Same time, same place next week!

We're going to Teviot after this session :-)

# End of Post

If you have any questions, join our [discord](https://discord.gg/zKhj937xW2) and ask away!

`42 49 54 53 49 47 20 3C 33 20 59 4F 55`