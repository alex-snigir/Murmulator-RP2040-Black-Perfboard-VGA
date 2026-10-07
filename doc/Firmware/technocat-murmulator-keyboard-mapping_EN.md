# Murmulator VGA — PS/2 → ZX Spectrum Keyboard Mapping

Reference table of modifier and SYMBOL SHIFT combination mapping for a PS/2 keyboard on Tecnocat firmware v0.96.20.

---

## Modifiers

| PC key (PS/2) | Scan code (make) | Function on ZX Spectrum | E0 prefix? |
|---|---|---|---|
| Left **Shift** | `0x12` | CAPS SHIFT | No |
| Right **Shift** | `0x59` | CAPS SHIFT | No |
| Left **Ctrl** | `0x14` | SYMBOL SHIFT | No |
| Left **Alt** | `0x11` | unverified | No |
| Right **Ctrl** | `E0 0x14` | SYMBOL SHIFT (confirmed to duplicate Left Ctrl) | Yes |
| Right **Shift** | `0x59` | Backspace/Delete (observed behavior, not CAPS SHIFT) | No |
| Right **Alt** | `E0 0x11` | Backspace/Delete (observed behavior) | Yes |

**Note:** Right Shift (without the E0 prefix, scan code `0x59`) and Right Alt (with the E0 prefix) consistently and predictably produce Backspace/Delete instead of the expected CAPS SHIFT duplication / no function. The behavior is deterministic (not random), which points to explicit mapping of specific scan codes in the Tecnocat firmware, rather than signal corruption or a general E0-prefix attribute. Requires checking in the firmware sources.

---

## SYMBOL SHIFT Combinations (Left Ctrl + key)

Tested in practice in **48 BASIC** (K cursor) — all 36 combinations work.
In **128 BASIC** (blue blinking cursor) — confirmed that all combinations except the keyword tokens (Ctrl+Q/W/E/Y/U/I/A/S/D/F/G) work correctly. These 11 tokens **are not inserted**; presumably a quirk/shortcoming of the 128-mode emulation, not related to the PS/2 signal.

| Spectrum key | Symbol / token | PS/2 combination | 48 BASIC (K cursor) | 128 BASIC (blue cursor) |
|---|---|---|---|---|
| SYM SHIFT + 1 | `!` | Ctrl + 1 | ✅ | ✅ |
| SYM SHIFT + 2 | `@` | Ctrl + 2 | ✅ | ✅ |
| SYM SHIFT + 3 | `#` | Ctrl + 3 | ✅ | ✅ |
| SYM SHIFT + 4 | `$` | Ctrl + 4 | ✅ | ✅ |
| SYM SHIFT + 5 | `%` | Ctrl + 5 | ✅ | ✅ |
| SYM SHIFT + 6 | `&` | Ctrl + 6 | ✅ | ✅ |
| SYM SHIFT + 7 | `'` | Ctrl + 7 | ✅ | ✅ |
| SYM SHIFT + 8 | `(` | Ctrl + 8 | ✅ | ✅ |
| SYM SHIFT + 9 | `)` | Ctrl + 9 | ✅ | ✅ |
| SYM SHIFT + 0 | `_` | Ctrl + 0 | ✅ | ✅ |
| SYM SHIFT + Q | `<=` (token) | Ctrl + Q | ✅ | ❌ |
| SYM SHIFT + W | `<>` (token) | Ctrl + W | ✅ | ❌ |
| SYM SHIFT + E | `>=` (token) | Ctrl + E | ✅ | ❌ |
| SYM SHIFT + R | `<` | Ctrl + R | ✅ | ✅ |
| SYM SHIFT + T | `>` | Ctrl + T | ✅ | ✅ |
| SYM SHIFT + Y | `AND` (token) | Ctrl + Y | ✅ | ❌ |
| SYM SHIFT + U | `OR` (token) | Ctrl + U | ✅ | ❌ |
| SYM SHIFT + I | `AT` (token) | Ctrl + I | ✅ | ❌ |
| SYM SHIFT + O | `;` | Ctrl + O | ✅ | ✅ |
| SYM SHIFT + P | `"` | Ctrl + P | ✅ | ✅ |
| SYM SHIFT + A | `STOP` (token) | Ctrl + A | ✅ | ❌ |
| SYM SHIFT + S | `NOT` (token) | Ctrl + S | ✅ | ❌ |
| SYM SHIFT + D | `STEP` (token) | Ctrl + D | ✅ | ❌ |
| SYM SHIFT + F | `TO` (token) | Ctrl + F | ✅ | ❌ |
| SYM SHIFT + G | `THEN` (token) | Ctrl + G | ✅ | ❌ |
| SYM SHIFT + H | `↑` (power) | Ctrl + H | ✅ | ✅ |
| SYM SHIFT + J | `-` | Ctrl + J | ✅ | ✅ |
| SYM SHIFT + K | `+` | Ctrl + K | ✅ | ✅ |
| SYM SHIFT + L | `=` | Ctrl + L | ✅ | ✅ |
| SYM SHIFT + Z | `:` | Ctrl + Z | ✅ | ✅ |
| SYM SHIFT + X | `£` | Ctrl + X | ✅ | ✅ |
| SYM SHIFT + C | `?` | Ctrl + C | ✅ | ✅ |
| SYM SHIFT + V | `/` | Ctrl + V | ✅ | ✅ |
| SYM SHIFT + B | `*` | Ctrl + B | ✅ | ✅ |
| SYM SHIFT + N | `,` | Ctrl + N | ✅ | ✅ |
| SYM SHIFT + M | `.` | Ctrl + M | ✅ | ✅ |
| SYM SHIFT + SPACE | `BREAK` | Ctrl + Space | unverified | unverified |

