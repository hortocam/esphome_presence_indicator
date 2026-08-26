# Home Assistant integration

Home Assistant is the **source of truth** for the meeting state. The ESPHome
device is a stateless renderer that just turns the right colour wire on.

MuteDeck's webhook (and its local API) sends a **flat JSON status**, not a single
state string:

```json
{ "control":"zoom", "call":"active", "mute":"active",
  "video":"inactive", "share":"inactive", "record":"disabled",
  "timestamp":"2025-01-26T10:30:00Z" }
```

Field values are `active` / `inactive` / `disabled`. So we keep the raw status in
HA, then **derive** a clean 4-state entity from it. Everything else keys off that
derived entity.

## Flow

```
MuteDeck (WorkLaptop)  --webhook-->  HA webhook endpoint
                                        |
                                        v
                          sensor.mutedeck_status   (raw JSON in attributes)
                                        |
                                        v
                          select.meeting_state  (derived 4-state)
                                        |
                                        v
                                 automation: state -> colour
                                        |
                                        v
                          switch.tally_red / _yellow / _green
                          (ESPHome device, native API)
```

## 1. Ingest the webhook into a raw status sensor

```yaml
# configuration.yaml
webhook:
  - meeting_status:
      # endpoint: POST http://<HA>/api/webhook/mutedeck_status
      webhook_id: mutedeck_status

sensor:
  - platform: template
    name: MuteDeck Status
    id: mutedeck_status
    # The state is derived by the automation below; attributes hold the raw fields.
```

```yaml
# automation to land the webhook JSON into the sensor
automation:
  - id: mutedeck_ingest
    alias: "MuteDeck webhook -> status sensor"
    trigger:
      - platform: webhook
        webhook_id: mutedeck_status
    action:
      - variables:
          md: "{{ trigger.json }}"
      - service: template.publish
        target: { entity_id: sensor.mutedeck_status }
        data:
          state: "{{ md.control }}"
          attributes:
            call: "{{ md.call }}"
            mute: "{{ md.mute }}"
            video: "{{ md.video }}"
            share: "{{ md.share }}"
            record: "{{ md.record }}"
            timestamp: "{{ md.timestamp }}"
```

> **Prefer polling?** The API also exposes `GET /v1/status` on the WorkLaptop
> (`http://172.16.20.12:3491/v1/status`). A regular HA automation can poll it on
> an interval and `template.publish` the same sensor. Webhook is lower-latency;
> polling is simpler if the WorkLaptop can't reach HA or you want to avoid
> opening a webhook port. MuteDeck listens on all IPs by default; you can
> restrict it to localhost if you don't need LAN access.

## 2. The derived 4-state entity

```yaml
select:
  - platform: template
    name: Meeting State
    options:
      - free
      - in_meeting
      - muted
      - dnd
    select_option:
      - service: template.publish
        target: { entity_id: select.meeting_state }
        data:
          state: "{{ option }}"
    # Derive from the raw status sensor attributes
    value_template: >
      {% set c = state_attr('sensor.mutedeck_status', 'call') %}
      {% set m = state_attr('sensor.mutedeck_status', 'mute') %}
      {% set r = state_attr('sensor.mutedeck_status', 'record') %}
      {% set s = state_attr('sensor.mutedeck_status', 'share') %}
      {% if c != 'active' %} free
      {% elif r == 'active' or s == 'active' %} dnd
      {% elif m == 'active' %} muted
      {% else %} in_meeting
      {% endif %}
```

Derivation table:

| State | When |
|---|---|
| `free` | `call != active` (not in a meeting) |
| `in_meeting` | `call == active` and not muted |
| `muted` | `call == active` and `mute == active` |
| `dnd` | `call == active` and (recording or screen-sharing) |

`video` (camera) is deliberately **not** treated as its own state — a camera-on
during a call just reads as `in_meeting`. Tweak the `value_template` to taste,
e.g. drop `share` too if you don't want screen-sharing to flag `dnd`.

## 3. State -> colour automation

This is the only place that decides what a state looks like physically. It drives
the ESPHome switches; because it's HA, you can drive **any** light instead (a HUE
bulb, a notification) by pointing the action elsewhere.

```yaml
automation:
  - id: meeting_state_to_tally
    alias: "Meeting state -> tally light"
    trigger:
      - platform: state
        entity_id: select.meeting_state
    action:
      - choose:
          - conditions: { condition: state, entity_id: select.meeting_state, state: in_meeting }
            sequence:
              - service: switch.turn_on
                target: { entity_id: switch.tally_red }
              - service: switch.turn_off
                target: { entity_id: [switch.tally_yellow, switch.tally_green] }
          - conditions: { condition: state, entity_id: select.meeting_state, state: muted }
            sequence:
              - service: switch.turn_on
                target: { entity_id: switch.tally_yellow }
              - service: switch.turn_off
                target: { entity_id: [switch.tally_red, switch.tally_green] }
          - default:   # free / dnd
            sequence:
              - service: switch.turn_on
                target: { entity_id: switch.tally_green }
              - service: switch.turn_off
                target: { entity_id: [switch.tally_red, switch.tally_yellow] }
```

Suggested colour mapping (adjust to taste):

| State | Colour |
|---|---|
| `free` | green |
| `in_meeting` | red |
| `muted` | yellow |
| `dnd` | green (blink or reuse) — or map `dnd` to the same as `muted` |

Set `restore_mode` on the ESPHome switches so the light returns to its last state
after a reboot.

## 4. Everything else

Once `select.meeting_state` exists, all of this works without touching the device:

- Dashboard card showing the meeting state as a widget.
- A `state` badge on your Lovelace home tab.
- Automations ("pause music when in a call", "don't fire notifications during
  dnd") driven purely by the select entity.
- Point the colour mapping at a different HA light later — the device never changes.
