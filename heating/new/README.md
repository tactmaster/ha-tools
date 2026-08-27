# Bathroom floor heating — consolidation

## What this replaces

| Old automation | unique_id | Status |
|---|---|---|
| `automation.floor_heating_on` | 1736024879431 | **broken** — targets a dead device_id |
| `automation.floor_heating_off` | 1736024940221 | **broken** — targets a dead device_id |

Both used `device_id` targets that are no longer in the device registry:

- `6d5793b89ba7e2a0b95eda8a4d6600cc` — not found
- `b1b0827d1652fadcbbd3851a803c7715` — not found

The real devices are `23333922284946f7299d2e41ef0a76a5` (Bathroom Heater) and
`57aab94bad8f5661f6b76ff985dde7e4` (Downstair Bathroom Heater). The new
automation targets **entity_ids**, which survive re-pairing.

## The rule

```
heaters ON   <=>  schedule.floor_heating = on  AND  input_boolean.bathroom_floor_heating = on
anything else -> heaters OFF
```

Two automations became one because both were computing the same thing from
opposite ends. One `choose` block expresses it directly.

## Entities used

| Entity | Role | State when pulled |
|---|---|---|
| `schedule.floor_heating` | when heating is allowed | off |
| `input_boolean.bathroom_floor_heating` | master enable | off |
| `switch.bathroom_heater_switch_0` | upstairs heater | off |
| `switch.downstair_bathroom_heater_switch_0` | downstairs heater | off |

`input_boolean.heating` is intentionally **not** used here — it stays for your
other heating automations.

## Manual override behaviour

Asymmetric, by design:

| You do this | Result |
|---|---|
| Turn a heater **off** while the rule says heat | reverted back on after ~3s |
| Turn a heater **on** while the rule says off | **allowed** — stays on |

So a heater can't be accidentally left off during a heating window, but you can
still boost outside the schedule. A manual boost holds until the next
schedule or toggle change, which then reasserts the rule.

The 3-second `for:` delay stops the automation reacting to its own switching.
`mode: single` + `max_exceeded: silent` stops stacked triggers looping or
filling the log.

Note the Shelly's physical button now works as a boost (on sticks) but can't
turn a heater off during a heating window.

If you later want manual-on reverted too, add this trigger back:

```yaml
  - trigger: state
    entity_id:
      - switch.bathroom_heater_switch_0
      - switch.downstair_bathroom_heater_switch_0
    to: "on"
    for: "00:00:03"
    id: manual_on
```

## How to install

1. Settings > Automations & scenes > **+ Create automation** > Create new >
   three-dot menu > **Edit in YAML**
2. Paste `bathroom_floor_heating.yaml` (drop the comment header if you like)
3. Save, name it **Bathroom Floor Heating**, add the `Heating` label
4. Test before deleting the old ones:
   - turn `input_boolean.bathroom_floor_heating` on while the schedule is on
     -> both switches should go on
   - turn the toggle off -> both should go off
5. Once confirmed, **delete** `Floor Heating On` and `Floor Heating Off`

Keep the old two disabled rather than deleted for a few days if you prefer a
fallback — though since they target dead device_ids they do nothing anyway.

## Worth checking separately

`schedule.heating_night` **does exist** (state: off) — it just isn't carrying the
`Heating` label, which is why it didn't appear in the earlier label pull. So
`automation.heating_nighttime` is wired to a real schedule.

Its `last_triggered` is still null though, and it targets
`device_id: d5574f4e3d430263327013652128fb62`. Worth checking whether that
device_id is also stale — same failure mode as the floor heating pair.

Two schedules exist but are unlabelled: `schedule.heating_night` and
`schedule.upstairs_heating`. Adding the `Heating` label would make future
label-based pulls complete.
