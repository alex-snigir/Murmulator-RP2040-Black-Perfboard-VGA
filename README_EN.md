*Read this document in Russian: [README.md](README.md)*

# Murmulator — RP2040 Black Clone, VGA Build on Perfboard

A Murmulator build (an adapter board from a Raspberry Pi Pico / YD-RP2040 clone to peripherals) with VGA output, a PS/2 keyboard, two joystick ports (with Bluetooth gamepad support via BlueRetro), audio output, and external power. The board is assembled on perfboard; the schematic is laid out in KiCad.

## Board Connectors

| Connector | Purpose |
|---|---|
| J1 | External power (PSU), DC jack |
| J2 | Raw +5V tap from the PSU (before D1) — for external peripherals (BlueRetro) |
| J3 | VGA (12-pin header, Conn_02x06) |
| J4 | Keyboard (PS/2, Mini-DIN-6) |
| J5 | External connector |
| J6 | Joystick 1 (DE-9) |
| J7 | Joystick 2 (DE-9) |
| J8 | Audio output |

## Documentation

- [`doc/murmulator-vga-project-summary_EN.md`](doc/murmulator-vga-project-summary_EN.md) — full build write-up (EN): SD module rework, VGA, PS/2 keyboard, joysticks and BlueRetro integration, audio, power, build checklist.
- [`doc/murmulator-vga-project-summary.md`](doc/murmulator-vga-project-summary.md) — the same document in Russian.

## Schematics (KiCad)

- [`schematics/rp2040_black_vga/`](schematics/rp2040_black_vga/) — KiCad project source files.
- [`schematics/rp2040_black_vga/schematics_rp2040_black_vga.pdf`](schematics/rp2040_black_vga/schematics_rp2040_black_vga.pdf) — schematic diagram in PDF.

## Key Build Features

- **SD module** reworked: the AMS1117 regulator and the 74VHCT125A buffer are desoldered and bridged with jumpers — the module becomes a clean 3.3V breakout with no signal-edge degradation at high SPI speeds.
- **VGA** — a passive resistor R2R ladder on GPIO pins, no separate power supply needed.
- **PS/2 keyboard** — powered from Vout (downstream of the BAT54C diode); the CLK/DATA signal lines are level-shifted 5V → 3.3V via a resistor/zener-clamp network.
- **Joysticks (Dendy-type, DE-9)** — the shift register is powered from 3.3V, which removes the need for a protective resistor on DATA.
- **BlueRetro** — a DIY ESP32-based adapter emulates both joystick ports over Bluetooth HID (PS3/4/5, Xbox, Wii/Switch, and others); the CLOCK/LATCH/DATA protocol is identical to a physical Dendy joystick. Requires an external PSU to be connected. 3.3V-logic build of the adapter: https://github.com/alex-snigir/BlueRetro-3.3V-logic
- **Audio output** — stereo (L/R) channels plus a mixed-in beeper signal (BEEP_OUT).
- **Power** — Schottky diode D1 (1N5819) backs up the path from the PSU to the +5V_VOUT node, bypassing the low-current onboard BAT54C.

See [`doc/murmulator-vga-project-summary_EN.md`](doc/murmulator-vga-project-summary_EN.md) for full details on every point above.

## Links

- Official documentation and other Murmulator build variants: https://murmulator.ru/howto
- Classic schematic (reference): https://github.com/AlexEkb4ever/MURMULATOR_classical_scheme
- Base version "Murmulator on a 7×9 perfboard with VGA": https://murmulator.ru/mm-maket
- BlueRetro 3.3V-logic adapter (joystick port integration): https://github.com/alex-snigir/BlueRetro-3.3V-logic
