# Building a VGA Murmulator on Perfboard

## 1. General Concept

Murmulator is an adapter board from the Raspberry Pi Pico to peripherals (SD card, VGA, PS/2 keyboard, joystick, audio boot).

**Connector labels on the board:**

| Connector | Purpose |
|---|---|
| J1 | External power (PSU), DC Jack |
| J2 | "Raw" +5V tap from the PSU (before D1) — for external peripherals, e.g. BlueRetro |
| J3 | VGA (12-pin header, Conn_02x06) |
| J4 | Keyboard (PS/2, Mini-DIN-6) |
| J5 | External connector |
| J6 | Joystick 1 (DE-9) |
| J7 | Joystick 2 (DE-9) |
| J8 | Audio out |

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

### 2.1. Pull-up Resistors on the SD Module Lines

After removing the 74VHCT125A, two SPI lines need to be separately checked/provided with pull-ups:

| Line | Pull-up needed? | Value | Why |
|---|---|---|---|
| **CS** | Yes, mandatory | ~10 kΩ to 3.3V | With the Pico's GPIO in an undefined state at startup, the CS line must remain high (deselected), otherwise the card may interpret noise on MOSI/CLK as commands |
| **MISO (DO)** | Recommended | ~10–50 kΩ to 3.3V | When CS is inactive, the card's output goes Hi-Z — a pull-up keeps the line from "floating" and picking up interference |
| **MOSI, SCK** | Not required | — | Always actively driven by the master (Pico), never go Hi-Z |

**What to check on a specific module:**
Some cheap SD modules route pull-ups right next to the 74VHCT125A buffer. After removing it and installing jumpers, the resistors may have:
1. remained in the circuit independently of the buffer — then everything is fine;
2. disappeared along with the buffer footprint — then they need to be added separately.

**Verification procedure:** measure resistance with an ohmmeter between CS and 3.3V, and between MISO and 3.3V (do this together with the short-circuit check after adding jumpers in section 2). A finite resistance (10–50 kΩ) means the resistor is present. An open circuit means you need to solder 10 kΩ resistors from CS and MISO to the 3.3V bus on the perfboard.

*Symptom of a missing CS pull-up: the card is detected unreliably at startup ("works every other time"), which is hard to diagnose after the fact.*

---

## 3. VGA (J3)

- A resistor R2R ladder on digital Pico GPIOs — a passive circuit, requires no separate power supply.
- Rather than buying a VGA connector, it's easier to cut a ready-made shielded cable off an old video card (often it has the connector, just not populated).
- Solder the resistors compactly, right against the cable/connector, with short leads — critical for image cleanliness (interference).

---

## 4. Keyboard (PS/2, J4)

**Connector:** J4 — Mini-DIN-6 (PS/2).

### 4.1. Keyboard Power

The keyboard is powered **separately from the main 3.3V peripheral supply**, directly from **Vout (pin 40 of the YD-RP2040)** — the node after the BAT54C diode-OR, not from 3V3(OUT). For details, see section 9 ("Board power supply").

| Parameter | Value |
|---|---|
| Source | Vout (after BAT54C), not 3V3(OUT) |
| Nominal drop | ~0.3–0.45V relative to the 5V input |
| Resulting voltage on pin 4 of the Mini-DIN-6 | ~4.7–4.8V (within PS/2 spec tolerance) |
| Keyboard current draw | ~100 mA (must be accounted for in the BAT54C current budget, see section 10) |

### 4.2. Signal Lines (5V → 3.3V Level Shifting)

PS/2 CLK and DATA are 5V logic on the keyboard side, while the Pico's GPIOs are 3.3V and not 5V-tolerant. A **passive circuit** is used for level matching: a pull-up resistor + series resistor + zener clamp on each line.

**Signal mapping (per the reference MURMULATOR_classical_scheme, KiCad):**

| Signal (raw, 5V, from keyboard) | After zener clamp | Pico GPIO |
|---|---|---|
| PS2 CLK (Mini-DIN-6, pin 5) | PS2_CLK_3V | GP0 |
| PS2 DATA (Mini-DIN-6, pin 1) | PS2_DATA_3V | GP1 |

