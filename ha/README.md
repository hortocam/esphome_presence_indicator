# Home Assistant setup (tally light)

There are **three pieces** that must all be in place. They live in separate
parts of HA (a package, a webhook trigger, and the MuteDeck app itself), which
is why this is more than one step. Each section has a **Verify** step so you
know when it's actually done.

## Prerequisites

- HAOS or Home Assistant Supervised at `http://<ha-ip>:8123` (this homelab: `172.16.10.16`).
- The tally ESP32 already **adopted** in HA (Settings → Devices & Services →
  the ESPHome device `Meeting Presence Tally`). It should show three switches:
  `switch.meeting_presence_tally_tally_red / _yellow / _green`.
- A file-editor add-on in HA (**Studio Code Server** or **File editor**) so you
  can edit YAML from the browser.

## Step 1 — Enable packages (one time, `configuration.yaml`)

Open `/config/configuration.yaml` and make sure the top has:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Keep the rest of `configuration.yaml` unchanged (all other keys at column 0,
NOT nested under `homeassistant:`).

> **Why `include_dir_named`:** each file under `packages/` becomes a package
> named after its filename. Content sits at column 0 (no extra indentation).
> `include_dir_merge_named` needs an extra indentation level and would NOT match
> this package — do not switch.

## Step 2 — Add the package

1. Create `/config/packages/` if it doesn't exist.
2. Copy `ha/packages/tally_light.yaml` from this repo into it.
3. **The filename MUST be exactly `tally_light.yaml` (underscores, no hyphens).**
   HA uses the filename as the package slug, and slugs reject hyphens — a
   hyphenated name makes HA silently drop the whole package with no entities.

What it provides:
- `input_text.mutedeck_webhook` — stores the raw MuteDeck payload.
- `sensor.mutedeck_status` — derived status sensor.
- `select.meeting_state` — the 4-state entity (free / in_meeting / muted / dnd)
  that drives everything.
- Two automations: `mutedeck_ingest` (webhook → input_text) and
  `meeting_state_to_tally` (select → colours).

**Verify:** Developer Tools → YAML → Check Configuration shows no errors, and
after restart the entities above exist in Developer Tools → States.

## Step 3 — Point MuteDeck at the webhook

In MuteDeck (the WorkLaptop): **Settings → Notifications → Enable Webhook**, and
paste:

```
http://172.16.10.16:8123/api/webhook/mutedeck_status
```

The `mutedeck_status` in the URL must match the `webhook_id` on the ingest
automation in the package.

**Verify:** join a meeting → the tally goes red. Mute → yellow. Hang up → green.

## If it doesn't load

- **No entities at all** → the package slug or path is wrong. Confirm the file is
  at `/config/packages/tally_light.yaml` (underscores) and Check Configuration.
- **Check Configuration error** → it names the exact line; fix and reload.
- **Webhook never fires** → MuteDeck on the WorkLaptop may not reach HA on
  `172.16.10.16:8123` (cross-VLAN firewall). Test from the WorkLaptop:
  `curl http://172.16.10.16:8123/api/webhook/mutedeck_status -X POST`.

## Colour mapping (edit the automation)

| State | Colour(s) |
|-------|-----------|
| free | green |
| in_meeting | red |
| muted | yellow |
| dnd | green + yellow (idea, parked) |

To change a mapping, edit the `meeting_state_to_tally` automation in the package
(Developer Tools → YAML → reload, or Settings → Automations → the automation).
Only one switch is on at a time by default; the DND green+yellow would turn two
on together (fine — the MOSFETs handle it).
