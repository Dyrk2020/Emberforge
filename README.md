# Emberforge

An educational multi-architecture operating system written in Rust. Emberforge targets
`no_std` environments from the ground up: a UEFI bootloader loads a freestanding kernel,
which brings up the platform itself. It is built as a playground for understanding how
real operating systems work — from firmware handoff and ACPI parsing to journaled
filesystems and hardware virtualization — while keeping the whole stack in one
self-contained Rust workspace.

**Status:** x86_64 is the fully functional architecture. riscv64 is an early bring-up
skeleton; aarch64 and loongarch64 are placeholders.

## Workspace layout

| Crate     | Purpose                                                                  |
|-----------|--------------------------------------------------------------------------|
| `common`  | Shared definitions: the `BootInfo` structure passed across the boot boundary, framebuffer descriptors, and memory-region types |
| `loader`  | UEFI bootloader (x86_64): boots the OS kernel ELF or a Linux kernel via the EFI stub |
| `kernel`  | Freestanding kernel, built from a custom `x86_64-kernel.json` target spec |
| `ext4`    | A from-scratch ext4 filesystem implementation with a jbd2-compatible journal |
| `libs`    | Per-architecture configuration data (aarch64, riscv64, x86_64)          |

The kernel is built with `panic = "abort"`, a custom linker script
(`kernel/src/linker/`), and `-Z build-std` for `core`/`alloc` — no host libc anywhere.

## Boot flow (x86_64)

1. **UEFI firmware** → OVMF (QEMU) or real firmware loads `esp/efi/boot/bootx64.efi`.
2. **`loader`** draws a boot menu (arrow-key navigation with an auto-boot countdown),
   then either:
   - **OS kernel mode:** loads `esp/kernel-x86_64` (an ELF), allocates page tables before
     `ExitBootServices`, builds an identity + higher-half mapping, and hands over a
     `BootInfo` struct (memory map, framebuffer, physical-memory offset, RSDP address).
   - **Linux EFI stub mode:** installs a LoadFile2 protocol to supply an initrd, and boots
     a stock vmlinuz through its own EFI entry point.
3. **`kernel`** receives `BootInfo` through a naked entry stub (ABI-aligned stack), then
   initializes logging, console, GDT, interrupts, memory, and platform devices.

## x86_64 kernel features

Everything below lives in `kernel/src/arch/x86/` and is actually implemented:

- **Boot & console** — framebuffer console with an embedded bitmap font
  (`noto-sans-mono-bitmap`), 16550 UART serial output, QEMU `isa-debug-exit` support.
- **GDT/TSS** — full descriptor setup with stack switching.
- **Interrupts** — IDT with `x86-interrupt` handlers: page fault, double fault (IST),
  breakpoint, machine check, timer, keyboard, LAPIC error, and spurious vectors.
- **APIC** — Local APIC and I/O APIC initialization driven by ACPI (MADT).
- **ACPI** — table parsing via the `acpi`/`aml` crates; AML interpreter for DSDT.
- **Memory** — heap allocator (linked-list allocator), page allocator, direct physical
  memory map established by the bootloader.
- **PCIe** — ECAM/MMIO PCI configuration access with device enumeration and class-code
  decoding.
- **Scheduler** — cooperative round-robin thread scheduler with per-thread context
  switch (assembly save/restore), thread states, tick-driven preemption from the APIC
  timer.
- **Virtualization** — AMD-V (SVM) guest support under the `svm` feature: VMCB layout,
  nested-page-table root setup, host save area, IOPM/MSRPM setup, and `vmrun` entry
  (`svm.rs`, `vmcb.rs`). Intel VMX is scaffolded.
- **Bochs Graphics Adapter (BGA)** — VBE-style display mode setting.

## ext4 implementation highlights

The `ext4` crate is a complete read/write ext4 stack, written from the on-disk format up
(`ext4/src/`):

- **Journal (jbd2)** — full transaction machinery: descriptor blocks with UUID/escape/
  last-tag flags, commit records, revoke blocks, checkpointing, crash recovery
  (`journal/recovery.rs`), and the jbd2 superblock — exposed through a
  `Jbd2Journal` engine.
- **On-disk layout** — superblock (32-bit and 64-bit block counts, feature flags with
  `INCOMPAT_64BIT`/`EXTENTS`/`FLEX_BG`/`FILETYPE` handling), block group descriptors,
  ext4 extent trees with an extent walker and modifier, HTree indexed directories,
  directory entries, and metadata checksums.
- **Allocators** — block and inode allocation with per-group allocation state and
  bitmap helpers (find-first-zero, zero runs, clear/set/test).
- **Filesystem core** — path resolver with symlink-following (bounded depth), directory
  and file readers/writers, inode reader/writer, and a `VFS` trait layer so the kernel
  can mount it through a `BlockDevice` trait — the same interface works on bare metal
  and on the host.
- **Host tests** — `ext4/src/tests.rs` runs the entire stack on host Rust with an
  in-memory `MemoryBlockDevice`, including tests against a real ext4 disk image. This is
  the project's main regression net: on-disk format parsing, journal commits, and
  allocator behavior are all exercised with plain `cargo test` on the host.

## Building and running

Requirements: a recent **nightly** Rust toolchain (pinned in `rust-toolchain.toml`,
including `rust-src`), QEMU with OVMF, and Linux. Optional: `busybox` for the initramfs.

```sh
# Build the UEFI loader and the kernel, and stage them into ./esp
make

# Build a minimal busybox initramfs (initrd.img)
make initramfs

# Stage a stock Linux kernel + initrd onto the ESP (for Linux EFI stub boot)
make vmlinuz VMLINUZ_SRC=/boot/vmlinuz-$(uname -r)

# Boot the OS under QEMU with OVMF
make qemu

# Boot with a gdbstub for debugging (attach via lldb-qemu or gdb :1234)
make qemu-debug
```

The Makefile auto-detects the OVMF firmware paths for Arch and Debian-style
distributions; override `OVMF_CODE_PATH`/`OVMF_VARS_PATH` for anything else. Kernel
feature flags (e.g. `KERNEL_FEATURES=svm`) are forwarded to the `kernel` crate.
Log verbosity is controlled with `LOG_LEVEL` (default `DEBUG`).

## Roadmap

- **x86_64** — the reference implementation: UEFI boot, memory management, APIC/ACPI,
  scheduling, PCI(e), ext4 mounting, and SVM guests.
- **riscv64** — early bring-up: SBI console, logging, panic handler, QEMU exit device,
  and a linker script; the kernel builds but platform support is minimal.
- **aarch64 / loongarch64** — skeleton stubs and target/toolchain plumbing (including a
  custom UEFI target spec), awaiting real hardware bring-up.
- **Next steps** — userspace processes and syscalls, block-device drivers backed by
  virtio, VMX guest support to match SVM, and journaling integration across the ext4
  write path.

## License

MIT — see the crate manifests (`license = "MIT"`).