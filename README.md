# Serif

Serif is an operating system for the NEC V20HL-based DreamWriter
family of portable word processors. It is written entirely in
[Nib](https://github.com/RealDeuce/Nib), a compiled systems language
designed for the platform.

## What is a DreamWriter?

The DreamWriter is a line of dedicated word processing machines built
around the NEC V20HL processor (8086/80186-compatible) with a custom
gate array (LC92040D-791) handling LCD display, keyboard scanning,
memory banking, and interrupt routing. Models include the T400
(480x64 LCD), T200, and T500 (480x128 LCD). They were sold under
various brand names including NTS DreamWriter and Dator 3000.

## What is Serif?

Serif replaces the DreamWriter's factory ROM with a from-scratch
operating system. It provides:

- **Driver framework** with a service-locator dispatch model
  (INT 0x21) and pre-multiplied jump tables for zero-overhead
  driver calls
- **LCD driver** with pixel-level blit operations, an 80x8 text
  console, and cursor support
- **Device drivers** for the LCD, keyboard, buzzer, timer, UART,
  RTC, parallel port, and power management, with storage planned
- **Platform constants** for the V20 CPU, gate array, and all
  on-board peripherals (USART, RTC, FDC)

## Building

Serif is cross-compiled using the Nib toolchain. Install
[Nib](https://github.com/RealDeuce/Nib), then:

```
nibbuild
```

This produces `serif.bin`, a 1MB ROM image ready to load in an
emulator or burn to flash.

## Testing

Serif can be tested with
[Dreamulator](https://github.com/RealDeuce/Dreamulator):

```
dreamulator --model dwT400 --rom serif.bin --remote 9999
```

The `--remote` flag enables a TCP debug console with register
inspection, memory dumps, disassembly, breakpoints, and
single-stepping.

## Project Structure

```
src/
  reset.nib       Boot sequence and reset vector
  lcd.nib         LCD display driver
  font.nib        8x8 character font (from DreamWriter ROM)
  buzzer.nib      Tone generator driver
  parallel.nib    Centronics parallel port driver
  power.nib       Power management driver
  lookup.nib      INT 0x21 service locator
  fault.nib       Default fault handler
  platform.nib    Gate array and V20 CPU constants
  spine.nib       Dispatch model constants
  control.nib     Shared gate-array control latch owner
  irq.nib         Shared gate-array IRQ mask/clear owner
  status.nib      Shared gate-array status helpers
  uart.nib        8251 USART serial driver
  rtc.nib         TC8521AP RTC driver
  timer.nib       Gate array one-shot timer driver
  fdc.nib         N82077AA FDC constants
docs/
  architecture.md Driver dispatch model and memory layout
```

## Naming

The project uses a calligraphy/typography metaphor: the
[Nib](https://github.com/RealDeuce/Nib) is the writing tool; Serif
is the system that gives the output its form.

## License

TBD
