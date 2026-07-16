# LocknAlert Home Assistant Blueprints

This document covers three automation blueprints: low battery scanning, MQTT command-status notifications, and alarm partition state-change notifications. Each section explains what the blueprint does, its inputs, how to install it, and how to instantiate an automation from it.

---

## 1. RF Low Battery Trouble Notifier

### What it does
Periodically scans every `binary_sensor` entity belonging to a chosen device (e.g. a Paradox panel or expander) for any entity whose entity_id contains a given keyword (default `rf_low_battery_trouble`) that is currently `on`. If any match, it sends one notification listing the affected zones.

Runs on a timer (`time_pattern`) rather than reacting to a single sensor turning on, because the underlying template needs to enumerate *all* entities on the device dynamically — a static `state:` trigger can't bind to a device-scoped, keyword-filtered entity list.

### Inputs

| Input | Description | Example |
|---|---|---|
| `target_device` | Device whose binary_sensors will be scanned (device picker) | Your Paradox MG5050 panel device |
| `trouble_keyword` | Substring to match in entity_id | `rf_low_battery_trouble` |
| `scan_interval` | How often to run the check | Every 30 minutes |
| `notify_service` | Notify service to call | `notify.mobile_app_raine_s_a55` |
| `notify_title` | Notification title | `Low Battery Alert` |

### Install
1. Save the blueprint YAML to `/config/blueprints/automation/locknalert/rf_low_battery_notifier.yaml`.
2. Settings → Automations & Scenes → Blueprints tab. It will appear as **RF Low Battery Trouble Notifier**.
3. Click **Create Automation**, fill in the inputs above, save.

### Known trade-off
This is polling, not event-driven. A trouble condition that appears between scans waits up to `scan_interval` before you're notified. Acceptable for battery drain (slow-moving), not for anything needing sub-minute latency.

---

## 2. Alarm Command Status Notifier

### What it does
Listens for a Paradox MQTT message on a `command_status` topic. When the payload contains a `reason:` field (e.g. a blocked arming attempt), it:
1. Extracts the reason text.
2. Writes it to an `input_text` helper (for dashboard display).
3. Notifies every recipient in a configurable list, gated per-recipient by their own `input_boolean` toggle.
4. Optionally speaks the reason via TTS on a media player, unless a Do Not Disturb switch is `on`.
5. After a delay, resets the `input_text` helper back to `"Hide"`.

### Inputs

| Input | Description | Example |
|---|---|---|
| `mqtt_topic` | Topic to listen on | `paradox/interface/command_status` |
| `status_input_text` | input_text helper to hold the current status | `input_text.command_status` |
| `notification_title` | Title shown on push notifications | `Partition arm blocked ❗️` |
| `recipients` | List of `{notify_service, toggle_entity}` pairs | see below |
| `tts_media_player` | Media player entity for TTS announcement | `media_player.cctv_alert` |
| `dnd_switch` | Switch that suppresses TTS when `on` | `switch.do_not_disturb` |
| `clear_delay_seconds` | Seconds before resetting the status text | `10` |

**Recipients format** (entered in the object selector as YAML):
```yaml
- notify_service: notify.mobile_app_franco_samsung
  toggle_entity: input_boolean.franco_notifications
- notify_service: notify.mobile_app_raine_s_a55
  toggle_entity: input_boolean.raine_notifications
```
Add one entry per person. Order doesn't matter.

### Install
1. Save to `/config/blueprints/automation/locknalert/alarm_command_status.yaml`.
2. Create an automation from it in Settings → Automations & Scenes → Blueprints.
3. Fill in the topic, helper entity, recipients list, TTS player, and DND switch.

### Notes
- The reason-extraction logic checks `'reason' in payload` *before* splitting on it, so a payload without a `reason:` field doesn't throw a template error (the original hand-written automation had this ordering backwards).
- If `action: "{{ repeat.item.notify_service }}"` (a templated service call) doesn't fire on your HA core version, you're on an older release that requires literal service names — see the "Notify service compatibility" note at the end of this document.

---

## 3. Alarm Indoor State Change Notifier

### What it does
Fires whenever a chosen `alarm_control_panel` entity changes to `arming`, `armed_home`, `armed_night`, `armed_away`, or `disarmed` (from any prior state, excluding `triggered`/`unavailable`/`unknown` edge cases). It figures out who made the change by matching the triggering user's `context.user_id` against the `person` entities in your house, builds a message like *"Raine has armed the Indoor alarm"*, and sends it to every recipient whose toggle is `on`.

Because the blueprint is generic on `panel_entity`, you create **one automation instance per partition** (Indoor, Outdoor, Garage, etc.) — the blueprint itself doesn't need to change.

### Inputs

| Input | Description | Example |
|---|---|---|
| `panel_entity` | The alarm_control_panel entity to watch | `alarm_control_panel.mg5050_fsrlpartition_indoor` |
| `panel_display_name` | Optional override for the name used in the message | Leave blank to use the entity's friendly name automatically; set e.g. `Indoor` if the entity's actual friendly name is long or awkward to read in a sentence |
| `known_users` | List of person friendly names eligible for "who did it" attribution — anyone else shows as "Home Assistant" | Franco Pretorius, Raine Pretorius, ... |
| `notification_title` | Title on the push notification | `FSrl Alarm State` |
| `recipients` | List of `{notify_service, toggle_entity}` pairs | see below |

**Recipients format** — same shape as the command-status blueprint:
```yaml
- notify_service: notify.mobile_app_raine_s_a55
  toggle_entity: input_boolean.raine_notifications
- notify_service: notify.mobile_app_minette_samsung
  toggle_entity: input_boolean.minette_notifications
```

### Install
1. Save to `/config/blueprints/automation/locknalert/alarm_indoor_state_notifier.yaml`.
2. Create an automation from it — do this **once per partition** you want notifications for.
3. For each instance, pick the relevant `panel_entity`, fill in `panel_display_name` only if the entity's default friendly name doesn't read naturally (e.g. entity name is "MG5050 FSrl Partition Indoor" but you want the message to say "Indoor"), and fill in the recipients list.

### Notes
- `who` falls back to `"Home Assistant"` if the triggering context has no matching person (e.g. the panel changed state via the MQTT bridge with no HA user attached) — this avoids a template error that the original hand-written version was exposed to.
- The verb ("has armed", "is arming", etc.) is looked up from a dict with a fallback string (`"has changed the state of"`) so an unexpected panel state never produces a blank notification body.

---

## Notify service compatibility (applies to all three blueprints)

All three blueprints call `action: "{{ repeat.item.notify_service }}"` — a **templated** service name, resolved per recipient. This requires a Home Assistant core version that supports templated action/service targets (2024.x and later).

If you're on an older core version, this will fail silently or error in the automation trace. To confirm:
```
ha core info
```
If unsupported, the fix is to switch from calling `notify.mobile_app_X` services to targeting **notify entities** instead:
```yaml
- action: notify.send_message
  target:
    entity_id: "{{ repeat.item.notify_entity }}"
  data:
    title: !input notification_title
    message: "{{ message }}"
```
with `recipients` supplying `notify_entity: notify.raine_s_a55` (the entity ID) instead of `notify_service`. Check Settings → Devices & Services → Entities, filter by `notify.`, to find the correct entity IDs on your system.
