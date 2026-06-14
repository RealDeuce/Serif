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

Serif is structured around device drivers with explicit API
boundaries, using a service-locator dispatch model (INT 0x21) with
pre-multiplied jump tables for zero-overhead driver calls.

See [`docs/architecture.md`](docs/architecture.md) for the full
design: boot sequence, dispatch model, driver module structure,
hardware IRQ assignments, and init ordering.

### Debugging

The hardware monitor uses `.dbg` files to map addresses back to
Nib source lines for debugging.

## Building

Requires the Nib toolchain (`../nib/`). With `nib.build` in place:

```
nibbuild          # builds serif.bin
```

## Coding Conventions

- All Nib source files follow the Nib language spec (`../nib/SPEC.md`)
- Let the register allocator do its job — avoid pinning variables
  to registers except at API boundaries and before inline asm
- Use `:=` for assignment to predefined register globals (e.g.,
  `ES := src_seg;`, not `seg ES = src_seg;`)
- Driver-internal functions are module-private (no `pub`)
- Driver init functions are named `<driver>_init()` to avoid
  collision in the flat namespace after `use`
- Constants use `const` with `UPPER_CASE` names
- Globals that live in RAM must be initialized in the driver's
  init function, not with initializers (which burn ROM space)
- Use `at()` with `end at;` for memory-mapped globals