Legend: ✅ confirmed working, ❌ confirmed not working, — not tested in this mode.

---

## Diagnostic Conclusions

1. **All single-byte (non-E0) scan codes are transmitted correctly** — 26+ confirmed Ctrl+[digit/letter/symbol] combinations work reliably in 48 BASIC. This confirms that basic PS/2 transmission of modifiers and alphanumeric keys does not suffer from a signal integrity problem.

2. **Right Shift (`0x59`, non-E0) and Right Alt (`E0 0x11`) produce Backspace/Delete** instead of the expected behavior — deterministically, not randomly. Requires checking the Tecnocat firmware sources: possibly this is intentional mapping of specific scan codes. At the same time, **Right Ctrl (`E0 0x14`) works correctly and reliably** duplicates SYMBOL SHIFT. Since the "problem" keys include both an E0 code (Right Alt) and a non-E0 code (Right Shift), the pattern **is not related to the presence of the E0 prefix as such** — this strengthens the hypothesis of **intentional/specific mapping** of particular scan codes in the firmware, rather than a hardware or logic problem tied to the class of E0 sequences.

3. **The "garbage" symbols problem in Lode Runner** (CPS, CS, SYM prefixes, numpad digits) — **the cause has been found, and it is not signal integrity.** The problem was originally attributed to multi-byte E0 scan codes (arrows, Insert, Delete, etc.), but the oscillograms showed clean transmission of E0 sequences, and the Lode Runner test (see "Open Questions") confirmed that the CPS prefix is added by the firmware's `[KBD>Cursor]` mode, which silently presses CAPS SHIFT on every arrow (see "Finding from the Firmware Sources").

4. **128 BASIC vs 48 BASIC**: the difference in handling SYMBOL SHIFT tokens is a quirk/limitation of the 128 editor emulation in Tecnocat, not related to the PS/2 signal. A candidate for separate investigation (not a priority for the current PS/2 diagnostics).

5. **Arrows and extended (numpad) keyboard digits work correctly in 128 BASIC** (blue cursor) — all 4 arrows and numpad digit keys are reproduced without distortion in this mode. This does not contradict the problem in Lode Runner: the distortions there were observed in a different context (a game, not the 128 BASIC editor), so the discrepancy may be related either to the specific application/screen mode or to a different keyboard polling rate in the game loop compared to the BASIC editor.

---

## Finding from the Firmware Sources (v0.96.20, main.c)

**Source:** github.com/MadedCat/Murmulator_rp2040, tag `murmulator_rp2040_v0.96.20`

By default (`DEF_CFG_KBD_MODE = 0`, `kbd_lock = false`) the firmware has the **"Keyboard maps to Cursor joystick"** mode active — joystick emulation via the keyboard, working **in parallel** with normal input (main.c, ~line 4844):

