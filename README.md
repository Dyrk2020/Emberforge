# Freestanding

An educational multi-architecture operating system written in Rust. Freestanding targets
`no_std` from the ground up: a UEFI bootloader loads a freestanding kernel, which brings up
the platform itself — a playground for understanding how real operating systems work, from
firmware handoff and ACPI parsing to journaled filesystems and hardware virtualization, all
in one self-contained workspace.

**Status:** x86_64 is fully functional; riscv64 is an early bring-up skeleton; aarch64 and
loongarch64 are placeholders.

## Workspace

| Crate    | Purpose                                                                                     |
|----------|---------------------------------------------------------------------------------------------|
| `common` | Shared `BootInfo` (memory map, framebuffer, physical-memory offset, RSDP) and memory-region types |
| `loader` | UEFI bootloader (x86_64): boots the OS kernel ELF or a Linux kernel via its EFI stub        |
| `kernel` | Freestanding kernel, built from a custom `x86_64-kernel.json` target spec                    |
| `ext4`   | From-scratch ext4 filesystem implementation with a jbd2-compatible journal                   |
| `libs`   | Per-architecture configuration data (aarch64, riscv64, x86_64)                               |

Built with `panic = "abort"`, a custom linker script (`kernel/src/linker/`), and
`-Z build-std` for `core`/`alloc` — no host libc anywhere.

## Highlights

- **UEFI boot (x86_64)** — `loader` draws a boot menu (arrow keys + auto-boot countdown),
  loads `esp/kernel-x86_64`, builds identity + higher-half page tables before
  `ExitBootServices`, and hands over `BootInfo`; can also boot a stock vmlinuz through the
  Linux EFI stub with a LoadFile2 initrd.
- **Kernel (x86_64)** — framebuffer console with embedded bitmap font, GDT/TSS, IDT with
  `x86-interrupt` handlers (page fault, double fault via IST, timer, keyboard, machine
  check), LAPIC/I/O APIC driven by ACPI (MADT), heap + page allocators, PCIe (ECAM) device
  enumeration, and a cooperative round-robin scheduler with per-thread assembly context
  switches and APIC-timer preemption.
- **Virtualization** — AMD-V (SVM) guest support under the `svm` feature: VMCB layout,
  nested-page-table root, host save area, IOPM/MSRPM, `vmrun` entry. Intel VMX scaffolded.
- **ext4** — complete read/write stack written from the on-disk format up: jbd2 journal
  (commit records, revoke blocks, checkpointing, crash recovery), 64-bit + extents +
  flex_bg superblock features, extent trees, HTree indexed directories, block/inode
  allocators, symlink-aware path resolution, and a `VFS`/`BlockDevice` trait layer usable
  on bare metal and the host.
- **Host-tested** — the ext4 stack runs on host Rust against an in-memory
  `MemoryBlockDevice` (`ext4/src/tests.rs`), including tests on a real ext4 disk image;
  plain `cargo test` on the host is the project's main regression net.

## Quick start

Requires nightly Rust (pinned in `rust-toolchain.toml`, with `rust-src`), QEMU with OVMF,
and Linux; optional `busybox` for the initramfs.

```sh
make                # build loader + kernel, stage into ./esp
make initramfs      # minimal busybox initramfs (initrd.img)
make vmlinuz VMLINUZ_SRC=/boot/vmlinuz-$(uname -r)  # stage Linux for EFI-stub boot
make qemu           # boot the OS under QEMU with OVMF
make qemu-debug     # boot with a gdbstub (attach via lldb-qemu or gdb :1234)
```

The Makefile auto-detects OVMF firmware paths on Arch and Debian-style distros; override
`OVMF_CODE_PATH`/`OVMF_VARS_PATH` otherwise. Kernel features forward via
`KERNEL_FEATURES=svm`; log verbosity via `LOG_LEVEL` (default `DEBUG`).

## Roadmap

- **x86_64** — reference implementation: UEFI boot, memory management, APIC/ACPI,
  scheduling, PCI(e), ext4 mounting, SVM guests.
- **riscv64** — early bring-up: SBI console, logging, panic handler, QEMU exit device;
  builds, but platform support is minimal.
- **aarch64 / loongarch64** — skeleton stubs and target/toolchain plumbing (custom UEFI
  target spec), awaiting real hardware bring-up.
- **Next** — userspace processes and syscalls, virtio-backed block drivers, VMX guests to
  match SVM, journaling integration across the ext4 write path.

## License

MIT — see the crate manifests (`license = "MIT"`).
