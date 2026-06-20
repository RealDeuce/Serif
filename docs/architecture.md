# Serif Architecture

## Memory Layout

### Bank Configuration (set by reset vector)

```
Bank 0  Port 0x10 = 0x10   00000-1FFFF  RAM
Bank 1  Port 0x11 = 0x11   20000-3FFFF  RAM (if present)
Banks 2-5                   40000-BFFFF  unconfigured at boot
Bank 6  Port 0x16 = 0x01   C0000-DFFFF  ROM bank 1
Bank 7  Port 0x17 = 0x00   E0000-FFFFF  ROM bank 0
```

### RAM Map (bank 0, T400 target)

```
0x0000-0x03FF   IVT (spine) — 256 far pointers
0x0400-0x0FFF   Spine registry, driver state, globals
0x1000-0x1FFF   LCD framebuffer (64 rows × 64 bytes/row)
0x2000-0x3F9F   Stack (8096 bytes, grows downward from SP=0x3FA0)
```

Each T400 LCD scan row reserves 64 bytes: 60 visible framebuffer
bytes followed by 4 currently-unused bytes. `fb_clear()` clears only
the visible 60-byte span so those row trailers remain available for
future metadata such as console cell attributes.

LCD buffer size is model-dependent: T200/T500 use 128 rows (8KB at
0x1000–0x2FFF), pushing the stack to 0x3000–0x4F9F. Compile-time
target selection is planned once Nib supports it.

## Boot Sequence

The reset vector at FFFF:0000 performs the following steps:

1. **Platform setup** — initialize segment registers, set up the
   stack, configure memory banking
2. **Fault vector fill** — write a pointer to an INT 5 (invalid /
   bounds) stub into all 256 IVT entries, so any unexpected vector
   fires a clean fault rather than jumping into uninitialized memory
3. **Service locator** — install the INT 0x21 handler into the IVT
4. **Driver init** — call each driver's `pub fn init()` in
   dependency order; each init configures its hardware, overwrites
   the IVT entries it owns with real IRQ handlers, and registers its
   jump table with the service locator

### Init Order

Some drivers depend on others being initialized first:

1. Shared gate-array latches and IRQ controller
2. Display: LCD base selector, framebuffer, then console
3. Keyboard
4. Timer
5. Buzzer
6. Serial, Parallel
7. Power management
8. RTC
9. Storage (built-in, PCMCIA, FDC)

The exact order may evolve, but the constraint is: a driver must not
depend on hardware whose driver hasn't initialized yet.

## Shared Gate-Array Resources

Some gate-array ports contain unrelated hardware controls and must be
owned centrally rather than shadowed independently by each driver:

| Port | Owner | Purpose |
|------|-------|---------|
| 0x30 | `control.nib` | Write-only control latch shadow |
| 0x60 | `irq.nib` | Active-low IRQ mask shadow |
| 0x90 | `irq.nib` | IRQ pending-source clear writes |
| 0xA0 | `status.nib` | Decoded read-only platform status |

Drivers call these internal modules directly. They are not public
service-locator drivers and do not expose jump tables. This keeps
shared latch state consistent while preserving each device driver's
ownership of its own hardware behavior.

## Driver Dispatch

### Service Locator (INT 0x21)

The only software interrupt in Serif's public API. Applications call
it once during program init to obtain a driver's jump table.

**Calling convention:**

| Register | Direction | Purpose |
|----------|-----------|---------|
| AH | in | Device ID (`DEV_*` from `spine.nib`) |
| AL | in | Requested API version (1-based) |
| ES:BX | out | Far pointer to driver jump table (on success) |
| CF | out | 0 = success, 1 = failure |
| AH | out | Error code on failure (`ERR_*` from `spine.nib`) |

Version matching is strict: if the app requests version 2 and the
driver only provides version 1, the call fails with ERR_NO_VERSION.
This forces apps to explicitly handle API evolution rather than
silently using an incompatible table.

### Jump Tables

Each driver exposes a dense `far32[]` array of entry points. The
service locator returns a pointer to this array.

Function constants are **pre-multiplied byte offsets** (0x00, 0x04,
0x08, …) so the hot-path call requires no runtime arithmetic:

```
// One-time lookup (program init):
//   AH = DEV_KBD, AL = 1
//   int 0x21
//   mov [kbd_table], BX
//   mov [kbd_table+2], ES
//
// Hot-path call (runtime):
//   les BX, [kbd_table]
//   bound BX_offset, [kbd_bounds]   // validate (INT 5 on fail)
//   call far [ES:BX + KBD_READ]    // one instruction, no math
```

### Bounds Checking

