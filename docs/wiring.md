# Wiring — Tally Light to ESP32

The tally light is **common-anode**: **black = common +12 V**, and each colour
wire (red / yellow / green) is switched **to ground** to turn that colour on.

## Circuit (NPN low-side switching)

Because the colours sink to GND, we use a simple NPN low-side switch per colour.
Only one colour is ever active, so current never stacks — no heat concern.

```
       12V +12V (black)
         │
  ┌──────┴──────┐        (each colour wire = collector)
  │  Red wire   │        Red wire   ─► collector(Q1)
  │  (anode)    │        Yellow wire ─► collector(Q2)
  └──────┬──────┘        Green wire  ─► collector(Q3)
         │
        ┌┴┐            Collector
  Q1   ( NPN )  ←─────   colour wire
        └┬┘            (2N3904 / 2N2222)
         │
        ┌┴┐
       [1k]  base resistor, GPIO -> 1k -> base
        │
      GPIO  (HIGH = colour ON)

   Emitter ─── GND  (all emitters to the same GND)
```

Per colour:
- **NPN** (2N3904 / 2N2222 — whatever is in the parts box)
- **1 kΩ** base resistor (GPIO → resistor → base)
- **Collector** → colour wire
- **Emitter** → GND
- **Base** → GPIO via the 1k

**GPIO HIGH turns the colour on.** No inversion needed (`inverted: false`,
the ESPHome default). The LED strings draw only a few tens of mA, so a small
signal NPN is plenty.

## Required wiring

| From | To | Notes |
|------|----|----|
| ESP32 GND | 12V supply GND | **must** share ground |
| 12V + | tally black (COM) | common +12V |
| GPIO (red) | 1k → base of Q1 | collector → red wire |
| GPIO (yellow) | 1k → base of Q2 | collector → yellow wire |
| GPIO (green) | 1k → base of Q3 | collector → green wire |
| all emitters | GND | same ground node |

## Check before powering

1. Confirm the light is common-anode (black = +): if it turns out common-cathode,
   the same NPNs still work but you switch to **high-side PNP** — see README.
2. Tie the 12V supply ground to the ESP32 ground (floating grounds cause the
   classic "light flickers / GPIO acts weird" failure).
3. Pick 3 GPIOs that are not strapping pins and not in use by WiFi flash.

## Pin placeholders

`GPIO_NC` in `presence_indicator.yaml` are intentional — set the real GPIO
numbers from the board silkscreen before flashing (wrong pins are the #1 cause
of a dead board).