**Actual line circuit (confirmed from the `schematics_rp2040_black_vga.net` netlist):**

```
J4 (5V raw) ──[R9/R10, 1 kΩ, pull-up to +5V_VOUT]── CLK/DATA node ──[R11/R12, 100 Ω]── GPIO (GP0/GP1) ──[D2/D3, zener]── GND
```

| Element | Value | Role |
|---|---|---|
| R9 (CLK) / R10 (DATA) | 1 kΩ | Pull-up of the raw 5V line to +5V_VOUT — mandatory, since the keyboard output is open-collector |
| R11 (CLK) / R12 (DATA) | 100 Ω | Series resistor between the raw line and the GPIO — limits current through the zener during breakdown, and forms an RC filter together with the GPIO input capacitance |
| D2 (CLK) / D3 (DATA) | 3.3V | Cathode on GPIO, anode on GND — clips the upper level of the signal at the GPIO |

Circuit: the raw 5V signal (already pulled up to +5V_VOUT by R9/R10) passes through a series resistor R11/R12 (100 Ω) → the zener clamp limits the upper level right at the GPIO → the signal reaches GP0/GP1.

**Circuit limitation:** this is a passive level converter without active edge restoration. The circuit is sensitive to:
- keyboard cable capacitance,
- resistor/zener value spread between individual units,
- crosstalk from neighboring signals (in particular from the VGA R2R ladder — see section 3, which notes that VGA resistors must be kept compact and tight against the connector specifically to minimize such interference).

### 4.3. Known Issue: Signal Corruption on Multi-byte Scancodes

**Symptom:** in some games (observed in Lode Runner), certain keys occasionally produce incorrect characters or extraneous combinations (CPS, CS, SYM prefixes, or numpad digits instead of the expected character).

**Diagnosis:** this is not a key-mapping bug in the firmware, but a PS/2 signal integrity issue — bit-level corruption of the service byte prefixes (E0) of extended/multi-byte scancodes. The main suspect is the passive resistor-zener circuit on CLK/DATA (section 4.2), which provides no active edge restoration and is sensitive to the factors listed above.

**Diagnostic steps (by priority):**

| Step | Tool | What to check |
|---|---|---|
| 1 | Oscilloscope / logic analyzer | CLK/DATA edge shape, jitter, interference from VGA |
| 2 | Multimeter/values | Match of actual R9/R10 (1 kΩ), R11/R12 (100 Ω), and zener values against the design values |
| 3 | Visual inspection of routing | Physical placement of the PS/2 lines relative to the VGA R2R resistors (risk of crosstalk) |
| 4 | Multimeter under load | Stability of the 5V keyboard power line during activity |
| 5 | Keyboard replacement | Rule out a defect/quirk specific to this keyboard unit |
| 6 | Firmware | Check UART debug output / parity error counters, if the firmware provides them |

**Note on the "quick fix" involving the pull-up:** it was previously assumed that lowering the pull-up value from ~10 kΩ to 4.7 kΩ would speed up the edges. **In fact, the pull-up (R9/R10) in the circuit is already 1 kΩ** — significantly stronger than assumed, and already stronger than the "target" 4.7 kΩ. Lowering the pull-up further in this configuration makes no sense and is not a relevant direction for the fix.

**More relevant directions for a practical experiment without measurement equipment:**
- **Check/reduce the value of the series resistors R11/R12 (100 Ω).** These, together with the GPIO input capacitance, form the RC filter that may be rounding off the edges of the E0 prefixes when keyboard cable capacitance is elevated — a more likely culprit than the pull-up.
- **Check the actual value of R9/R10** — if on the specific board they differ from 1 kΩ (e.g. due to an assembly error or component batch), this should also be ruled out with a multimeter.

### 4.4. Open Questions

