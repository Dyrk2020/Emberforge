# Freestanding

An educational multi-architecture operating system written in Rust. `no_std` from the ground up: a UEFI bootloader loads a freestanding kernel, which brings up the platform itself — from firmware handoff and ACPI parsing to a journaled filesystem and hardware virtualization, all in one self-contained workspace.

**Status:** x86_64 fully functional · riscv64 early bring-up skeleton · aarch64 / loongarch64 placeholders.

## Workspace

| Crate | Purpose |
|---|---|
| `common` | shared definitions: `BootInfo` passed across the boot boundary, framebuffer descriptors, memory-region types |
| `loader` | UEFI bootloader (x86_64): boots the OS kernel ELF or a Linux kernel via the EFI stub; boot menu, kernel config |
| `kernel` | freestanding kernel, custom `x86_64-kernel.json` target spec |
| `ext4` | from-scratch ext4 filesystem with a jbd2-compatible journal |
| `libs` | per-architecture configuration data (aarch64, riscv64, x86_64) |

## What the kernel covers

- x86_64 paging, kernel heap, interrupt plumbing (IDT + `x86_64` crate + inline asm)
- APIC timer, ACPI table walk, PCIe enumeration
- scheduler, process abstraction, context switch
- hardware virtualization scaffolding: AMD SVM (546-line context + VMCB) and Intel VMX skeletons
- riscv64 bring-up files (11 modules) starting to grow

## Build & run

Nightly Rust with `-Z build-std=core,alloc` (no host libc anywhere), `panic = "abort"`, custom linker scripts.

```bash
make            # builds loader.efi + kernel
make run        # QEMU with OVMF (paths auto-detected for arch/debian)
```

OVMF firmware is expected under `/usr/share/OVMF/` (distro-dependent paths handled in the `Makefile`).

`common::BootInfo` is the contract across the boot boundary: framebuffer info + physical memory regions; the loader maps the kernel ELF, hands off, and the kernel never depends on firmware again.
