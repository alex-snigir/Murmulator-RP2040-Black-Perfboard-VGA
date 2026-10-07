# Building a VGA Murmulator on Perfboard

**Document version:** 1.0 · **Date:** 2026-10-03

## 1. General Concept

Murmulator is an adapter board from the Raspberry Pi Pico to peripherals (SD card, VGA, PS/2 keyboard, joystick, audio loading).

**Connector labels on the board:**

| Connector | Purpose |
|---|---|
| J1 | External power (PSU), DC Jack → `+5V_RAW` rail |
| J2 | Tap of the `+5V_RAW` rail — for external peripherals, e.g. BlueRetro |
| J3 | VGA (12-pin header, Conn_02x06) |
| J4 | Keyboard (PS/2, Mini-DIN-6) |
| J5 | External connector |
| J6 | Joystick 1 (DE-9) |
| J7 | Joystick 2 (DE-9) |
| J8 | Audio out |
| J9 | Audio in |
| J10 | `+3V3` tap (JST-XH, 2 pins: GND, +3V3) |
| J11 | Additional tap of the `+5V_VOUT` node ("Power 5V Out") — for external peripherals powered after the diode-OR (the same node as keyboard J4) |
| J12 | Second power input from USB (Micro USB module) instead of the DC jack → via D1 to the `+5V_RAW` rail |
| J13 | Selector (jumper): `+5V_RAW` → Vin (1–2, default position) or → Vout (2–3) |
| J14 | Jumper bypassing D1 (when the diode is not needed) |
| J15 | MicroSD module connection (SPI, +3V3 power) |
| J16 | Duplicate of J2 (`+5V_RAW` tap) on the other side of the PCB |

**Materials:**
- Schematic and documentation: https://murmulator.ru/howto
- Classic schematic page: https://github.com/AlexEkb4ever/MURMULATOR_classical_scheme

**Base version:** the build is originally based on the documentation of the official "Murmulator on a 7×9 perfboard with VGA" variant — https://murmulator.ru/mm-maket (documentation archive: https://file.murmulator.ru/data/murmulator-types/mm-maket/mm-docs.zip). The current version extends it with additional features while remaining on perfboard.

**Related project documents:**
- `blueretro-murmulator-project-summary.md` — full details of the DIY BlueRetro adapter build (ESP32 board selection, input power protection, LED indication, firmware, oscilloscope verification), aligned with section 5.1 below.
- `blueretro-murmulator-user-guide.md` — instructions for pairing/disconnecting Bluetooth gamepads.

---

## 2. SD Module Rework (AMS1117 + 74VHCT125A)

The standard cheap SD module (5V/3.3V, SPI) requires rework — otherwise there are voltage drops/failures at high SPI speed.

| Element | Action | Why |
|---|---|---|
| **AMS1117** (3.3V regulator) | Desolder; feed 3.3V from the Pico directly to the pad where the regulator's output used to be (card VCC) | Unnecessary link — the Pico already outputs clean 3.3V, the regulator only adds a voltage drop |
| **74VHCT125A** (SPI buffer, SOIC/TSSOP-14) | Desolder; bridge input→output with jumpers on the SCK, MOSI, CS lines (and MISO, if it also goes through the buffer) | Signal propagation delay through the buffer degrades edges at high SPI speed, causing the SD card to fail |

**Work order:**
1. Use a multimeter to check which traces on the specific module instance run through each chip (revisions differ).
2. Desolder the AMS1117 and 74VHCT125A **before** mounting the module on the perfboard (easier to work with a hot air gun/soldering iron).
3. Solder jumpers in place of the removed components.
4. Check for shorts between adjacent traces.
5. Only after that, connect 3.3V power and the SPI lines from the Pico.

**Result:** the module turns from a "5V adapter with level-shifting" into a clean 3.3V SD card breakout, with no active/passive elements between the Pico and the card on the data lines.

**MicroSD module connection — connector J15** (per the netlist `rp2040_black_vga.net` of 2026-08-29):

| J15 pin | Signal | Pico GPIO |
|---|---|---|
| 1 | GND | — |
| 2 | VCC | `+3V3` |
| 3 | MISO | GP4 |
| 4 | MOSI | GP3 |
| 5 | SCK | GP2 |
| 6 | CS | GP5 |

### 2.1. Pull-up Resistors on the SD Module Lines

After removing the 74VHCT125A, two SPI lines need to be separately checked/provided with pull-ups:

| Line | Pull-up needed? | Value | Why |
|---|---|---|---|
| **CS** | Yes, mandatory | ~10 kΩ to 3.3V | With the Pico's GPIO in an undefined state at startup, the CS line must remain high (deselected), otherwise the card may interpret noise on MOSI/CLK as commands |
| **MISO (DO)** | Recommended | ~10–50 kΩ to 3.3V | When CS is inactive, the card's output goes Hi-Z — a pull-up keeps the line from "floating" and picking up interference |
| **MOSI, SCK** | Not required | — | Always actively driven by the master (Pico), never go Hi-Z |

**What to check on a specific module:**
Some cheap SD modules place pull-ups right next to the 74VHCT125A buffer. After desoldering it and installing jumpers, the resistors may have:
1. stayed in the circuit independently of the buffer — then everything is fine;
2. disappeared along with the buffer footprint — then they need to be added separately.

**Check procedure:** measure with an ohmmeter between CS and 3.3V, and between MISO and 3.3V (together with the short check after the jumpers from step 2). A finite resistance (10–50 kΩ) means the resistor is in place. An open circuit means you need to solder 10 kΩ resistors from CS and MISO to the 3.3V rail on the perfboard.

**Confirmed by the netlist (`rp2040_black_vga.net`):** the pull-ups are already in the schematic — R28 (10 kΩ) on CS (GP5) and R27 (10 kΩ) on MISO (GP4), both to the `+3V3` net. This matches the recommendation above (CS ~10 kΩ mandatory, MISO 10–50 kΩ recommended).

