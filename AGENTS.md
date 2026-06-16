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
far32 jump tables for zero-overhead driver calls.

See [`docs/architecture.md`](docs/architecture.md) for the full
design: boot sequence, dispatch model, driver module structure,
hardware IRQ assignments, and init ordering.

### Emulator Testing

The Dreamulator emulates the DreamWriter T400 hardware. Binary path:

```
/usr/home/admin/T400/dreamulator/build/dreamulator
```

**Launching** (always clear NVRAM for clean state):

```sh
rm -f /home/admin/.fltk/dreamulator/dreamulator/serif.bin.nvram
/usr/home/admin/T400/dreamulator/build/dreamulator --remote 9999 --rom serif.bin
```

**Remote debug protocol** (connect via `printf "cmd\n" | nc -w 1 localhost 9999`):

| Command | Description |
|---------|-------------|
| `stop` | Pause execution |
| `run` | Resume execution |
| `reset` | Reset CPU to power-on state |
| `step` | Execute one instruction |
| `regs` | Dump all registers and flags |
| `status` | Show PC and run state |
| `dis SEG:OFF N` | Disassemble N instructions |
| `mem SEG:OFF N` | Dump N bytes of memory |
| `bp SEG:OFF` | Set breakpoint |
| `cbp SEG:OFF` | Clear breakpoint (by address, not index) |
| `lbp` | List active breakpoints |
| `key ROW BIT down/up` | Simulate keyboard matrix press/release |
| `io PORT` | Read I/O port |
| `wio SEG:OFF` | Write watchpoint on memory address |

**Standard debug workflow**: connect → `stop` → set breakpoints → `reset` → `run`.
Boot to HLT/REPL takes ~1 second when working correctly.

**NVRAM persistence**: The emulator saves RAM state between runs to
`~/.fltk/dreamulator/dreamulator/<romname>.nvram`. Always delete this
file before testing to avoid stale state from previous builds.

**Disassembly**: Prefer `nibdis` over the emulator's `dis` command for
inspecting generated code — it uses the map and debug files:

```sh
../nib/nibdis -m serif.map -d serif.dbg -l function_name serif.bin
```

### Debugging

The hardware monitor uses `.dbg` files to map addresses back to
Nib source lines for debugging.

## Building

Requires the Nib toolchain (`../nib/`). With `nib.build` in place:

```sh
../nib/nibbuild          # incremental build
../nib/nibbuild -f       # force rebuild all modules
```

See `../nib/USAGE.md` for full nibbuild options including
`--nib`, `--asm`, and `--bind` stage argument passthrough.

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
- Write-once locals should use `const` (initialized with `=`,
  never assigned with `:=`)
- Globals that live in RAM must be initialized in the driver's
  init function, not with initializers (which burn ROM space)
- Use `at()` with `end at;` for memory-mapped globals
- Use `fn bare` for functions that run before the stack exists
  (reset vector, boot entry). Call `frame_enter()` after setting
  up SS:SP if the function has spills, `frame_leave()` before exit.
- Interrupt handlers use `fn interrupt` (no vector number).
  The handler symbol is a far32 constant. Install into the IVT
  via `ivt_install(vector, handler)` from `ivt.nib`.
- Use `pushf(); cli(); ... popf();` to protect critical sections
  (e.g., ring buffer access shared between IRQ and mainline code)