- [ ] Oscilloscope check of CLK/DATA (similar to what has already been done for BlueRetro in section 5.1).
- [ ] Check the physical routing of the PS/2 lines relative to the VGA R2R ladder for crosstalk.
- [ ] Measure the actual values of R9/R10 (expected 1 kΩ) and R11/R12 (expected 100 Ω) on the assembled board — rule out an assembly error/component batch issue.
- [ ] Test with a reduced value of R11/R12 (100 Ω → lower) as a first practical step — a more likely point of influence on the edges than the pull-up.
- [ ] Test with a different PS/2 keyboard to rule out a hardware defect specific to this unit.

---

## 5. Joysticks (Dendy, 2 Ports, DE-9, J6/J7)

**Connector labels on the board:** Joystick 1 — **J6**, Joystick 2 — **J7**.

Dendy-type joysticks are used (9-pin, narrow connector) — internally containing an active shift register chip (CD4021 type or equivalent), not just mechanical contacts.

**DE-9 pinout (Dendy 9-pin):**

| DE-9 pin | Purpose |
|---|---|
| 2 | DATA — shift register output |
| 3 | LATCH — latch/counter reset |
| 4 | CLOCK — shift clock pulse |
| 6 | +5V — standard power for the register chip |
| 8 | GND |

**Why the register is powered from 3.3V instead of the standard 5V:**
The joystick's DATA output follows its supply voltage level. When powered from 5V, the DATA line becomes 5V logic, and the RP2040's GPIOs are not 5V-tolerant (max ~3.6V) — a protective resistor (~1 kΩ) on DATA would be needed as a crude current limiter into the port's protection diode. The register chips (CD4021-like) are standard CMOS logic with a wide supply range (typically 3–18V), so powering them from 3.3V instead of 5V is a normal, safe solution: DATA comes out as 3.3V logic right away, no resistor is needed, and the connection to the GPIO is direct.

**Final connection scheme (2 ports):**

| Signal | Joystick 1 (DE-9 pin) | Joystick 2 (DE-9 pin) | Pico GPIO |
|---|---|---|---|
| DATA | 2 | 2 | GP16 (J6) / GP17 (J7) — separate |
| LATCH | 3 | 3 | GP15 — shared for both ports |
| CLOCK | 4 | 4 | GP14 — shared for both ports |
| VCC | 6 | 6 | 3.3V — shared bus |
| GND | 8 | 8 | GND — shared bus |

CLOCK and LATCH are shared between both joysticks (both registers are clocked and latched synchronously); DATA lines must be separate per port, otherwise the outputs would conflict on the same line.

**Open question:** need to cross-check against the GPIO pinout expected by the specific firmware in use (pico-nes, murmulator-os, etc.) "out of the box" — if the firmware's default pins differ from GP14/15/16/17, the joysticks will be physically wired correctly but won't respond without a config edit/firmware rebuild.

### 5.1. BlueRetro Integration (Bluetooth Gamepads Instead of Physical Dendy Joysticks)

> 📄 **Full details of the BlueRetro build** (board selection, input power protection, BOOT button, LED indication, firmware version, oscilloscope verification of signals) are covered in a separate document, **`blueretro-murmulator-project-summary.md`**. Below is a summary aligned with that document, sufficient for the context of the main Murmulator build.

There are no physical Dendy joysticks in the project — both DE-9 ports (J6/J7) were designed from the start as **virtual**, served by a single DIY-built **BlueRetro** adapter (ESP32, firmware from github.com/darthcloud/BlueRetro — the repository was archived by its owner on 12/14/2025, no more active development, but firmware releases and documentation remain available).

**Board used: ESP32 D1 mini (MH-ET LIVE, CH9102 USB-serial chip).** Chosen after comparison with the 30-pin WROOM DevKit (ruled out — GPIO0 is not physically broken out, which is needed for BlueRetro's BOOT/pairing) and the 38-pin WROOM DevKit (technically suitable and matches the BlueRetro author's official recommendation, but not chosen — would have required reworking the already-finished KiCad schematic). A known trade-off — the D1 mini's onboard AMS1117 heats up to ~54°C, close to the PLA enclosure's softening temperature; the cooling solution has been deliberately deferred (risk accepted). See `blueretro-murmulator-project-summary.md` for selection details.

