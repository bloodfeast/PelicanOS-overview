# PelicanOS

A thin microkernel for running sandboxed programs, written in Rust with no
external crates. It boots on **aarch64, x86_64 and RISC-V**.

> **The source is private.** PelicanOS is proprietary (copyright (c) 2026
> Brandon White, all rights reserved). This repository describes what it
> is, how it is built and what it measures. Source access for interviews
> and evaluations is available on request through my GitHub profile.

## The rule

**The OS stays out of the way.** It multiplexes the hardware, keeps programs
from trampling each other, and does nothing else. When a design question
comes up, the answer is whichever option makes the kernel do less, decide
less, and hide less.

What follows from it:

- **Sandboxed by construction.** A program starts with only the handles its
  parent gives it. There is no filesystem in the kernel, no global
  namespace, and nothing to install.
- **Batch everything.** Programs and the small user-space servers they
  share things through talk over shared-memory rings: write N requests,
  notify once, get N completions. Bulk data stays in a pool inside the ring,
  so no system call per buffer and no page moved.
- **Drivers are programs.** A program that owns a device drives it itself:
  its registers, its DMA memory and its interrupt are the program's. Disk
  and NIC drivers are a library.
- **No hidden work.** No page cache, swap, overcommit, background kernel
  threads or migration a program didn't ask for. Mapped memory is committed
  memory, and a syscall does what it says and nothing else.
- **A build choice, not a knob.** What a system carries is chosen at compile
  time: no config files, no module loading, no driver probing.
- **Faster than Linux, or it's a bug.** Every hot path has a benchmark with a
  Linux counterpart, measured on the same machine under the same hypervisor.

## What runs on it

- **Rust's `std`** and **unmodified tokio applications**, through a port of
  the language runtime onto the kernel's calls, with TCP, UDP and DNS from a
  user-space socket server.
- **Its own storage**: a key-value store that drives the disk and a NIC
  itself, with a log-structured on-disk format cleaned while it serves.
  On top of it: named objects, `std::fs`, and **SQLite**, where a commit is
  one disk write. The same requests can be served from an S3-compatible
  bucket over TLS.
- **Network services**: a web server on its own TCP stack and NIC, and a
  load balancer that forwards connections by rewriting addresses.
- **A shell** on the serial console or a screen that runs programs by name,
  from the boot image or from storage, onto a running machine.
- **A model, walled in**: an LLM inference server split into programs whose
  walls the kernel enforces. The model holds no network handle, the network
  side can't reach the weights, every request passes an audited gate, and
  the weights are sealed on disk.

## Measured against Linux

Medians of five interleaved rounds against Linux's best of three passes
(stock, `mitigations=off`, `idle=poll`) under the same hypervisor on the
same machine. At v0.29.0: **ahead on all 65 rows under QEMU** (EPYC 7763)
and **on 62 of 65 under Firecracker** (EPYC 9V45). A selection, QEMU, EPYC 7763:

| Row | PelicanOS | Linux |
|---|---|---|
| Key-value GET over a ring, one at a time | 0.90 us | 7.2 us |
| The same, beside 200 threads spinning on its core | 0.51 us | 1747 us |
| Word ping-pong between two cores (futex's job) | 0.13 us | 5.4 us |
| Timer wakeup from idle, 100 us deadline | 2.3 us | 50.6 us |
| SQLite INSERT, one row per transaction, durable | 218 us | 960 us |
| SQLite SELECT by key | 0.92 us | 1.40 us |
| tokio HTTP, a connection per GET, 16 at a time | 76 us | 323 us |
| tokio HTTP, 1 KiB GETs over 32 open connections | 17.5 us | 125 us |

Behind under Firecracker: the TLB shootdown (570 against 288 us, on a host
with slow cross-core interrupts), SQLite at 100 rows a transaction (129
against 105 us) and the balancer's 64 KiB file (629 against 612 us).
These are system paths inside virtual machines; model inference is CPU
arithmetic that no OS speeds up, and it has not been compared yet.

## How claims are checked

Every claim is either measured or checked by a self-test that runs on every
boot on all three architectures, and each test has a **mutant**: the system
broken on purpose to show the test fails when the claim stops holding.
A hostile program tries, on every boot, to see what one program could learn
of another (registers across calls and switches, fresh threads, fresh
memory, wrong calls). What the system does **not** do is written down next
to what it does: it is tested, not formally verified; it runs on CPUs in
virtual machines, not yet bare metal; and outside a confidential VM the host
can read guest memory.

## Platforms

| Architecture | Boot | Interrupts, timer | Paging |
|---|---|---|---|
| aarch64 | arm64 Image (QEMU, U-Boot) | GICv3, generic timer | 4-level, 4 KiB |
| x86_64 | PVH (QEMU microvm, Firecracker, Xen), UEFI | x2APIC, TSC-deadline | 4-level |
| riscv64 | OpenSBI | SBI IPI, Sstc, PLIC | Sv39 |

Tested under QEMU, under KVM on x86_64, and under HVF on Apple Silicon. It
boots as shipped under Firecracker.

## Size and history

About **46,000 lines of Rust**, tests included: the kernel is about 17,000.
Small enough for one person to hold whole is a requirement, checked every
release. Developed in 44 tagged releases, each with a changelog of what
changed, why, and what it measured:

- v0.1 to v0.3: boot on three architectures, cores and time, physical memory, a lock-free allocator, programs and threads
- v0.4 to v0.5: rings and pooled buffers, handles and events, programs driving hardware
- v0.6 to v0.12: the storage server, a shell, programs from storage, a durable log for other machines
- v0.15 to v0.21: a web server, a load balancer, faults that end only the faulting program, measurement where it would run
- v0.22 to v0.29: memory programs ask for, a screen, Rust's `std`, sockets and tokio, files and SQLite, the user's bucket, an isolated model server

## Contact

Brandon White, through [my GitHub profile](https://github.com/bloodfeast) or LinkedIn.
