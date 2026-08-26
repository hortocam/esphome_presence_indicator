# Protoboard layout — Meeting Presence Tally

Target: a single **PY-5CM×7CM** perfboard (18 columns `1–18`, 24 rows `A–X`,
0.1"/2.54mm pitch, copper pads on one side). This circuit is small (3 MOSFETs +
6 resistors), so it fits comfortably on **one board, one side** — no stacking.

> **Confirm before soldering** (both are unknowns I can't see):
> 1. The physical NodeMCU-32S header positions of GPIO25/26/27 and GND on your
>    board — read the silkscreen (see the `esp32-firmware` skill).
> 2. Which of red/yellow/green maps to which MOSFET if you care about colour
>    order on the board (matches the HA config: R→GPIO25, Y→GPIO26, G→GPIO27).

---

## The big idea: three copper-strip buses, everything else short jumps

The copper is on one side, so use it. Run **three long solder-bridged strips of
copper pads** as your power/ground rails, and every other connection is a short
point-to-point jumper. This is the classic "perfboard-as-minimal-PCB" pattern.

Pick three **vertical columns** (rows A–X) to be your buses — they stay clean and
don't fight the horizontal component rows:

| Bus | Column | What it carries |
|-----|--------|-----------------|
| **GND** | `1` | ESP32 GND, all 3 MOSFET sources, 5V-buck GND, 12V return |
| **12V+** | `18` | tally black (COM, +12V) feed |
| **5V+** | `14` | ESP32 VIN from the 5V buck (short column) |

> Solder the bus columns by bridging every copper pad down the column. Solder the
> jumper that feeds each bus in from the power source at the top, then every
> component lead that lands on a bus column is auto-connected. One ground node =
> no floating-ground gremlins.

---

## Component placement (south-to-north)

### Row W — GND bus header (column 1)
- NodeMCU **GND** header pin → column `1` (GND bus).
- 5V buck **GND** → column `1`.
- (If the tally's 12V return must share ground — it does, tie the 12V supply
  GND to column `1` too.)

### Rows U–S — the three MOSFETs (STP16NF06L, through-hole)

Mount all three side by side with their **source** legs on the GND bus.

| Row | MOSFET | source → GND | drain → tally colour | gate |
|-----|--------|--------------|----------------------|------|
| U | **Q1 (red)** | col 1 | col 2 (red wire) | via 1k to col 3 |
| T | **Q2 (yellow)** | col 1 | col 2 (yellow) | via 1k to col 4 |
| S | **Q3 (green)** | col 1 | col 2 (green) | via 1k to col 5 |

- Each MOSFET's **source** pin soldered to column `1` (GND bus).
- Each **drain** → its tally colour wire (route the wire to a solder pad, e.g.
  col `2`, one per colour — tag `RED`/`YEL`/`GRN`).
- Each **gate** → a **1kΩ** resistor → the GPIO column.

### Rows R–Q — the 6 resistors

- **3× 1kΩ gate resistors**, one per MOSFET: gate → 1k → GPIO column.
- **3× 10kΩ gate pulldowns**, one per MOSFET: gate → 10k → GND column `1`.

This is the wiring from `docs/wiring.md`: GPIO→1k→gate, gate→10k→GND,
drain→colour, source→GND.

### Column 3/4/5 — GPIO header (from NodeMCU-32S)

Route the three GPIO signals **up** to the top, off the ESP32:

| GPIO | Column | → 1k → gate of |
|------|--------|----------------|
| GPIO25 (red) | col 3 | Q1 |
| GPIO26 (yellow) | col 4 | Q2 |
| GPIO27 (green) | col 5 | Q3 |

These are short vertical jumpers from the ESP32 header down to the 1k resistors.

### Power input

- 5V buck **out** → column `14` (5V bus) → ESP32 VIN.
- 12V feed (from the buck/PSU) → column `18` (12V bus) → tally black wire.

---

## DIYLC pad map (18 cols × 24 rows)

A compact grid to lay out in DIYLC (columns `1–18`, rows `A–X`). Bus columns are
solder-bridged strips; everything else is a component or a short jumper.

```
     1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18
  A  . . . . . . . . . .  .  .  .  .  .  .  .  .    <- power-in end
  .  G N G G G . . . . .  .  .  .  5V .  .  . 12V   (buses run the full col)
  .  N R R R . . . . . .  .  .  .  .  .  .  .  .
  .  D D D . . . . . . .  .  .  .  .  .  .  .  .
  W  [GND][G] [G] [G] . .  .  .  .  .  .  .  .  .   <- NodeMCU GND -> GND
  V  .     .  .  .  . . .  .  .  .  .  .  .  .  .
  U  .  Q1 .  .  .  . . .  .  .  .  .  .  .  .  .   Q1 red  (source->GND)
  T  .  Q2 .  .  .  . . .  .  .  .  .  .  .  .  .   Q2 yellow
  S  .  Q3 .  .  .  . . .  .  .  .  .  .  .  .  .   Q3 green
  R  .  [1k][1k][1k] . . .  .  .  .  .  .  .  .  .   gate resistors
  Q  .  [10k][10k][10k] . . . .  .  .  .  .  .  .  .  pulldowns -> GND
  ...
  D  .  .  .  .  .  . . .  .  .  .  .  .  .  .  .  .
  C  .  .  .  .  .  . . .  .  .  .  .  .  .  .  .  .
  B  .  .  .  .  .  . . .  .  .  .  .  .  .  .  .  .
      <- GPIO25/26/27 jump up to ESP32 header near here
```

Legend:
- `GND`/`5V`/`12V` = the three solder-bridged copper bus columns.
- `Q1/Q2/Q3` = the three MOSFETs (drain legs to `RED`/`YEL`/`GRN` pads).
- `1k` = gate resistor columns, `10k` = pulldown columns.
- Exact rows are flexible — keep the three buses straight and everything else
  short. Adjust to your actual NodeMCU header layout.

## Build order (solder bottom-up, verify at each step)

1. **Solder the three copper bus strips** (cols 1, 14, 18) — bridge the pads.
2. **Mount the 3 MOSFETs** with sources on the GND bus.
3. **Add the 3 × 1k and 3 × 10k resistors.**
4. **Solder the 3 drain wires** (RED/YEL/GRN) to pads.
5. **Solder power-in** (5V→ESP32, 12V→black).
6. **Set the ESP32**, wire GPIO25/26/27 + GND to the board.
7. **Power up on the bench, tie grounds, test each colour** (mirror the
   breadboard test you already passed).
8. **Tack down the tally wires and the common ground.**

---

## Verification gate

- Continuity on the three buses before powering.
- No pad bridges between the gate columns and adjacent ones.
- On first power: **nothing lit** (gate pulldowns hold it dark) — then toggle
  each GPIO in HA and confirm the right colour lights.

---

## Future: the D1-Mini / PCB version (this board is the prototype)

This perfboard is the **prototype**. When it's proven, design the real board in
KiCad:

- **D1 Mini** (ESP8266 or ESP32-S variant) — needs only 5 pins (3× GPIO + VCC
  + GND), much smaller than the NodeMCU-32S.
- **SMD MOSFETs**: SOT-23 logic-level N-channel (e.g. IRLML6344 / DMN3150 —
  the SMD cousins of the STP16NF06L). 3× SOT-23 + 6× 0402/0603 resistors.
- Board footprint can shrink to ~2cm square.

KiCad is the right tool for that step; DIYLC is the right tool for this
perfboard plan.