*Symptom of a missing CS pull-up: the card is detected unreliably at startup ("works every other time"), which is hard to diagnose after the fact.*

---

## 3. VGA (J3)

- Resistor R2R ladder on the Pico's digital GPIOs — a passive circuit, needs no separate power.
- It is easier not to buy a VGA connector, but to cut a ready-made shielded cable from an old video card (it often has a connector that is present but not soldered).
- Solder the resistors compactly, close to the cable/connector, with short leads — critical for image cleanliness (interference).

---

## 4. Keyboard (PS/2, J4)

**Connector:** J4 — Mini-DIN-6 (PS/2).

### 4.1. Keyboard Power

The keyboard is powered **separately from the main 3.3V peripherals**, directly from **Vout (pin 40 of the YD-RP2040)** — the node after the BAT54C diode-OR, not from 3V3(OUT). For details, see section 9 ("Board Power").

| Parameter | Value |
|---|---|
| Source | Vout (after BAT54C), not 3V3(OUT) |
| Nominal drop | ~0.3–0.45V across BAT54C relative to the 5V input (in the default J13 position; when powered from J12 the drop across D1 is added, see section 10) |
| Resulting voltage on Mini-DIN-6 pin 4 | 4.62–4.67V when powered from the module's USB Type-C or from J1 (measured, within PS/2 specification tolerance); 4.04–4.29V when powered from J12 via D1 — see the measurements in section 10 |
| Keyboard current draw | ~100 mA (account for it when budgeting current through BAT54C, see section 10) |

### 4.2. Signal Lines (level-shifting 5V → 3.3V)

PS/2 CLK and DATA are 5V logic on the keyboard side, the Pico's GPIOs are 3.3V and not 5V-tolerant. A **passive circuit** is used for level matching: a pull-up resistor + a series resistor + a zener clamp on each line.

**Signal mapping (per the MURMULATOR_classical_scheme reference schematic, KiCad):**

| Signal (raw, 5V, from the keyboard) | After the zener clamp | Pico GPIO |
|---|---|---|
| PS2 CLK (Mini-DIN-6, pin 5) | PS2_CLK_3V | GP0 |
| PS2 DATA (Mini-DIN-6, pin 1) | PS2_DATA_3V | GP1 |

**Actual line circuit (confirmed by the netlist `rp2040_black_vga.net`):**

```
J4 (5V raw) ──[R9/R10, 1 kΩ, pull-up to +5V_VOUT]── CLK/DATA node ──[R11/R12, 100 Ω]── GPIO (GP0/GP1) ──[D2/D3, zener]── GND
```

| Element | Value | Role |
|---|---|---|
| R9 (CLK) / R10 (DATA) | 1 kΩ | Pull-up of the raw 5V line to +5V_VOUT — mandatory, since the keyboard output is open-collector |
| R11 (CLK) / R12 (DATA) | 100 Ω | Series resistor between the raw line and the GPIO — limits current through the zener when it conducts, and together with the GPIO input capacitance forms an RC filter |
| D2 (CLK) / D3 (DATA) | 3.3V | Cathode to GPIO, anode to GND — clips the upper signal level at the GPIO |

Circuit: the raw 5V signal (already pulled up to +5V_VOUT by R9/R10) → passes through the series resistor R11/R12 (100 Ω) → the zener clamp limits the upper level at the GPIO itself → the signal reaches GP0/GP1.

**Connection to the GPIOs — via jumpers on J5 (per the netlist `rp2040_black_vga.net` of 2026-10-03):** the `PS/2_DATA_3V` and `PS/2_CLK_3V` nodes (after R11/R12, at the D2/D3 cathodes) are not connected to the GPIOs directly on the board — they are brought out to J5 (pins 29 and 31), with GP1 (pin 30) and GP0 (pin 32) brought out next to them. The connection is made with jumpers J5:29–30 (DATA → GP1) and J5:31–32 (CLK → GP0); on the schematic they are indicated by the net ties NT5/NT6 ("Exclude from board" symbols, not included in the netlist or on the board).

---

## 5. Joysticks (Dendy, 2 ports, DE-9, J6/J7)

**Connector labels on the board:** Joystick 1 — **J6**, Joystick 2 — **J7**.

Dendy-type joysticks are used (9-pin, narrow connector) — inside is an active shift register chip (CD4021 type or equivalent), not just mechanical contacts.

**DE-9 pinout (Dendy 9-pin):**

| DE-9 pin | Purpose |
|---|---|
| 2 | DATA — shift register output |
| 3 | LATCH — latch/counter reset |
| 4 | CLOCK — shift clock pulse |
| 6 | +5V — standard power for the register chip |
| 8 | GND |

**Why the register is powered from 3.3V rather than the standard 5V:**
The joystick DATA output follows its supply level. When powered from 5V, the DATA line becomes 5V logic, while RP2040 GPIOs are not 5V-tolerant (max ~3.6V) — a protective resistor (~1 kΩ) on DATA would be needed as a crude current limit into the port's protection diode. The register chips (CD4021-like) are ordinary CMOS logic with a wide supply range (typically 3–18V), so powering them from 3.3V instead of 5V is a standard and safe solution: DATA becomes 3.3V logic right away, no resistor is needed, and the connection to the GPIO is direct.

**Final wiring (2 ports):**

| Signal | Joystick 1 (DE-9 pin) | Joystick 2 (DE-9 pin) | Pico GPIO |
|---|---|---|---|
| DATA | 2 | 2 | GP16 (J6) / GP17 (J7) — separate |
| LATCH | 3 | 3 | GP15 — shared by both ports |
| CLOCK | 4 | 4 | GP14 — shared by both ports |
| VCC | 6 | 6 | 3.3V — shared rail |
| GND | 8 | 8 | GND — shared rail |

CLOCK and LATCH are shared by both joysticks (both registers are clocked and latched synchronously), DATA must be a separate line for each port, otherwise the outputs conflict on one line.

