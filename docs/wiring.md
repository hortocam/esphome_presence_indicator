# Wiring — Tally Light to ESP32

The tally light is **common-anode**: **black = common +12 V**, and each colour
wire (red / yellow / green) is switched **to ground** to turn that colour on.

## Switch: N-channel logic-level MOSFET (STP16NF06L)

Because the colours sink to GND, we use a **low-side N-channel MOSFET** per
colour. The STP16NF06L is a logic-level MOSFET, so the ESP32's 3.3 V GPIO drives
it fully on — no base-current math, no saturation concerns. It's rated far above
the LED current, so with only one colour active at a time it runs cold.

> **Do NOT use an S8550 here** — that part is a **PNP**, which cannot sink in a
> low-side switch. The STP16NF06L (or any logic-level N-MOSFET, or an NPN like
> the PN2222A with a base resistor) are the right parts.

## Circuit (one MOSFET per colour)

```
       12V +12V (black)
         │
  ┌──────┴──────┐        (each colour wire = drain)
  │  Red wire   │        Red wire   ─► drain(Q1)
  │  (anode)    │        Yellow wire ─► drain(Q2)
  └──────┬──────┘        Green wire  ─► drain(Q3)
         │
       drain(Q)  ←── colour wire
        ┌┴┐
  Q1   ( FET )    STP16NF06L (N-channel, logic-level)
        └┬┘
        │
       source ─── GND  (all sources to the same GND)

   GPIO ──1k── gate ──10k── GND
   (gate pulldown keeps the light OFF at boot
    before the GPIO is initialized)
```

Per colour:
- **N-channel logic-level MOSFET** (STP16NF06L)
- **1 kΩ gate resistor** (GPIO → gate)
- **10 kΩ gate-to-GND pulldown** (keeps it OFF during boot)
- **Drain** → colour wire
- **Source** → GND

**GPIO HIGH turns the colour on.** The gate draws almost no current, so the GPIO
swings it directly. No inversion needed (`inverted: false`, the ESPHome default).

## Required wiring

| From | To | Notes |
|------|----|----|
| ESP32 GND | 12V supply GND | **must** share ground |
| 12V + | tally black (COM) | common +12V |
| GPIO25 | 1k → gate Q1 | drain → red wire |
| GPIO26 | 1k → gate Q2 | drain → yellow wire |
| GPIO27 | 1k → gate Q3 | drain → green wire |
| Q1/Q2/Q3 gates | 10k → GND | boot-safety pulldown |
| all sources | GND | same ground node |

## Check before powering

1. Confirm the light is common-anode (black = +). If it turns out common-cathode,
   swap to high-side switching (PNP/MOSFET on the +12 V side) — the ESP32 pins
   stay the same.
2. Tie the 12V supply ground to the ESP32 ground (floating grounds cause the
   classic "light flickers / GPIO acts weird" failure).
3. Pick 3 GPIOs that are not strapping pins and not in use by WiFi flash.
   GPIO25/26/27 are clean (not strapping; not the flash/WiFi pins).

## Pin assignments

`GPIO25` = red, `GPIO26` = yellow, `GPIO27` = green — set in
`esphome/presence_indicator.yaml`. These are the board's GPIO numbers; map each
to the physical header pin from the silkscreen labels (see the `esp32-firmware`
skill).