```c
if(now_kbd_mode==0){ // Cursor joystick emulation
    if (KBD_UP)    {kb_data[0]|=(1<<0); kb_data[4]|=(1<<3);}; // CAPS+7
    if (KBD_DOWN)  {kb_data[0]|=(1<<0); kb_data[4]|=(1<<4);}; // CAPS+6
    if (KBD_LEFT)  {kb_data[0]|=(1<<0); kb_data[3]|=(1<<4);}; // CAPS+5
    if (KBD_RIGHT) {kb_data[0]|=(1<<0); kb_data[4]|=(1<<2);}; // CAPS+8
    if((KBD_R_ALT)||(KBD_R_SHIFT)||(KBD_NUM_PERIOD)||(KBD_DELETE)){
        kb_data[0]|=(1<<0); kb_data[4]|=(1<<0);
    }; // CAPS+0 = "Fire"
}
```

**Interpretation of the finding:**

1. **Right Alt, Right Shift, NUM_PERIOD (numpad `.`) and Delete** are four different physical ways to press the **"Fire"** button of the virtual cursor joystick
2. "Fire" is translated into **CAPS SHIFT + 0**, which on a real ZX Spectrum is the **DELETE** key
3. This fully explains the previously observed behavior: Right Alt/Right Shift produced "Backspace/Delete" not because of a firmware bug or a signal problem, but because they literally sent CAPS+0

**Important side effect for Lode Runner diagnostics:** this mode is enabled **by default** and works simultaneously with the normal layout (it does not require explicitly choosing a joystick mode). This means that on every press of **any arrow**, **CAPS SHIFT** is also silently "pressed" (bit `kb_data[0]|=1<<0`). If Lode Runner reads the Spectrum keyboard matrix directly, the hidden activation of CAPS SHIFT on every arrow may be the **true cause** of the observed "CPS" prefixes and symbol confusion in the game — rather than PS/2 signal integrity, which, as the oscillograms showed, is physically sound.

**How to disable/check:** the mode is switched via `cfg_def_kbd_mode` (settings menu, items "5"–"9" in `kbd_config`) or locked via `kbd_lock=true` (the HAT_A combination on a connected joystick). It is worth checking the current value of "Keyboard maps to..." in Tecnocat SETTINGS and trying to lock/change this mode to see whether the artifacts in Lode Runner disappear.

### Confirmation in Practice: Switching [KBD>Cursor] → [KBD>Kempst]

Verified by the user: in F12 → SETTINGS the "Keyboard maps to..." mode was `[KBD>Cursor]` (= `now_kbd_mode==0`, analysis above). After switching to `[KBD>Kempst]` (`now_kbd_mode==1`):

- **Left Shift, Right Shift, Left Alt, Right Alt began to behave the same** — the special OR block (`R_ALT || R_SHIFT || NUM_PERIOD || DELETE → CAPS+0`) exists **only** in the `now_kbd_mode==0` block (Cursor joystick) and writes to `kb_data`. In the `now_kbd_mode==1` block (Kempston) a similar OR exists, but it writes **only to the Kempston port** (`data_joy|=0b00010000`), without touching `kb_data` — that is why the "Backspace" side effect for the right-hand keys disappeared
- **Delete no longer deletes** — confirmed by the base translation table `kb_u_codes.c`:
  ```c
  {KB_U2_DELETE, 0x02, {0x00,0x00,0x00,0x00, 0x00,0x00,0x00,0x00}},  // DELETE → nothing
  ```
  The Delete key in the base (unconditional) mapping table is not bound to any Spectrum key — it "worked" only through the Cursor-mode-specific OR block. Outside this mode Delete has no effect
- **Regular Backspace continues to delete** — because it is an **unconditional** mapping in the base table (`kb_u_codes.c`), independent of `now_kbd_mode`:
  ```c
  {KB_U1_BACK_SPACE, 0x01, {1<<0,0x00,0x00,0x00, 1<<0,0x00,0x00,0x00}},  // BACKSPACE → CAPS+0 always
  ```