**Joystick pinout and firmware — verified:** the Tecnocat firmware works with BlueRetro on the current wiring (GP14 CLOCK, GP15 LATCH, GP16/GP17 DATA) without config changes. The previously noted open question is closed.

### 5.1. BlueRetro Integration (Bluetooth gamepads instead of physical Dendy joysticks)

> 📄 **Full details of the BlueRetro build** (board selection, input power protection, BOOT button, LED indication, firmware version, oscilloscope verification of signals) have been moved to a separate document **`blueretro-murmulator-project-summary.md`**. Below is a summary aligned with it, sufficient for the context of the main Murmulator build.

There are no physical Dendy joysticks in the project — both DE-9 ports (J6/J7) were designed from the start as **virtual**, served by a single homemade DIY **BlueRetro** adapter (ESP32, firmware from github.com/darthcloud/BlueRetro — the repository was archived by the owner on 14.12.2025, there is no more active development, but firmware releases and documentation remain available).

**Board used: ESP32 D1 mini (MH-ET LIVE, CH9102 USB-serial chip).** Chosen after comparison with the 30-pin WROOM DevKit (rejected — GPIO0, needed for BlueRetro BOOT/pairing, is not physically broken out) and the 38-pin WROOM DevKit (technically suitable and matches the BlueRetro author's official recommendation, but not chosen — it would have required reworking the already finished KiCad schematic). A known trade-off is that the onboard AMS1117 on the D1 mini heats up to ~54°C, close to the softening temperature of the PLA enclosure; the cooling decision has been deliberately postponed (risk accepted). Selection details are in `blueretro-murmulator-project-summary.md`.

**Bluetooth gamepad compatibility:** BlueRetro works over standard Bluetooth HID (BR/EDR and LE) and supports PS3/PS4/PS5, Xbox One, Xbox Series X|S, Wii/Switch and generic HID BT devices.

**The signal protocol does not differ from a physical Dendy joystick:** Dendy, NES/Famicom and the BlueRetro output in FC/NES mode all use the same shift-register principle (CLOCK/LATCH/DATA), so the Murmulator cannot tell a physical joystick from one emulated by BlueRetro.

**Signal direction:** the Nintendo PISO (Parallel-In, Serial-Out) protocol — **the console (in this case the Murmulator/RP2040) generates CLOCK and LATCH**, the controller (a physical Dendy register or BlueRetro emulating it) is the passive side that only responds to them with the **DATA** signal. Therefore, on the BlueRetro ESP32 the OUT0 (LATCH) and CUP (CLOCK) signals are **inputs** receiving the signal from the Pico, not outputs; the only signal the ESP32 itself generates is **DATA**.

**ESP32 (BlueRetro, FC/NES mode) → DE-9 → Pico GPIO mapping:**

| Signal | Direction | ESP32 pin (BlueRetro) | System Pin (std. NES 7-pin) | DE-9 J6 pin | DE-9 J7 pin | Pico GPIO |
|---|---|---|---|---|---|---|
| LATCH (shared, OUT0) | Pico → ESP32 (input) | IO32 | 3 | 3 | 3 | GP15 |
| CLOCK port 1 (P1_CUP) | Pico → ESP32 (input) | IO5 | 2 | 4 | — | GP14 |
| CLOCK port 2 (P2_CUP) | Pico → ESP32 (input) | IO18 | 2 | — | 4 | GP14* |
| DATA port 1 (P1_D0) | ESP32 → Pico (output) | IO19 | 4 | 2 | — | GP16 |
| DATA port 2 (P2_D0) | ESP32 → Pico (output) | IO22 | 4 | — | 2 | GP17 |
| GND | — | ESP32 GND | 1 | 8 | 8 | GND |
| VCC (+5V, NES pin 7) | — | — | 7 | 6 | 6 | do not connect |

*\*CLOCK for J6 and J7 is one and the same signal from a single Pico output (GP14), physically routed to two ESP32 inputs (IO5 and IO18). Since there is one source (Pico, master) and the ESP32 is only a receiver (input) in both cases, this is the standard and safe "one driver — multiple receivers" topology: there can be no bus conflict or signal desynchronization, and no separate oscilloscope check is needed before joining the lines.*

**VCC (pin 6, DE-9) is not used:** the ESP32 inside BlueRetro natively runs on 3.3V logic (consistent with the already adopted decision to power the joystick node from 3.3V — see above), the CLOCK/LATCH/DATA signal lines go to the Pico GPIOs directly, without level-shifters (which in official BlueRetro builds are only needed to match a real 5V NES console).

**Power for the ESP32 module itself (BlueRetro) is separate from the DE-9 port power.** DE-9 pin 6 (VCC) on both ports stays at **3.3V**, as originally decided for physical Dendy joysticks (see above) — BlueRetro power is not routed through the signal connector, to keep the ports compatible with a physical joystick in the future.

The ESP32 module (**DevKit with an onboard AMS1117 regulator**, 5V→3.3V on the module board itself) is powered from the main board via connector **J2** (JST-XH, tap of the `+5V_RAW` rail; duplicate — J16 on the other side of the PCB, see section 10) → by wire → connector **J3** of the BlueRetro board. On the BlueRetro board itself, J3 is the standard power input, already protected by its own Schottky diode **D1 = 1N5819** (connected between J3 and the ESP32 5V input, U1 pin 35). The result is that BlueRetro is powered from the `+5V_RAW` rail through its own Schottky diode, with no load on the YD-RP2040 module's onboard BAT54C and no overlap with the keyboard's current budget on Vout (in the default J13 position). The regulator on the DevKit board itself steps down the incoming ~4.5–4.6V (after the diode drop) to 3.3V for the ESP32; no separate external regulator on the perfboard is required.

**⚠️ Limitation: BlueRetro/joysticks work only when powered via the `+5V_RAW` rail — from J1 (DC jack) or J12 (Micro USB).** The `+5V_RAW` rail is powered only from these two inputs — unlike Vout (the node after BAT54C), where the module's USB Type-C and Vin converge and which powers the keyboard regardless of how the board is powered. The YD-RP2040 has no accessible "raw" USB VBUS before the diode (unlike the original Pico, where VBUS is broken out on a separate pin) — the module's USB Type-C goes through the same BAT54C as Vin, and only the combined Vout is brought out. Therefore, when the board is powered only from the module's own USB Type-C (without J1/J12), the ESP32/BlueRetro and, consequently, both joysticks will not work. To work with BlueRetro joysticks, power must be fed to J1 or J12.

**Why not from Vout (pin 40) of the RP2040:** Vout on the YD-RP2040 is the node right after the diode-OR on the **BAT54C**, a signal diode with a current limit of ~150–300 mA per leg (see sections 9–10). This node is already partially loaded: the PS/2 keyboard (section 4) is powered precisely from Vout and draws about ~100 mA at idle, with possible spikes during activity. An ESP32 with an active Bluetooth radio would add another 200–300 mA peak through the same diode — the total current would almost certainly exceed the BAT54C rating, especially when peaks coincide (a key press + a BT packet transmission at the same time). Therefore the ESP32 is powered directly from the 5V rail, bypassing Vin/Vout/BAT54C, without sharing this node with the keyboard.

**ESP32 peak consumption — measured:** a multimeter recorded **~155 mA** on the 5V rail with the BT radio active. Since the ESP32 is powered via J2 (main board) → J3 (BlueRetro), bypassing BAT54C and the Vout node, this load does not share the current budget with the PS/2 keyboard and is not limited by the BAT54C rating (~150–300 mA per leg) — the current margin is sufficient. The open question on this point is closed.

**Status LED indication (implemented):** pin **IO17** — global-status LED (pairing/error, blue, two-resistor fail-safe circuit); in addition, two port-status LEDs are wired on **IO2/IO4** (green, via 2N7000 MOSFETs, since IO2/IO4/IO17 are ESP32 strapping pins). Logic: during pairing, the global LED and the LED of the first free port pulse; when a gamepad connects, the corresponding port LED lights solid. The full circuit, resistor values and measured Vf are in `blueretro-murmulator-project-summary.md`.

**Verified with an oscilloscope (consistent with the table above):** CLOCK (GP14→IO5/IO18) — 12.5 kHz, 50% duty cycle; LATCH (GP15→IO32) — period 1200 µs, pulse 40 µs; DATA — inverted logic (LOW = button pressed), 80 µs pulse per clock. The canonical NES bit order (A, B, Select, Start, Up, Down, Left, Right) is confirmed for the Xbox One gamepad. The "one driver (Pico) — receivers (ESP32)" topology on the joined CLOCK lines is confirmed electrically, with no bus conflict.

**Firmware:** `v25.04_hw1.zip` (HW1 specification, file `BlueRetro_hw1_nes.bin`), flashed via `pio pkg exec -- esptool.py`. BlueRetro module input power protection (connector J3, diode D1=1N5819, RESET button SW2, decoupling capacitors) and details of the BOOT/IO0 button (external pull-up R1=10kΩ) — see `blueretro-murmulator-project-summary.md`.

---

## 6. Audio Out (J8)

**Connector:** J8 — output audio jack (L/R). **J8B (OUT_A)** — a duplicate pin header (PinHeader) in parallel with J8, for convenient connection of the audio output with a separate wire/cable (for example, if the enclosure/wiring does not allow using the jack itself directly, or for test measurements).

**J8B (OUT_A) pinout:**

| J8B pin | Signal | Corresponds to |
|---|---|---|
| 1 | L | the same node as L on jack J8 |
| 2 | GND | common ground (the same node as S on jack J8) |
| 3 | R | the same node as R on jack J8 |

**GPIO assignment:**

| Net | Pico GPIO | Channel |
|---|---|---|
| LEFT_OUT | GP27 | Left channel (game sound) |
| RIGHT_OUT | GP26 | Right channel (game sound) |
| BEEP_OUT | GP28 | Beeper / TAP loading indication, mixed into both channels |

**Connection to the GPIOs — via jumpers on J5:** the `RIGHT_OUT`, `LEFT_OUT` and `BEEP_OUT` nets are not connected to the GPIOs directly on the board — they are brought out to J5 next to the corresponding GPIOs and connected with jumpers: J5:21–22 (`RIGHT_OUT` → GP26), J5:23–24 (`LEFT_OUT` → GP27), J5:25–26 (`BEEP_OUT` → GP28). On the schematic these jumpers are indicated by the net ties NT2/NT3/NT4 ("Exclude from board" symbols, not included in the netlist or on the board).

**Circuit (main L/R channels):**

| Channel | Series resistor (GPIO → node) | Coupling capacitor (node → jack) | Node bias resistor to GND |
|---|---|---|---|
| L | R21, 1 kΩ | C4, 10 µF | R22, 330 Ω |
| R | R25, 1 kΩ | C6, 10 µF | R26, 330 Ω |

**Mixing the beeper (BEEP_OUT, GP28) into both channels:**

| Element | Value | Role |
|---|---|---|
| R23 | 2 kΩ | Summing BEEP_OUT → node L (via C5) |
| R24 | 2 kΩ | Summing BEEP_OUT → node R (via C7) |
| C5 | 10 nF | Couples node L with the BEEP mixing bus |
| C7 | 10 nF | Couples node R with the BEEP mixing bus |

**An important feature by design:** the beeper is deliberately mixed in much quieter than the game sound — the coupling capacitors in the BEEP path are three orders of magnitude smaller (10 nF vs. 10 µF in the main channel), and the summing resistors (2 kΩ) are higher than the series resistors of the main channel (1 kΩ). This is a deliberate circuit decision (a quiet indicator signal on top of the main sound), not an error/imbalance.

**Tecnocat firmware setting:** by default, sound output in SETTINGS is set to **NONE** — there will be no sound on J8 until it is manually switched to **Beeper+AY**.

---

## 7. Audio In (J9)

**Connector:** J9 — 3.5mm input audio jack (TRS, `AudioJack3_SwitchTR`). Only the **Tip** is used (in the netlist — the `/LOAD_IN_L` signal); the **R/RN** contacts (right channel) are not connected anywhere (`unconnected-(J9-PadR)`, `unconnected-(J9-PadRN)`). The input is effectively **mono** — as it should be for a tape/TAP signal (the right and left channels of a TAP file are usually identical, so only one is used).

**Purpose: loading programs.** The audio input serves the same role as the tape input of classic 8-bit computers (ZX Spectrum, etc.) — an audio signal from an external source (tape recorder, phone/computer playing a TAP file, tape emulator) is fed to J9/J9B1, and the circuit on transistors Q1/Q2 shapes it into a digital signal for loading programs.

**J9B1 (IN_A)** — a duplicate 2-pin header (PinHeader) in parallel with the Tip signal and GND of jack J9 — similar to J8B (section 6), for convenient connection of the audio input with a separate wire instead of the jack itself.

**J9B1 (IN_A) pinout:**

| J9B1 pin | Signal | Corresponds to |
|---|---|---|
| 1 | LOAD_IN_L (Tip) | the same node as T on jack J9 |
| 2 | GND | common ground (the same node as S on jack J9) |

**Transistors Q1/Q2 — S9014 used instead of BC850.** In the netlist `rp2040_black_vga.net`, both shaper transistors (Q1, Q2) are marked as **S9014** (NPN, TO-92) — used as an alternative to the **BC850** specified in the original classic Murmulator schematic (`original_classic_murmulator_37NJU22_emul_RPPICO_card.pdf`, equivalent positions Q3/Q4). S9014 and BC850 are comparable general-purpose NPN transistors, used here as a comparator/switching stage rather than a power amplifier, so the substitution is not critical to exact parameters (what matters is NPN, a similar hFE and sufficient speed for the TAP signal clock rate).

**Shaper circuit (2 transistor stages, per the netlist):**

```
J9 (T, Tip) / J9B1 pin1 ──┬── R33, 100R ── GND   (input signal load)
   (net /LOAD_IN_L)       │
                          C11, 100n              (coupling capacitor)
                           │
                          R32, 10k
                           │
                          ├── D4 (4148): cathode → Q1 base, anode → Q1 collector (B-C clamping diode)
                           │
                          Q1 base (S9014), Q1 emitter → GND
                           │
                          Q1 collector ──┬── R29, 10k ── Q2 base (S9014), Q2 emitter → GND
                                          │
                                          └── R30, 10k ── node ──┬── C12, 100n ── GND
                                                                  └── R31, 1k ── Q2 collector → LOAD_IN_D
```

Q1 works as the first shaping stage (switching between cutoff and saturation according to the input audio signal taken via C11/R32 from Tip), Q2 is the second stage, additionally inverting/amplifying the digital edge. The resulting signal is the **LOAD_IN_D** net.

**Connecting LOAD_IN_D to a GPIO — via jumper J5:17–18.** The `LOAD_IN_D` net is brought out to J5 (pin 17), with GP22 brought out next to it (pin 18); the connection is made with jumper J5:17–18 (`LOAD_IN_D` → GP22). On the schematic it is indicated by the net tie NT1 (an "Exclude from board" symbol, so in the netlist `rp2040_black_vga.net` of 2026-10-03 the `LOAD_IN_D` and `GP_22` nets are separate). The previously noted open question is closed.

**Power for the R30/R31 collector node — confirmed by the netlist (`rp2040_black_vga.net` of 2026-08-29).** Resistors R30 and R31 (the collector loads of Q1 and Q2) are connected to the `+3V3` net — the pull-up to power is in place, the stage outputs a 3.3V logic level. The previously noted open question is closed.

### 7.1. Audio-in verification (loading check)

To verify the audio input, the online TAP file player **[iratahack.com/zxtape](https://www.iratahack.com/zxtape/)** was used — the service plays a `.tap` file as an audio signal right in the browser (emulating a tape signal), which is fed to J9/J9B1 with an ordinary audio cable from the computer/phone output.

**Audio output settings on the laptop:**

Disable sound "enhancers" — Windows and audio drivers by default mix equalizer/spatial effects/noise suppression into the signal, which distort the TAP signal waveform and interfere with its correct recognition by the shaper circuit (section 7):

1. Press `Win + R`, type `mmsys.cpl` and press Enter.
2. Right-click your audio output/speakers/headphones → Properties → Advanced tab.
3. Find the **Enhancements** (Signal Enhancements) or **Spatial Sound** tab.
4. Check **"Disable all enhancements"** and turn off Dolby/DTS if they are active.

**Check procedure:**
1. Open https://www.iratahack.com/zxtape/ in the browser on the device the signal will be fed from.
2. Upload/select the desired `.tap` file on the site.
3. Connect this device's audio output (headphone jack) to J9 (or to the duplicate header J9B1) with an audio cable.
4. Start playback on the site simultaneously with the load command in the Murmulator firmware/emulator (the equivalent of `LOAD ""` on the ZX Spectrum).
5. Make sure the program loaded successfully (by the BEEP_OUT/loading beeper indication — see section 6 — and/or by the loaded program actually starting).

**Status:** the verification method is confirmed — the iratahack.com/zxtape service was used as the TAP signal source for loading programs via J9.

---

## 8. External Connector (J5)

**Purpose:** J5 is a 40-pin expansion connector (EXT) intended for connecting external boards/peripherals. By design it is similar to the `J2 EXT` connector from the original classic Murmulator schematic (`original_classic_murmulator_37NJU22_emul_RPPICO_card.pdf`, component `J2`).

### 8.1. Power on J5: pins 38/39/40

**In the original schematic (genuine Raspberry Pi Pico):**

| J2 EXT pin (original) | Connected to | Pico pin |
|---|---|---|
| 39, 40 (doubled) | **VBUS** — raw power, available only when USB is connected | 40 |
| 38 | **VSYS** — combined node (after the Pico's internal diode), available from USB or from VSYS power | 39 |

The idea of the original: the external board on J2 is given both the "raw", high-current USB output (VBUS, doubled on 2 pins for current) and the universal VSYS signal (available regardless of the power source).

**The YD-RP2040 physically has no separate "raw VBUS-from-USB-only"** (see section 9) — there is only Vin (before combining) and Vout (after combining via BAT54C). There is no direct 1:1 equivalent of VBUS.

**Verified against the current netlist (`rp2040_black_vga.net` of 2026-10-03): decision made.**

| J5 pin | Net per netlist | Corresponds to |
|---|---|---|
| 39 | `/+5V_VOUT` | Vout (U1, pin 40) — moved, as planned |
| 40 | `/+5V_VOUT` | Vout (U1, pin 40) — moved, as planned |
| 38 | `/+5V_IN` | Vin (U1, pin 39); in the default J13 position (1–2) connected to the `+5V_RAW` rail — **intentionally left on `+5V_IN`** |

Pins 39 and 40 have already been moved to `+5V_VOUT` (the same node as keyboard J4) — they work regardless of the board's power source, including in USB-only mode. Pin 38 remained on the `/+5V_IN` (Vin) net — it is powered **only** when there is power on the `+5V_RAW` rail (J1 or J12) and J13 is in the default position 1–2 (see section 10), i.e. in "module USB Type-C only" mode and with J13 in position 2–3 pin 38 is unpowered. This is an intentional decision: in USB-only mode the power pins 39/40 are available to the external board on J5.

```
J5, pins 39/40 ──┐
                   ├── node +5V_VOUT ── U1, Vout (pin 40)   [already so in the schematic]
J4, pin 4 (keyboard)──┘

J5, pin 38 ── node +5V_IN ── U1, Vin (pin 39)   [from the +5V_RAW rail via J13 in position 1–2 — see section 10]
```

**Important limitation of the `+5V_VOUT` node:** in USB-only mode and in the default J13 position (1–2) the only path to this node is through the onboard **BAT54C** signal diode, limit ~150–300 mA. The keyboard already sits on this node (~100 mA at idle, spikes possible). The total current of the keyboard + the board on J5 (pins 39/40) in USB-only mode must not exceed the BAT54C rating — with a significant current draw by the board on J5 (comparable, for example, to BlueRetro, ~150+ mA) the budget may not be enough. The actual consumption of the board on J5 needs to be estimated/measured before relying on this node. There is no external board on J5 yet — the measurement is postponed until one appears.

**Status:** decision made — pins 39/40 on `+5V_VOUT`, pin 38 stays on `+5V_IN` (by analogy with pin 38/VSYS in the original schematic, where the "raw" input and the combined output were kept separate). Confirmed by the netlist.

### 8.2. Other J5 Pins

Signal/GPIO pins — per the netlist `rp2040_black_vga.net`, correspond to the Pico's physical GPIOs (similar to the pinout in other sections — VGA, PS/2, joysticks, audio). Details on specific EXT connector signals — to be clarified separately if needed.

---

## 9. Board Power: Vin/Vout Architecture and USB-only Mode

**Module used: YD-RP2040** (a Raspberry Pi Pico clone). This board **has no VBUS/VSYS pins** in the usual form — instead, **Vin (pin 39)** and **Vout (pin 40)** are broken out, and the diode-OR of the sources is already implemented on the module itself via the **BAT54C** dual Schottky diode.

- No separate power supply is required in the base circuit — the whole device is powered through the module itself.
- Input: USB Type-C on the module (5V) **or** the **Vin** pin (the equivalent of VSYS on the original Pico).
- The onboard converter on the module provides a stable **3.3V** on the **3V3(OUT)** pin.
- The main peripherals on the perfboard are powered from 3V3(OUT): the reworked SD module, the joystick (VGA needs no power — passive circuit).
- **The PS/2 keyboard (J4) is powered separately — 5V taken from the Vout pin** (the node after the BAT54C diodes, not from 3V3(OUT)). This is the standard supply voltage for the PS/2 protocol, so the keyboard power line is taken before the 3.3V regulator, not after it.
  - The Data and Clock lines remain 5V logic on the keyboard side, while the module's GPIOs are 3.3V, so the signal lines need a step-down stage (voltage divider or zener clamps), separately from the power question.

### Vin / Vout instead of VBUS / VSYS

On the YD-RP2040, the diode-OR between USB and external power is already routed on the board:

```
USB (VBUS) ──[diode 1 in BAT54C]──┐
                                    ├──> Vout (pin 40) ──> [regulator] ──> 3.3V
Vin (pin 39) ──[diode 2 in BAT54C]─┘
```

| Pin | What it is | Notes |
|---|---|---|
| **Vout (pin 40)** | VSYS equivalent — node after the diodes, input of the 3.3V regulator | Voltage = what actually comes from the active source, minus the drop across the BAT54C diode |
| **Vin (pin 39)** | Ready-made input for external power | Already connected to Vout via a BAT54C diode — no separate Schottky diode needed |

**Important:** as with VBUS/VSYS on the original Pico — if the module is not always connected via USB, you cannot take 5V for peripherals directly from the USB line; when powered from Vin, this line will be unpowered.

**USB-only power mode:** since Vout combines USB and Vin via BAT54C, all peripherals powered from Vout/3V3(OUT) — the keyboard (section 4), the SD module (section 2), physical joysticks (section 5) — work when the board is powered from USB only, without an external PSU. The exception is **BlueRetro**: it is powered via J2/J16 from the `+5V_RAW` rail, which receives power only from J1 or J12 and is not combined with the module's USB Type-C via BAT54C, so it requires power on J1 or J12 (for details — section 5.1 and section 10).

---

## 10. External Power (J1/J12/J13/J14, taps J2/J16/J11): +5V_RAW Rail and the J13 Selector

For the YD-RP2040, an additional Schottky diode to power the module itself is not required — the diode-OR between the module's USB Type-C and Vin is already implemented on the module via BAT54C. On the Murmulator board, external power is collected on a common **`+5V_RAW`** rail, and where this rail is fed into the module (Vin or Vout) is chosen with the jumper selector **J13**.

**Power circuit (per the netlist `rp2040_black_vga.net` of 2026-08-29):**
```
External PSU 5V(+) ──> J1 (DC Jack), pin 1 ────────────────────┐
                                                                  ├── rail +5V_RAW ──┬── J2 (JST-XH), pin 2   (tap for external peripherals, BlueRetro)
J12 (Micro USB module), VBUS ── D1 (1N5819, anode→cathode) ─────┘                  ├── J16, pin 2          (duplicate of J2 on the other side of the PCB)
          └── J14 (jumper bypassing D1) ──────────────────────────┘                  ├── C19 (470µF)
                                                                                      └── J13, pin 2 (common pin of the selector)

J13 (selector): pin 1 ── /+5V_IN  (U1, Vin, pin 39; J5 pin 38)    ← default position: jumper 1–2
                pin 3 ── /+5V_VOUT (U1, Vout, pin 40; J4 pin 4; J11 pin 2; J5 pins 39/40)   ← jumper 2–3

External PSU GND ──> J1, pin 2 / common board GND (J12, J2, J16 — GND also on the common rail)
```

**`+5V_RAW` sources:**
- **J1** — DC jack of the external PSU, connected to the rail directly, without a diode.
- **J12** — a second power input from USB (Micro USB module) instead of the DC jack. Connected through the Schottky diode **D1 (1N5819)**: anode — J12 VBUS, cathode — `+5V_RAW`. D1 prevents voltage from the rail (for example, from the PSU on J1) from flowing back into the USB source on J12.
- **J14** — a jumper bypassing D1 when the diode is not needed (for example, when J12 is the only source and you want to remove the ~0.3–0.45V drop across D1). With J14 closed there is no reverse-current protection for J12 — do not connect a PSU on J1 and USB on J12 at the same time.

**J13 selector — where `+5V_RAW` is fed:** the default position (1–2, `+5V_RAW` → `+5V_IN`) is indicated on the schematic by the net tie NT8 (an "Exclude from board" symbol, not included in the netlist or on the board).

| J13 position | Where `+5V_RAW` goes | Path to `+5V_VOUT` (keyboard J4, J11, J5:39/40) | Notes |
|---|---|---|---|
| **1–2 (default)** | Vin (U1, pin 39) + J5 pin 38 | Through the module's onboard BAT54C (Vin → BAT54C → Vout) | Current to `+5V_VOUT` is limited by BAT54C (~150–300 mA per leg). The RAW source and the module's USB Type-C are isolated by the BAT54C diodes — no reverse current between them |
| **2–3** | Vout (U1, pin 40) directly | Bypassing BAT54C | Current to `+5V_VOUT` is limited only by the source (and by D1 1A when powered from J12). Vin and J5 pin 38 are then unpowered |

**⚠️ Peculiarity of position 2–3 (confirmed by measurement F, see below):** in this position `+5V_RAW` and `+5V_VOUT` are a single node with no diode between them. If the board is powered only from the module's USB Type-C (without J1/J12), Vout (~4.7V after BAT54C) reaches the `+5V_RAW` rail through J13, i.e. J2/J16 (BlueRetro) and J1 — the BlueRetro load will go through BAT54C, and the voltage will appear on the DC jack connector. Position 2–3 makes sense only when powered from J1/J12.

**Voltages with J13 = 1–2 — measured with a multimeter (2026-10-03):**

| Variant | Source | BlueRetro on J16 | J12 VBUS | `+5V_RAW` = `+5V_IN` | `+5V_VOUT` (keyboard J4) |
|---|---|---|---|---|---|
| A | PSU 5V 1A on J1 | no | — | 5.10V | 4.67V |
| B | PSU 5V 1A on J1 | yes | — | 5.08V | 4.65V |
| C | Baseus 30W charger on J12 (via D1) | no | 5.04V | 4.72V | 4.29V |
| D | Baseus 30W charger on J12 (via D1) | yes | 4.83V | 4.47V | 4.04V (keyboard works) |
| E | Module USB Type-C (from a laptop) | no | — | 0V at first, then slowly rising | 4.62V |

**Conclusions from the measurements:**
- Drop across BAT54C (Vin → Vout) — **~0.43V** in all variants A–D.
- Drop across D1 (1N5819) — **~0.32V** without BlueRetro (C) and **~0.36V** with BlueRetro (D).
- When powered from J12 with the BlueRetro load, VBUS sags by ~0.21V (5.04 → 4.83V) — a sag on the charger/cable/Micro USB module side; the PSU on J1 sags by only ~0.02V under the same load.
- Variant D is the worst case: `+5V_VOUT` = 4.04V, and the keyboard still works; BlueRetro power is fine — 3.296V measured on the 3V3 pin of the ESP32 D1 mini. The voltage when powered from J12 can be raised with jumper J14 (bypassing D1, ~+0.3V) — in that case a PSU must not be connected to J1 at the same time.
- Variant E: when powered only from the module's USB Type-C, the `+5V_RAW` rail is not powered, but the voltage on it slowly rises — probably the reverse leakage current of the Schottky diodes (BAT54C) charging the unloaded capacitors C19/C20. With a load connected (BlueRetro) this voltage does not hold.

**J13 = 2–3 position — measured (2026-10-04), variant F:** powered from the module's USB Type-C (from a laptop), no BlueRetro: `+5V_VOUT` = `+5V_RAW` = 4.61V — the Vout voltage does reach the `+5V_RAW` rail (J1, J2, J16). `+5V_IN` (Vin) — ~3.3V right after power-on and slowly rising: in this position Vin is not connected to anything, and it is charged by the BAT54C leakage current through the unloaded C20 (100nF — hence faster than C19 470µF in variant E). J12 VBUS (J14 open) — a steady 4.58V: the reverse leakage current of D1 (noticeably higher for the 1N5819 than for the BAT54C) holds the unloaded J12 input almost at the `+5V_RAW` level. Both voltages are "parasitic", with no load capability; `+5V_IN`/J5:38 and J12 must not be used in this mode.

**J13 = 2–3 position — other variants (estimate, not measured):**

| Source | `+5V_VOUT` (directly from `+5V_RAW`) |
|---|---|
| J1 (PSU 5V) | ~5.0–5.1V |
| J12 via D1 | ~4.5–4.7V |
| J12 with J14 closed | ~4.8–5.0V |
| Module USB Type-C only | 4.61V (measured, variant F) |

**Purpose of J2 / J16:** taps of the `+5V_RAW` rail (J16 is a duplicate of J2 on the other side of the PCB) — used to power external peripherals, in particular the **BlueRetro** adapter (see section 5.1), which already has its own Schottky diode D1 = 1N5819 at the J3 input on its own board. The load on J2/J16 does not pass through the module's BAT54C.

**Purpose of J11:** an additional tap of the `+5V_VOUT` node (the same node as keyboard J4) — for peripherals that should receive 5V from the common Vout node and work regardless of whether the board is powered via the module's USB Type-C or via `+5V_RAW` (J1/J12). In current budget this is the same case as the keyboard: in the default J13 position (1–2) and in USB-only mode, the total current of J4 + J11 + J5:39/40 through BAT54C is limited to ~150–300 mA — if the device on J11 draws noticeable current, the actual current should be measured and, if necessary, it should be powered from J2/J16 or J13 should be moved to position 2–3.

**J11 — confirmed by the netlist (`rp2040_black_vga.net` of 2026-08-29):** J11 pin 2 is on the `/+5V_VOUT` net (together with U1 Vout, J4, J5:39/40, R9/R10). The previously found isolated `/+5V_OUT` net has been eliminated, J11 is electrically powered.

**Details and limitations:**
- BAT54C is a low-power dual signal diode, not a power diode. The typical maximum forward current through one leg is on the order of **150–300 mA** (check against the datasheet for the specific marking on the board).
- Peripherals with noticeable current draw (for example, the BlueRetro Bluetooth module) should be powered via J2/J16 (the `+5V_RAW` rail, bypassing BAT54C), not from Vout/J11.
- External PSU GND — directly to the common board GND, without a diode.
- There is no reverse polarity protection on J1.

*Note: the original Raspberry Pi Pico (not the YD-RP2040) has no such built-in point for external power — there the diode-OR is implemented only between USB and VSYS, and for an external PSU you would have to add your own Schottky diode on VSYS manually, as described in the Pico datasheet, section 4.5 "Powering Pico".*

### 10.1. Power Rail Decoupling Capacitors (added)

As of the current revision of `rp2040_black_vga.pdf`/`.net`, decoupling has been added to all three power nets:

| Net | Capacitors | Comment |
|---|---|---|
| `+5V_RAW` (rail J1/J2/J16, after D1 from J12) | C19, 470 µF (bulk, electrolytic) | Right at the DC jack input — damps drops/ripple from the external PSU |
| `/+5V_IN` (Vin, pin 39 U1, from `+5V_RAW` via J13) | C20, 100 nF | Local ceramic right at the module's input pin |
| `/+5V_VOUT` (Vout, pin 40, keyboard J4, J5:39/40) | C17, 100 nF + C18, 100 µF (bulk) | Especially important for the stability of the node to which the PS/2 line pull-ups R9/R10 are tied (see section 4.2) |
| `+3V3` | C8, C9, C10, C12, C15 (100 nF) + C16 (100 µF, bulk) | Local decoupling at the joysticks (J6/J7), the MicroSD module (J15), the +3V3 tap (J10) and the U1 regulator output |

Verified against the netlist: all second pins of the new capacitors correctly return to GND, no opens found. Requires manual checking during assembly: the voltage rating of the electrolytics (≥16V recommended for 5V rails, ≥10V for 3.3V) and polarity when soldering (C_Polarized).

---

## 11. Assembly Order Checklist

1. [ ] Rework the SD module (desolder AMS1117 + 74VHCT125A, jumpers).
2. [ ] Dry layout of components on the perfboard according to the block diagram from the base version documentation (see item 1).
3. [ ] Route the GND and 3.3V rails along the board edges with separate traces/wire.
4. [ ] Solder in the Pico — preferably on pin header sockets, not directly (for easy replacement/reflashing).
5. [ ] Connect the SD module SPI, VGA R2R resistors, PS/2 keyboard, joystick to the corresponding Pico GPIOs.
6. [x] External power circuit — per the netlist of 2026-08-29: `+5V_RAW` rail from J1 and from J12 (via D1, bypass — J14), selector J13 (Vin — default / Vout — bypassing BAT54C), taps J2/J16, see section 10. During assembly, check the voltage on `+5V_VOUT` with a multimeter in the power variants used.
7. [x] J5: pins 39/40 moved to `+5V_VOUT`, pin 38 intentionally left on `+5V_IN` — see section 8.1.
8. [ ] Check the placement of the connectors (VGA, SD, audio, joystick) against your enclosure before final assembly.
9. [x] Audio input (section 7): LOAD_IN_D is connected to GP22 with jumper J5:17–18 (NT1 on the schematic); the pull-up of the Q1/Q2 collector node to `+3V3` is confirmed by the netlist.
10. [x] Decoupling capacitors on the `+3V3`, `+5V_RAW`, `+5V_IN`, `+5V_VOUT` rails — added and verified against the netlist, see section 10.1.
11. [x] J11 pin 2 net in KiCad — confirmed by the netlist: J11 is correctly merged with `+5V_VOUT`, electrically powered.

---

## Links

- All build variants: https://murmulator.ru/types