Callers validate function offsets with the 80186 `bound` instruction
against a min/max pair. Out-of-range offsets fire INT 5 through the
IVT. This catches invalid function numbers without any conditional
branch in the call path.

### Driver Registration

During init, each driver calls an internal registration function to
add itself to the service locator's registry. The registry is a
fixed-size array (MAX_DRIVERS = 16 entries, 8 bytes each):

| Offset | Size | Field |
|--------|------|-------|
| 0 | u8 | Device ID |
| 1 | u8 | API version |
| 2 | far | Jump table pointer (4 bytes) |
| 6 | u16 | Table size in bytes (BOUND max value) |

## Driver Module Structure

Each driver is a Nib module with a consistent shape:

```
// example_driver.nib

use "spine.nif";
use "platform.nif";

// --- Function offsets (pre-multiplied) ---
pub const EXAMPLE_FUNC_A = 0x00;
pub const EXAMPLE_FUNC_B = 0x04;
pub const EXAMPLE_FUNC_C = 0x08;

// --- Module-private state ---
u8[64] buffer;
u16 flags;

// --- Jump table ---
pub far32[3] example_table = {@func_a, @func_b, @func_c};

// --- API functions ---
// Each declares in REG parameter pins, ds(...), and clobbers()

fn api far func_a(arg: u8 in AL) ds(0x0000) clobbers(FLAGS) { ... }
fn api far func_b() ds(none) clobbers(FLAGS) { ... }
fn api far func_c() ds(caller) clobbers(FLAGS) { ... }

// --- IRQ handlers (if applicable) ---
fn interrupt irq_handler() ds(0x0000) { ... }

// --- Init (called by boot, not in jump table) ---
pub fn init() {
    // 1. Configure hardware (ports, etc.)
    // 2. Populate jump table
    // 3. Install IRQ handler(s) into IVT
    // 4. Register with service locator
}
```

Key points:

- **`init` is not in the jump table.** It is a regular `pub fn`
  called once by the boot sequence. Applications never call it.
- **API functions are module-private.** They are reached only
  through the jump table, not by direct `pub` calls.
- **Far API functions declare a DS policy.** Use `fn api far` for
  table-only driver entries. Use `ds(0x0000)` for APIs that touch
  Serif RAM globals, `ds(none)` for DS-free bodies, and `ds(caller)`
  when caller DS is intentionally used.
- **IRQ handlers are module-private.** They are installed into the
  IVT by init, not exposed to applications.
- **State is module-private.** Drivers do not share state directly.

## Hardware IRQs

The gate array routes hardware interrupts to vectors F8–FF. These
are real interrupt handlers in the IVT, separate from the jump-table
API surface.

| Vector | IRQ Source | Typical Owner |
|--------|-----------|---------------|
| F8 | Power / timer wake | Power driver |
| F9 | One-shot timer (PORT_TIMER) | Timer driver |
| FA | Keyboard scan cycle | Keyboard driver |
| FB | Keyboard row scan | Keyboard driver |
| FC | USART receive | Serial driver |
| FD | USART transmit ready | Serial driver |
| FE | Centronics ACK | Parallel driver |
| FF | Power / warm wake | Power driver |

Driver init installs handlers into these vectors and configures the
IRQ mask (PORT_IRQ_MASK at 0x60) to enable the relevant sources.

The TC8521/RP5C01 RTC alarm output is not wired to a Serif IRQ in the
current driver model. MAME exposes an alarm callback on the chip device
but does not connect it in the Nakajima board driver, and Dreamulator
does not currently assert an RTC alarm IRQ. Serif therefore exposes RTC
alarm register programming and software match checks, not interrupt
delivery.

## Source Layout

```
src/
  reset.nib       Reset vector and boot sequence
  fault.nib       INT 5 default fault handler (HLT loop)
  lookup.nib      INT 0x21 service locator and driver registry
  platform.nib    Gate array and V20 CPU constants
  spine.nib       Service locator, device IDs, dispatch constants
  control.nib     Shared gate-array control latch owner
  irq.nib         Shared gate-array IRQ mask/clear owner
  status.nib      Shared gate-array status helpers
  uart.nib        8251 USART serial driver
  rtc.nib         TC8521AP RTC driver
  timer.nib       Gate array one-shot timer driver
  fdc.nib         N82077AA FDC constants (T200/T500)
  lcd.nib         LCD hardware base-select driver
  fb.nib          Public 1bpp framebuffer/blitter driver
  console.nib     Public 80x8 text console driver
  font.nib        8×8 character font (from DreamWriter ROM)
  buzzer.nib      Tone generator driver
  parallel.nib    Centronics parallel port driver
  power.nib       Power management driver
```