- **Shift+0 (left and right) works as Backspace** — as expected, since in the base table both Shifts are mapped identically to CAPS SHIFT:
  ```c
  {KB_U1_L_SHIFT, 0x01, {1<<0,0x00,0x00,0x00, 0x00,0x00,0x00,0x00}},
  {KB_U1_R_SHIFT, 0x01, {1<<0,0x00,0x00,0x00, 0x00,0x00,0x00,0x00}},
  ```
  and physically pressing the `0` key together with CAPS SHIFT is exactly the Delete code on a real Spectrum (CAPS+0), regardless of the joystick mode
- **Left Alt and Right Alt are not mapped to anything at all in the base table** (`{0x00,0x00,0x00,0x00, 0x00,0x00,0x00,0x00}` for both) — their only function in Tecnocat comes from the mode-specific OR blocks (Cursor/Kempston/Sinclair/QAOPM), or from the firmware's system hotkeys (Alt+F11, Alt+F12, etc.)

**Final conclusion:** the behavior is fully determined by the firmware code and fully explained. The recommended configuration for clean typing (without side effects from the Cursor Joystick) is a mode other than `[KBD>Cursor]` (for example, `[KBD>Kempst]`, as you chose), or `kbd_lock=true` with a physical joystick connected.

## Open Questions

- ✅ **Resolved:** the cause of Backspace/Delete on Right Alt/Right Shift is the "Cursor Joystick" mode (CAPS+0 = Fire), see the "Finding from the Firmware Sources" section above.
- ✅ **Confirmed in practice:** switching to `[KBD>Kempst]` eliminates the side effect for Right Alt/Right Shift; Delete stops working (as expected — no direct mapping in the base table); regular Backspace and Shift+0 continue to work (unconditional mapping).
- ✅ **Verified in practice (Lode Runner, Redefine Keys menu, reproduced twice):** in `[KBD>Cursor]`, pressing Left and Right produces the "garbage" CPS symbol — the hidden CAPS SHIFT press that Cursor mode adds to the arrow (CAPS+5 / CAPS+8). In `[KBD>Kempst]` the Left/Right/Up/Down arrows do not respond at all in Redefine Keys — as expected: in this mode the arrows go to the Kempston joystick port, not to the keyboard matrix that Redefine Keys polls. The hidden CAPS SHIFT hypothesis is confirmed. To control the game with the arrows in `[KBD>Kempst]` mode, select Kempston control in the game (if available); otherwise, assign letter keys in Redefine Keys.
- Clarify in the documentation/sources the specifics of the 128 BASIC editor and how tokens are entered with the blue cursor.
- Test SYM SHIFT + SPACE (BREAK).

---

## Oscilloscope References (Tektronix, Ch1/Ch2 = 2.00V/div)

### Left Ctrl — make code (press), reference clean signal

- **Capture date:** 22 May 2025, 09:04:59
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** a single block of **11 clock pulses** on CLK — corresponds to one PS/2 frame (start bit + 8 data bits + parity + stop bit), make code `0x14` (Left Ctrl pressed)
- **Signal quality:** both channels clean, sharp edges, no ringing or dips
- **Status:** reference make frame for comparison with captures of E0-prefixed keys (arrows, Right Shift, Right Alt) during further diagnostics

### Left Ctrl — break code (release), reference clean signal

- **Capture date:** 22 May 2025, 09:05:55
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** **two blocks** of clock pulses on CLK — corresponds to the two bytes of the break code in PS/2 Scan Set 2 (`F0` + `14`), Left Ctrl released
- **Signal quality:** both channels clean, sharp edges, no distortion
- **Status:** reference break frame (2 bytes) for comparison with E0-prefixed keys, where the break code consists of 3 bytes (`E0` + `F0` + code)

### Right Ctrl — make code (press), E0-prefixed signal

- **Capture date:** 22 May 2025, 09:07:32
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** **two blocks** of clock pulses on CLK — corresponds to the two bytes of the E0-prefixed make code (`E0` + `14`), Right Ctrl pressed
- **Signal quality:** both channels clean, sharp edges, no distortion — the E0 prefix is transmitted as reliably as single-byte codes
- **Consistent with:** the previously confirmed behavior — Right Ctrl works normally as SYMBOL SHIFT, duplicating Left Ctrl

### Right Ctrl — break code (release), E0-prefixed signal