**Bluetooth gamepad compatibility:** BlueRetro operates over standard Bluetooth HID (BR/EDR and LE) and supports PS3/PS4/PS5, Xbox One, Xbox Series X|S, Wii/Switch, and generic HID BT devices.

**Signal protocol is identical to a physical Dendy joystick:** Dendy, NES/Famicom, and BlueRetro's output in FC/NES mode all use the same shift-register principle (CLOCK/LATCH/DATA), so the Murmulator cannot tell a physical joystick apart from an emulated BlueRetro one.

**Signal direction:** the Nintendo PISO protocol (Parallel-In, Serial-Out) — **the console (in this case the Murmulator/RP2040) generates CLOCK and LATCH**, while the controller (a physical Dendy register or a BlueRetro emulating it) is the passive side that only responds to them with a **DATA** signal. Therefore, on the BlueRetro ESP32 the OUT0 (LATCH) and CUP (CLOCK) signals are **inputs** receiving a signal from the Pico, and the only signal the ESP32 itself generates is **DATA**.

**Mapping ESP32 (BlueRetro, FC/NES mode) → DE-9 → Pico GPIO:**

| Signal | Direction | ESP32 pin (BlueRetro) | System pin (standard NES 7-pin) | DE-9 J6 pin | DE-9 J7 pin | Pico GPIO |
|---|---|---|---|---|---|---|
| LATCH (shared, OUT0) | Pico → ESP32 (input) | IO32 | 3 | 3 | 3 | GP15 |
| CLOCK port 1 (P1_CUP) | Pico → ESP32 (input) | IO5 | 2 | 4 | — | GP14 |
| CLOCK port 2 (P2_CUP) | Pico → ESP32 (input) | IO18 | 2 | — | 4 | GP14* |
| DATA port 1 (P1_D0) | ESP32 → Pico (output) | IO19 | 4 | 2 | — | GP16 |
| DATA port 2 (P2_D0) | ESP32 → Pico (output) | IO22 | 4 | — | 2 | GP17 |
| GND | — | ESP32 GND | 1 | 8 | 8 | GND |
| VCC (+5V, NES pin 7) | — | — | 7 | 6 | 6 | not connected |

*\*CLOCK for J6 and J7 is the same signal from a single Pico output (GP14), physically routed to two ESP32 inputs (IO5 and IO18). Since there's a single source (Pico, master) and the ESP32 is only a receiver (input) in both cases, this is a standard, safe "one driver — multiple receivers" topology: there can be no bus conflict or signal desync, and no separate oscilloscope check is needed before combining the lines.*

**VCC (pin 6, DE-9) is not used:** the ESP32 inside BlueRetro natively runs on 3.3V logic (matching the already-adopted decision to power the joystick node from 3.3V — see above), so the CLOCK/LATCH/DATA signal lines connect directly to the Pico GPIOs, without level-shifters (which in official BlueRetro builds are only needed to interface with a real 5V NES console).

**Power for the ESP32 (BlueRetro) module itself is separate from the DE-9 port power.** DE-9 pin 6 (VCC) on both ports stays at **3.3V**, as originally decided for physical Dendy joysticks (see above) — BlueRetro's power is not routed through the signal connector, to preserve port compatibility with a physical joystick in the future.

