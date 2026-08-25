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
  presence_indicator.yaml   # ESPHome device config (switches, pins, WiFi)
  secrets.yaml              # LOCAL-ONLY, gitignored - fill in, never commit
docs/
  wiring.md                 # NPN low-side switching + polarity notes
  home-assistant.md         # MuteDeck webhook -> 4-state entity -> automation
README.md
LICENSE                     # MIT
```

## Status

- [x] Repo scaffold, ESPHome layout
- [x] Wiring spec (common-anode, NPN low-side) — see `docs/wiring.md`
- [x] GPIO pins set (R=GPIO25, Y=GPIO26, G=GPIO27)
- [ ] Full build + first flash + serial boot log
- [ ] MuteDeck -> HA integration wired
- [ ] Enclosure + install

MIT Licensed — see `LICENSE`.