- **Capture date:** 22 May 2025, 09:08:54
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** **three blocks** of clock pulses on CLK — corresponds to the three bytes of the E0-prefixed break code (`E0` + `F0` + `14`), Right Ctrl released
- **Signal quality:** both channels clean, sharp edges, no distortion
- **Conclusion:** the full 3-byte E0 cycle (make E0+code, break E0+F0+code) is transmitted cleanly for Right Ctrl — confirms that the transmission of multi-byte E0 sequences itself is physically sound for this key; the "garbage" symbols problem in Lode Runner is probably specific to particular keys/codes (arrows, etc.) or to the game context, rather than a general degradation of E0 as a class

### Left Alt — make code (press), clean signal

- **Capture date:** 22 May 2025, 09:10:40
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** a single block of clock pulses on CLK — corresponds to one PS/2 frame, make code `0x11` (Left Alt), without the E0 prefix
- **Signal quality:** clean, no distortion
- **Function on the Spectrum:** not yet confirmed in this session (see the modifiers table)

### Left Alt — break code (release), clean signal

- **Capture date:** 22 May 2025, 09:11:24
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** a block of clock pulses on CLK, corresponds to the break code `F0`+`11` (Left Alt), without the E0 prefix
- **Signal quality:** clean, no distortion

### Right Alt — make code (press), E0-prefixed signal

- **Capture date:** 22 May 2025, 09:12:18
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** two blocks of clock pulses on CLK, corresponds to the two bytes of the E0-prefixed make code (`E0` + `11`), Right Alt pressed
- **Signal quality:** clean, sharp edges, no distortion — the same result as for Right Ctrl
- **Important:** the physical transmission of the E0 prefix for Right Alt is also sound; the observed Backspace/Delete behavior on the Spectrum is therefore the result of the firmware processing a correctly received code, not a consequence of a corrupted signal

### Right Alt — break code (release), E0-prefixed signal

- **Capture date:** 22 May 2025, 09:13:43
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** three blocks of clock pulses on CLK, corresponds to the three bytes of the E0-prefixed break code (`E0` + `F0` + `11`), Right Alt released
- **Signal quality:** clean, sharp edges, no distortion
- **Summary for Right Alt:** the full 3-byte E0 cycle (make + break) is transmitted without errors — finally confirms that the Backspace/Delete behavior on Right Alt/Right Shift is caused by software mapping in the Tecnocat firmware, not by a hardware problem with the PS/2 signal

### Left Shift — make code (press), clean signal

- **Capture date:** 22 May 2025, 09:18:22
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** a single block of clock pulses on CLK, corresponds to the make code `0x12` (Left Shift), without the E0 prefix
- **Signal quality:** clean, no distortion

### Left Shift — break code (release), clean signal

- **Capture date:** 22 May 2025, 09:19:12
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** two blocks of clock pulses on CLK, corresponds to the break code `F0`+`12` (Left Shift), without the E0 prefix
- **Signal quality:** clean, no distortion

### Right Shift — make code (press), clarification on the E0 prefix

- **Capture date:** 22 May 2025, 09:20:50
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** **a single block** of clock pulses on CLK — a single frame, WITHOUT the E0 prefix
- **Important clarification:** this corrects an earlier assumption in the table — Right Shift, unlike Right Ctrl and Right Alt, **has no E0 prefix** per the PS/2 Scan Set 2 specification (make code `0x59`, single-byte). The modifiers table above has been updated.
- **Significance for diagnostics:** since Right Shift is not an E0 code, the observed Backspace/Delete behavior when it is pressed is already the second example of a non-E0-prefixed key producing a non-standard function (alongside, possibly, a different explanation for Right Alt). This weakens the "E0-specific" software mapping version and points rather to mapping by specific scan codes (`0x59` and `E0 11`), not by the E0-prefix attribute as such

### Right Shift — break code (release), confirmation of no E0

- **Capture date:** 22 May 2025, 09:21:39
- **Timebase settings:** M = 400µs/div, trigger A Ch2, level 1.64V
- **CH1 (yellow) = DATA**, **CH2 (green) = CLK**
- **Observation:** two blocks of clock pulses on CLK, corresponds to the break code `F0`+`59` (Right Shift), without the E0 prefix — consistent with the make code signature
- **Signal quality:** clean, no distortion
- **Summary for Right Shift:** the full 2-byte cycle (make + break, both without E0) is transmitted cleanly; the status of Right Shift as a non-E0 key in PS/2 Scan Set 2 is finally confirmed
