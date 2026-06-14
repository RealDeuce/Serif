# Serif

Serif is the operating system for the NEC V20HL-based DreamWriter
portable word processor. It is written entirely in Nib, a compiled
language designed for the platform.

## Naming Convention

The project uses a calligraphy/typography metaphor throughout. The
nib is the writing tool; Serif is the system that gives the output
its form.

| Term | Meaning |
|------|---------|
| **Nib** | The language and compiler toolchain |
| **Serif** | The operating system |
| **Brief** | Single-segment executable (≤64K, like COM) |
| **Book** | Multi-segment executable |
| **Page** | 64K memory segment |
| **Shelf** | Directory |
| **Bookshelf** | Root directory / volume |
| **Spine** | IVT / system dispatch table |
| **Colophon** | Version and build metadata block |

File extensions follow the metaphor: `.brief` for single-segment
executables, `.book` for multi-segment binaries.

## Platform

- **CPU**: NEC V20HL (8086/80186-compatible with V20 extensions)
- **Memory**: 20-bit address space (1MB), segmented (64K per segment)
- **Language**: Nib (see `../nib/` for the toolchain)
- **Build**: Cross-compiled from FreeBSD/Linux/macOS using Nib toolchain

## Architecture

Serif is structured around device drivers with explicit API boundaries:

- The **reset vector** at FFFF:0000 performs platform setup (segments,
  stack, segment registers)
- Each device driver exposes a `pub` API surface with pinned registers
  and `clobbers()` declarations
- Driver `init` functions install interrupt handlers into the **spine**
  (IVT at 0000:0000) and populate a per-device function table
- Application code accesses drivers via software interrupts that
  dispatch through the function table
- The hardware monitor uses `.dbg` files to map addresses back to
  Nib source lines for debugging

## Coding Conventions

- All Nib source files use the Nib language spec (`../nib/SPEC.md`)
- API boundary functions use `in REG` parameter pins and `clobbers()`
- Driver-internal functions are module-private (no `pub`)
- Constants for port addresses and hardware registers use `const`
  with uppercase names
- The spine (IVT) is declared as `far[256] ivt at(0x0000:0x0000)`
  with `&handler` references