The ESP32 module (**a DevKit with an onboard AMS1117 regulator**, 5V→3.3V on the module itself) is powered from the main board via connector **J2** (JST-XH, "raw" +5V tap from the PSU, before diode D1 of the main board — see section 10) → wire → connector **J3** on the BlueRetro board. On the BlueRetro board itself, J3 is the standard power input, already protected by its own Schottky diode **D1 = 1N5819** (placed between J3 and the ESP32's 5V input, U1 pin 35). Result: two independent legs from the "raw" PSU, each with its own Schottky diode — one to the main board (keyboard/Vout), the other to BlueRetro (ESP32) — with no overlap of current budgets and no load on the YD-RP2040 module's onboard BAT54C. The regulator on the DevKit board itself steps the incoming ~4.5–4.6V (after the diode drop) down to 3.3V for the ESP32; no separate external regulator is needed on the perfboard.

**⚠️ Limitation: BlueRetro/joysticks only work with an external PSU connected.** The "external PSU 5V bus" line physically exists only when an external power supply is connected to Vin — unlike Vout (the node after BAT54C), where both sources (USB and PSU) converge and which powers the keyboard independently of how the board is powered. The YD-RP2040 has no accessible "raw" USB VBUS before the diode (unlike the original Pico, where VBUS is broken out as a separate pin) — USB is routed through the same BAT54C as Vin, and only the combined Vout is broken out externally. So when the board is powered by USB alone (without an external PSU), the ESP32/BlueRetro and, accordingly, both joysticks will not work. Full Murmulator operation with BlueRetro joysticks requires an external PSU to be connected.

**Why not powered from Vout (pin 40) of the RP2040:** Vout on the YD-RP2040 is the node right after the diode-OR formed by **BAT54C**, a signal diode with a per-leg current limit of ~150–300 mA (see sections 9–10). This node is already partially loaded: the PS/2 keyboard (section 4) is powered from Vout and draws roughly ~100 mA at idle, with possible spikes during activity. An ESP32 with an active Bluetooth radio would add another peak of 200–300 mA through the same diode — the combined current would almost certainly exceed the BAT54C rating, especially if peaks coincide (a keypress + a BT packet transmission at the same time). That's why the ESP32 is powered directly from the 5V bus, bypassing Vin/Vout/BAT54C, without sharing that node with the keyboard.

**ESP32 peak consumption — measured:** a multimeter recorded **~155 mA** on the 5V bus with the BT radio active. Since the ESP32 is powered via J2 (main board) → J3 (BlueRetro), bypassing BAT54C and the Vout node, this load does not share a current budget with the PS/2 keyboard and is not constrained by the BAT54C rating (~150–300 mA per leg) — there is sufficient current headroom. This open item is now closed.

**LED status indication (implemented):** pin **IO17** — global status LED (pairing/error, blue, two-resistor fail-safe circuit), plus two port-status LEDs on **IO2/IO4** (green, via 2N7000 MOSFETs, since IO2/IO4/IO17 are ESP32 strapping pins). Logic: during pairing, the global LED and the LED of the first free port pulse; when a gamepad connects, the corresponding port LED lights solid. Full circuit, resistor values, and measured Vf are in `blueretro-murmulator-project-summary.md`.

**Verified with an oscilloscope (consistent with the table above):** CLOCK (GP14→IO5/IO18) — 12.5 kHz, 50% duty cycle; LATCH (GP15→IO32) — 1200 µs period, 40 µs pulse; DATA — inverse logic (LOW = button pressed), 80 µs pulse per clock. The canonical NES bit order (A, B, Select, Start, Up, Down, Left, Right) was confirmed for an Xbox One gamepad. The "one driver (Pico) — receivers (ESP32)" topology on the combined CLOCK lines was electrically confirmed, with no bus conflict.

**Firmware:** `v25.04_hw1.zip` (HW1 spec, file `BlueRetro_hw1_nes.bin`), flashed via `pio pkg exec -- esptool.py`. Input power protection for the BlueRetro module (connector J3, diode D1=1N5819, RESET button SW2, decoupling capacitors) and details of the BOOT/IO0 button (external pull-up R1=10kΩ) — see `blueretro-murmulator-project-summary.md`.

---

## 6. Audio Output (audio out, J8)

**Connector:** J8 — output audio jack (L/R). **J8B (OUT_A)** — a duplicate header (PinHeader), wired in parallel with J8, for convenient connection of the audio output via a separate wire/cable (e.g. if the enclosure/wiring doesn't allow using the jack directly, or for test measurements).

**J8B (OUT_A) pinout:**

| J8B pin | Signal | Corresponds to |
|---|---|---|
| 1 | L | same node as L on jack J8 |
| 2 | GND | common ground (same node as S on jack J8) |
| 3 | R | same node as R on jack J8 |

**GPIO assignment:**

| Net | Pico GPIO | Channel |
|---|---|---|
| LEFT_OUT | GP27 | Left channel (game audio) |
| RIGHT_OUT | GP26 | Right channel (game audio) |
| BEEP_OUT | GP28 | Beeper / TAP loading indicator, mixed into both channels |

**Circuit (main L/R channels):**

| Channel | Series resistor (GPIO → node) | Decoupling capacitor (node → jack) | Node bias resistor to GND |
|---|---|---|---|
| L | R21, 1 kΩ | C4, 10 µF | R22, 330 Ω |
| R | R25, 1 kΩ | C6, 10 µF | R26, 330 Ω |

**Mixing the beeper (BEEP_OUT, GP28) into both channels:**

| Element | Value | Role |
|---|---|---|
| R23 | 2 kΩ | Summing BEEP_OUT → node L (via C5) |
| R24 | 2 kΩ | Summing BEEP_OUT → node R (via C7) |
| C5 | 10 nF | Couples node L to the BEEP mixing bus |
| C7 | 10 nF | Couples node R to the BEEP mixing bus |

**Important design detail:** the beeper is mixed in deliberately much quieter than the game audio — the decoupling capacitors in the BEEP path are three orders of magnitude smaller (10 nF vs. 10 µF on the main channel), and the summing resistors (2 kΩ) are higher than the main channel's series resistors (1 kΩ). This is an intentional circuit decision (a quiet indicator signal layered over the main audio), not an error/imbalance.

**Connecting the audio module to the main board — 4 wires:**

| Wire | Purpose |
|---|---|
| RIGHT_OUT → GP26 | signal |
| LEFT_OUT → GP27 | signal |
| BEEP_OUT → GP28 | signal |
| GND → board's common GND bus | mandatory connection, without which the module has no reference point for the signals |

---

## 7. Audio Input (audio in)

*Reserved for an audio input (e.g. for loading from cassette tape) — details to be added.*

---

## 8. External Connector (J5)

*Reserved for an external connector — details to be added.*

---

## 9. Board Power: Vin/Vout Architecture and USB-only Mode

**Module used: YD-RP2040** (a Raspberry Pi Pico clone). This board has **no VBUS/VSYS pins** in the usual sense — instead it breaks out **Vin (pin 39)** and **Vout (pin 40)**, and the diode-OR combining of sources is already implemented on the module itself via a dual Schottky diode, **BAT54C**.

- No separate power supply is required in the base circuit — the whole device is powered through the module itself.
- Input: microUSB on the module (5V) **or** the **Vin** pin (analogous to VSYS on the original Pico).
- The onboard converter on the module outputs a stable **3.3V** on pin **3V3(OUT)**.
- Main peripherals on the perfboard are powered from 3V3(OUT): the reworked SD module, the joystick (VGA needs no power — passive circuit).
- **The PS/2 keyboard (J4) is powered separately — 5V, taken from the Vout pin** (the node after the BAT54C diodes, not from 3V3(OUT)). This is the standard supply voltage for the PS/2 protocol, so the keyboard's power line is taken before the 3.3V regulator, not after it.
  - The Data and Clock lines remain 5V logic on the keyboard side, while the module's GPIOs are 3.3V, so the signal lines need a step-down stage (voltage divider or zener clamps), separate from the power supply question.

### Vin / Vout Instead of VBUS / VSYS

On the YD-RP2040, the diode-OR between USB and external power is already routed on the board:

```
USB (VBUS) ──[diode 1 in BAT54C]──┐
                                    ├──> Vout (pin 40) ──> [regulator] ──> 3.3V
Vin (pin 39) ──[diode 2 in BAT54C]─┘
```

| Pin | What it is | Notes |
|---|---|---|
| **Vout (pin 40)** | Analogous to VSYS — the node after the diodes, input to the 3.3V regulator | Voltage = whatever the active source actually supplies, minus the drop across the BAT54C diode |
| **Vin (pin 39)** | Ready-made input for external power | Already connected through the BAT54C diode to Vout — no separate Schottky diode needed |

**Important:** as with VBUS/VSYS on the original Pico, if the module isn't always connected via USB, you cannot take 5V for peripherals directly from the USB line — while powered from Vin, that line will be unpowered.

**USB-only power mode:** since Vout combines USB and Vin through the BAT54C, all peripherals powered from Vout/3V3(OUT) — the keyboard (section 4), the SD module (section 2), and physical joysticks (section 5) — work when the board is powered by USB alone, without an external PSU. The exception is **BlueRetro**: it's powered via J2, a separate "raw" tap that is physically connected only to the Vin/PSU line and is not combined with USB through the BAT54C, so it requires an external PSU to be connected (see section 5.1 and section 10 for details).

---

## 10. External Power (J1/J2): Diode D1 and Separate Rails for Peripherals

For the YD-RP2040, an additional Schottky diode for powering the module itself is not needed — the diode-OR between USB and the external PSU is already implemented on the module via BAT54C. Nevertheless, an external Schottky diode **D1 (1N5819)** has been added to the circuit, connected in parallel with the onboard BAT54C, between the "raw" +5V from the PSU and the `+5V_VOUT` node (Vout/pin 40 + keyboard J4). Confirmed from `schematics_rp2040_black_vga.net`.

**Connection (current circuit):**
```
External PSU 5V(+) ──> J1 (DC Jack), pin 1 ──┬── D1 (1N5819), anode      (raw +5V, before the diode)
                                               ├── J2 (JST-XH), pin 2      (additional raw +5V tap — for external peripherals)
                                               └── J5, pin 39 (Vin RP2040)

D1 (1N5819), cathode ──> +5V_VOUT node ──┬── U1 Vout (pin 40)
                                          └── J4, pin 4 (PS/2 keyboard, 5V)

External PSU GND ──> J1, pin 2 / board common GND
```

**Purpose of D1:** reinforces/backs up the path from the PSU to the `+5V_VOUT` node, bypassing the low-current BAT54C, removing the current limit (~150–300 mA) for the keyboard.

**Purpose of J2:** a separate "raw" +5V tap (before D1, without BAT54C's limits) — used to power external peripherals, in particular the **BlueRetro** adapter (see section 5.1), which already has its own Schottky diode D1 = 1N5819 on its J3 input.

**Details and limitations:**
- BAT54C is a low-power signal-grade dual diode, not a power diode. The typical maximum forward current per leg is roughly **150–300 mA** (verify against the datasheet of the specific part marking on the board).
- Peripherals with notable current draw (e.g. the BlueRetro Bluetooth module) should be powered via J2 (raw +5V, bypassing BAT54C and D1), not from Vout.
- External PSU GND goes directly to the board's common GND, without a diode.

*Note: the original Raspberry Pi Pico (not the YD-RP2040) has no such built-in point for external power — there, the diode-OR is implemented only between USB and VSYS, and for an external PSU you would need to add your own Schottky diode on VSYS by hand, as described in the Pico datasheet, section 4.5 "Powering Pico".*

---

## 11. Assembly Order Checklist

1. [ ] Rework the SD module (desolder AMS1117 + 74VHCT125A, add jumpers).
2. [ ] Dry-fit component layout on the perfboard per the block diagram from the base version's documentation (see section 1).
3. [ ] Route GND and 3.3V rails along the board edges as separate traces/wires.
4. [ ] Solder in the Pico — preferably onto pin headers, not directly (for ease of replacement/reflashing).
5. [ ] Connect the SD module's SPI, VGA R2R resistors, PS/2 keyboard, and joystick to the corresponding Pico GPIOs.
6. [x] Schottky diode D1 (1N5819) from the PSU (J1/J2) to the `+5V_VOUT` node — implemented in the schematic, see section 10.
7. [ ] Check the placement of connectors (VGA, SD, audio, joystick) against your enclosure before final assembly.

---

## Links

- All build variants: https://murmulator.ru/types
