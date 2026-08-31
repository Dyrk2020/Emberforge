# Freestanding

An educational multi-architecture operating system written in Rust: a UEFI bootloader launches a freestanding (`no_std`) kernel that brings up the platform itself - a single self-contained workspace and playground for how real operating systems work.

**Arch status:** x86_64 fully functional; riscv64 early bring-up skeleton; aarch64 and loongarch64 placeholders.

**Workspace crates:**
- `common` - shared `BootInfo` (memory map, framebuffer, physical-memory offset, RSDP) and memory-region types
- `loader` - UEFI bootloader (x86_64): boots the OS kernel ELF, or a Linux kernel via its EFI stub
- `kernel` - the freestanding kernel, built from a custom `x86_64-kernel.json` target spec
- `ext4` - from-scratch ext4 filesystem implementation with a jbd2-compatible journal (host-testable)
- `libs` - per-architecture configuration data (aarch64, riscv64, x86_64)

MIT - see the crate manifests.
