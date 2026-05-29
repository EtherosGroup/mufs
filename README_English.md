# Mount Unit File System

> **MUFS redefines filesystem organization: mount points are the primary naming primitive, not directory trees.**

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Spec](https://img.shields.io/badge/spec-v1.0-green.svg)](MUFS.md)

---

MUFS is a kernel-space filesystem framework where mount points — not directories — are the fundamental unit of storage organization. Partitions carry their own mount metadata in the superblock, eliminating external configuration files. System paths are resolved dynamically through a variable engine rather than being hardcoded at compile time.

---

## Key Innovations

| Concept | Traditional Filesystems | MUFS |
|---------|------------------------|------|
| **Organization** | Directory tree hierarchy | Typed mount points (`DMP` / `SMP` / `VMP`) |
| **Root semantics** | `/` contains files and directories | `/` is the device identity, non-writable |
| **Configuration** | External file (`/etc/fstab`) | Self-describing superblock extension |
| **System paths** | Hardcoded (`/tmp`, `/home`) | Dynamic variable mapping (`%temp%`, `%home%`) |
| **Device enumeration** | Static nodes + user-space daemon | `/devices` VMP — kernel-generated in real time |
| **Boot dependency** | initramfs → pivot_root → fstab | Single-phase kernel module initialization |

## Architecture

```
                   User-Space Applications
               MUFS unified namespace view
          /system     /data     /devices     ...

                ── Syscall Interface ──

                     MUFS Kernel Module
         ┌──────────────┼──────────────┐
     Mount Manager   Path Engine   Variable Engine
     Superblock I/O  VMP Generator  Hotplug Listener
         └──────────────┼──────────────┘

                   Virtual Filesystem (VFS)

                 Physical Filesystem Drivers
          ext4           exFAT           ...

                    Block Device Layer
```

## Mount Point Types

| Type | Full Name | Backed By | Created By |
|------|-----------|-----------|------------|
| **DMP** | Device Mount Point | Physical block device | User (`mount DMP ...`) |
| **SMP** | System Mount Point | Physical block device | Installer (`mount SMP ...`) |
| **VMP** | Virtual Mount Point | Kernel-generated content | System (`/devices` is mandatory) |

## Superblock Self-Description

Each partition declares its role directly in the ext4 superblock at offset `0x200`:

```
Byte 0    4    6    8                                               56
┌──────┬────┬────┬──────────────────────────────────────────────────┐
│ MAGIC│VER │FLAG│            mount name (UTF-8)                     │  ← Header (56 B)
├──────┴────┴────┴──────────────────────────────────────────────────┤
│ TYPE │PERM│  var_tag   │  var_target   │ CRC32 │    reserved      │  ← Body (104 B)
└──────┴────┴───────────┴──────────────┴──────┴────────────────────┘
                     160 bytes total  (ext4 reserves ~500 B)
```

This means:
- Move a drive to another machine → mount config moves with it
- No `/etc/fstab` to synchronize
- The partition and its role are an atomic unit

## Quick Start

### Format a system partition

```bash
mufs.mkfs \
    --type SMP \
    --name system \
    --var temp:temp/ \
    --var home:users/ \
    --var logs:logs/ \
    /dev/sda2
```

### Mount a data partition

```bash
mount DMP /devices/sda1 data     # → /data/
mount DMP /devices/sdb1 media    # → /media/
```

### Inspect the system

```bash
mufsctl list              # show all mount points
mufsctl info /system      # mount point details
mufsctl vars              # variable mapping table
mufsctl check /dev/sda1   # verify superblock integrity
mufsctl repair /dev/sda1  # recover from backup superblock
```

### Path resolution with variables

```
Application opens:   %temp%/file.log
Variable engine:     %temp% → /system/temp/
Resolved path:       /system/temp/file.log
Delegated to:        ext4 driver for "system" mount
```

## Documentation

- **[MUFS.md](MUFS.md)** — Full technical specification (v1.0)
- **[MUFS.md §4](MUFS.md#4-超级块扩展定义)** — Superblock extension layout
- **[MUFS.md §7](MUFS.md#7-系统变量机制)** — System variable engine
- **[MUFS.md §10](MUFS.md#10-命令行接口)** — CLI reference

## Requirements (Planned)

| ID | Feature | Priority |
|----|---------|----------|
| F1 | Mount point lifecycle management | P0 |
| F2 | Path resolution & normalization | P0 |
| F3 | DMP full implementation | P0 |
| F4 | SMP full implementation + installer tooling | P0 |
| F5 | VMP framework + `/devices` | P0 |
| F6 | System variable engine | P0 |
| F7 | ext4 superblock extension I/O | P0 |
| F8 | Superblock backup synchronization | P0 |
| F9 | Magic number detection & auto-discovery | P0 |
| F10 | Hotplug listener + `/devices` updates | P1 |
| F11 | CRC32 verification + auto-recovery | P0 |
| F12 | `mufsctl` user-space utility | P1 |
| F13 | `mufs.mkfs` formatting tool | P1 |

## License

This project is licensed under the **Apache License 2.0**. See [LICENSE](LICENSE) for details.

---

*MUFS is designed as the filesystem foundation for a next-generation open-source operating system.*
