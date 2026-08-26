# ESPHome Presence Indicator

ESPHome-based **meeting presence tally light**. A NodeMCU-32S (ESP32) switches the
three colour channels of a 12 V LED tally light to reflect Cameron's live meeting
status, as reported by **MuteDeck** running on the WorkLaptop.

![Tally light](docs/tally-light.jpg)

## Why this exists

Cameron's meeting status lives in MuteDeck (runs on the WorkLaptop, knows status
across multiple meeting platforms: Teams, Zoom, Meet, Slack). MuteDeck exposes a
pollable API **and** a webhook. This project turns that status into a physical
"don't interrupt me" light on the desk — red when on a call, green when free,
yellow when muted or away.

## Architecture

```
+----------------+   status           +----------------------+
|  MuteDeck       | -----------------> | Home Assistant       |
|  (WorkLaptop)   |   webhook/API      |  (source of truth)  |
+----------------+                    +----------+-----------+
                                                | 4-state select entity
                                                v
                                     +----------------------+
                                     |  ESP32 (ESPHome)     |
                                     |  - 3 switch outputs  |
                                     |  - NPN low-side driv-|
                                     |    ers -> 12V tally  |
                                     |  - WiFi / native API |
                                     +----------------------+
```

**Home Assistant is the single source of truth.** MuteDeck lands its status into
a 4-state entity in HA; HA maps that state to a colour; the ESPHome device is a
*stateless renderer* that just turns the right colour wire on. This decouples the
light from MuteDeck entirely — you can map the state to **any** HA light, drive
automations off state changes, and add dashboard widgets without touching
firmware.

## Repository layout

```
esphome/
  presence_indicator.yaml   # ESPHome device config (switches, pins, WiFi, API)
  secrets.yaml              # LOCAL-ONLY, gitignored - fill in, never commit
ha/
  packages/tally_light.yaml # HA package: webhook, sensor, derived select, automations
  README.md                 # Step-by-step HA install (3 steps + verify)
  architecture.md           # How the MuteDeck -> HA -> ESP32 chain fits together
docs/
  wiring.md                 # Low-side STP16NF06L MOSFET circuit + polarity notes
  protoboard-layout.md      # Perfboard (PY-5CM×7CM) pad map + build order
README.md
LICENSE                     # MIT
```

## Install

Two halves, two hosts:

1. **ESP32:** see `esphome/presence_indicator.yaml` — flash from the Mac
   (`esphome run ... --device /dev/cu.usbserial-0001`), adopt in HA, paste the
   native-API encryption key.
2. **Home Assistant:** follow `ha/README.md` — enable packages, drop in
   `ha/packages/tally_light.yaml` (as `tally_light.yaml`), point MuteDeck's
   webhook at `http://172.16.10.16:8123/api/webhook/mutedeck_status`.

## Status

- [x] Repo scaffold, ESPHome layout
- [x] Wiring spec (common-anode, low-side STP16NF06L MOSFETs) — see `docs/wiring.md`
- [x] GPIO pins set (R=GPIO25, Y=GPIO26, G=GPIO27)
- [x] Full build + first flash + adopted in HA
- [x] All 3 switches verified in HA; nothing lit at power-on/boot (gate pulldowns)
- [x] MuteDeck -> HA integration wired (webhook -> input_text -> sensor + derived select)
- [x] E2E verified live: Google Meet join -> red, mute -> yellow, hang up -> green
- [ ] DND state mapping (idea: green+yellow) + how MuteDeck reports DND — parked
- [ ] Enclosure + install

MIT Licensed — see `LICENSE`.
