# Module Power Architecture Comparison: YD-2040-2022-V1.1 (Black Clone) vs. Official Raspberry Pi Pico

Analysis based on `RP2040 Official Pico Module — Powerchain.jpg` (section 4.4 of the Pico datasheet), `YD20402022V1.1 — Powerchain.jpg` (clone schematic) and the main Murmulator board `rp2040_black_vga.pdf`/`.net` (U1 = "YD-RP2040 TypeC Black Clone"). Complements sections 9–10 of `murmulator-vga-project-summary_EN.md` (which covers the rails at the Murmulator board level: J1/J2/J4/J5); this document covers what happens **inside the module itself** between the power connector and the 3V3 output.

---

## 1. Official Raspberry Pi Pico

**Input:** micro-USB connector (5-pin: VBUS/D+/D-/ID/GND, no CC resistors — USB 2.0 Type-B does not require CC negotiation).

**Power path:**

```
VBUS (pin 40) ──┬── R10(5.6k)+R1(10k) divider ──> GPIO24  (USB power presence sense)
                 │
                 └── D1 (MBR120VLSFT1G, Schottky SOD-323) ──> VSYS (pin 39)

VSYS (pin 39) ──> C1 (47µF) ──> U2 RT6150B-33GQW (buck-boost SMPS, VIN/VINA)
                                   │
                                   ├── EN ── R8(100k, pull-up) ── test point TP4 ── GPIO23 (SMPS mode control)
                                   ├── PS ── (Power Save select)
                                   └── LX1/LX2 ── L1 (2.2µH, 2.3A) ── VOUT ──> C2(47µF) ──> 3V3 (pin 36)
```

**Key features:**
- The diode-OR (VBUS→VSYS) is implemented with a **discrete** Schottky diode D1 (MBR120VLSFT1G) right on the Pico board.
- The regulator is a **buck-boost switching converter** RT6150B, up to 2.3 A, operating over a wide input voltage range (~1.8–5.5 V) — which is why the Pico can officially be powered from 2×AA batteries just as well as from USB.
- GPIO23 and GPIO24 are **reserved** for power management (SMPS mode / VBUS sense) and are not available as regular GPIOs on a "genuine" Pico.
- VSYS is additionally divided resistively (not shown in this fragment of the picture, but documented in the datasheet) and monitored via ADC3/GPIO29 — a "rough" battery voltmeter.

---

## 2. YD-2040-2022-V1.1 (Black Clone)

**Input:** USB Type-C (TYPEC-304-BCP16). Unlike micro-USB, Type-C **requires** resistors on the CC lines for negotiation with the host:

```
USB-C  CC1 ── R1 (5.1k) ── GND
       CC2 ── R2 (5.1k) ── GND
       D+/D- ── R3/R4 (27R) ── (signal, not power)
       VBUS ──┐
```
This is the standard UFP "sink" configuration, signaling to the host the default 5 V / up to 500 mA — already routed on the clone, nothing needs to be added separately.

**Power path:**

```
VBUS (from USB-C) ──┐
                      ├── D2 (BAT54C, dual low-power Schottky) ──> V+ ──> F1 (fuse) ──> C1(1µF) ──> U1 ME6215C33M5G
Vin (pin 39)   ───────┘                                                                                    │
                                                                                                            OUT(5) ──> C2(1µF) ──> 3V3 (pin 36)
```

The node after D2 is physically brought out to **pin 40**, the same place where VBUS is on the original Pico — but electrically it is already the combined (via the diode) node, hence the label on the clone schematic: *"Vout, pin 40 (VBUS position but 4.7V)"*. This matches what is already recorded in `murmulator-vga-project-summary_EN.md` (section 9) as the empirical ~4.7–4.8 V on the keyboard node — confirmation of the same diode drop from an independent source.

**Key features:**
- The diode-OR is implemented with a **dual signal** diode BAT54C (not a power Schottky) — lower power than D1 on the original Pico. Hence the ~150–300 mA per leg limitation already described in section 10 of the main summary.
- The regulator is a **linear LDO** ME6215C33M5G (typical current ~300 mA), not a switching buck-boost. There is no L1 inductor — less board area, but:
  - lower efficiency with a large input/output difference (heats up more at high current);
  - lower maximum output current (RT6150 — 2.3 A, typical ME6215 — usually 300–500 mA depending on package/marking);
  - narrower input range — an LDO cannot "boost", i.e. power from a source <3.6V (for example, from a single discharged Li-ion cell) will not give a stable 3.3V, whereas the Pico's buck-boost handles this.
- EN (pin 3) is not routed to a controllable GPIO — judging by the picture, the regulator is "always on" (tied statically), there is no software power-gating via GPIO on this node.
- **GPIO23 and GPIO24 are not reserved for power management** — they are broken out to the header as regular GPIO/ADC (GPIO23 is visible on connector P2, pin 17). This means:
  - firmware/code examples that on the original Pico read GPIO24 as "USB connected/not connected" or toggle GPIO23 to control the SMPS mode (Power Save vs Forced PWM) **will not work the same way** on this clone — there it is just a free I/O line, not a regulator service signal;
  - on the other hand, the clone has one more useful GPIO/ADC available to the user circuit than the original Pico.
- A VSYS→ADC3 divider (supply voltage monitoring) is not visible in this fragment of the clone schematic — apparently not implemented (unlike the Pico, where R2/R5/R6+ADC3 provide such monitoring as standard).

---

## 3. Cross-check with the Main Murmulator Board (`rp2040_black_vga.pdf`, U1)

The U1 pin numbering in the main schematic ("YD-RP2040 TypeC Black Clone", 40-pin DIP profile) matches the pinout in the clone picture:

| Net on the Murmulator board | U1 pin | Correspondence on the clone Powerchain picture |
|---|---|---|
| `Vin` | 39 | Vin (39) — input before diode D2/BAT54C |
| `Vout` (→ `+5V_VOUT`) | 40 | Vout (40) — node after D2, physically in the place of the original Pico's VBUS |
| `+3V3` | 36 | 3V3 (36) — output of the ME6215C33M5G regulator |
| `RUN` | 30 | RUN (pin 9 on the P2 header in the picture) |

The match confirms that the module on the board is indeed a clone of exactly this revision (YD-2040-2022-V1.1), not a different pin layout — a useful check before soldering.

**Practical conclusion for the project:** on the Murmulator board (sections 9–10 of `murmulator-vga-project-summary_EN.md`) external power is collected on the `+5V_RAW` rail (J1 — DC jack, J12 — Micro USB via D1 1N5819), and the J13 selector allows feeding this rail either to Vin (default, through BAT54C) or directly to Vout — **bypassing** the internal low-current BAT54C. This bypass is provided precisely because on the clone (unlike the original Pico with its power MBR120) the diode-ORing is low-power by design — this is an architectural difference of the module, not a peculiarity of the Murmulator board itself. The second practical point is that the linear ME6215C33M5G regulator is weaker (in current and efficiency) than the original's RT6150 buck-boost, so if in the future peripherals drawing noticeable current specifically from `3V3` (pin 36, not from `+5V_VOUT`) are added to the perfboard, keep in mind the more modest current budget of this node compared to a "real" Pico.

---

*Sources: RP2040 Official Pico Module — Powerchain.jpg (section 4.4 of the Pico datasheet), YD20402022V1.1 — Powerchain.jpg, rp2040_black_vga.pdf/.net (U1).*
