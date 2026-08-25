# Home Assistant integration

Home Assistant is the **source of truth** for the meeting state. The ESPHome
device is a stateless renderer that just turns the right colour wire on.

## Flow

```
MuteDeck (WorkLaptop)  --webhook-->  HA webhook endpoint
                                        |
                                        v
                              select.meeting_state  (4-state)
                                        |
                                        v
                                 automation: state -> colour
                                        |
                                        v
                          switch.tally_red / _yellow / _green
                          (ESPHome device, native API)
```

## 1. The 4-state entity

Create a `select` entity that models the meeting state. Example:

```yaml
# configuration.yaml (or a package / blueprint)
select:
  - platform: template
    name: Meeting State
    options:
      - free
      - in_meeting
      - muted
      - do_not_disturb
```

This is the single entity every automation and dashboard widget keys off.

## 2. Ingest MuteDeck status into HA

Expose an HA webhook that MuteDeck POSTs to when its status changes.

```yaml
# configuration.yaml
webhook:
  - meeting_status:
      # endpoint: POST http://<HA>/api/webhook/mutedeck_status
      webhook_id: mutedeck_status
```

```yaml
automation:
  - id: mutedeck_ingest
    alias: "MuteDeck -> meeting state"
    trigger:
      - platform: webhook
        webhook_id: mutedeck_status
    action:
      - service: select.select_option
        target:
          entity_id: select.meeting_state
        data:
          option: "{{ trigger.json.state }}"
```

Adjust `trigger.json.state` to the exact field name MuteDeck sends. If you
prefer polling, a regular HA automation can poll the MuteDeck API on an
interval and update `select.meeting_state` the same way.

## 3. State -> colour automation

This is the only place that decides what "in a meeting" looks like physically.
It drives the ESPHome switches; because it's HA, you can drive **any** light
instead (a different HA light, a HUE bulb, a notification) by just pointing the
action elsewhere.

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
          - default:   # free / do_not_disturb
            sequence:
              - service: switch.turn_on
                target: { entity_id: switch.tally_green }
              - service: switch.turn_off
                target: { entity_id: [switch.tally_red, switch.tally_yellow] }
```

Set `restore_mode` on the ESPHome switches so the light returns to its last
state after a reboot.

## 4. Everything else

Once `select.meeting_state` exists, all of this works without touching the
device:

- Dashboard card showing the meeting state as a widget.
- A `state` badge on your Lovelace home tab.
- Automations ("pause music when in a call", "don't fire notifications during
  do_not_disturb") driven purely by the select entity.
- Point the mapping at a different HA light later — the device never changes.
