# Front door light — full conflict map (HA + Node-RED)

## The bug in the Node-RED flow

```
inject "Evening 22:00"  cron 00 22 * * *   ->   "Morning Off"  ->  scene.morning_off
```

A node **labelled "Evening"** calls the **Morning Off** scene at 22:00. That is
the mystery 22:00-local turn-off traced to the Supervisor user. The label says
evening; the action is the morning scene.

`scene.morning_off` sets `light.front_door_zigbee_light` -> **off**, and is
called from two places in the flow:

| Path | Fires |
|---|---|
| suncalc out2 (`start=sunrise`, `end=goldenHour`) | ~sunrise — correct |
| inject "Evening 22:00" | 22:00 — **wrong scene** |

A second naming trap: the node named **"Dark and awake scene"** actually calls
`scene.dark_and_wake` (on), while `scene.dark_and_awake` is a *different*
scene that turns things **off**. Two scenes one letter apart with opposite
effects, and the node name points at the wrong one.

## Everything that moves this light

Times are **local** (your HA runs UTC+2).

| # | Actor | Where | When | Effect |
|---|---|---|---|---|
| 1 | `automation.outside_light` | HA | sunset −30, if before 22:00 | **on** 100% |
| 2 | `automation.turn_off_subrise` | HA | sunrise +30 | **off** |
| 3 | `automation.front_door_switch_to_zigbee` | HA | switch, **both** edges | **toggle** |
| 4 | `automation.turn_light_off` | HA | 22:05 | **off** via `scene.dark_and_awake` |
| 5 | `automation.outside_light_730` | HA | 07:00, if before sunrise+30 | **on** via `scene.dark_and_wake` |
| 6 | `automation.morning_is_here` | HA | 06:00, if before sunrise | **on** via `scene.morning` |
| 7 | Node-RED "Morning 7:00" | NR | 07:00, if sunset..sunrise | **on** via `scene.dark_and_wake` |
| 8 | Node-RED "Evening Scene" | NR | sunset, if 12:00–22:00 | **on** via `scene.evening` |
| 9 | Node-RED "Morning Off" | NR | ~sunrise | **off** via `scene.morning_off` |
| 10 | Node-RED "Evening 22:00" | NR | 22:00 | **off** via `scene.morning_off` |

Ten actors on one bulb. `scene.bedtime` (23:55) does **not** touch it.

## The exact duplicates

**1. 07:00 morning-on — identical**

`automation.outside_light_730` and Node-RED "Morning 7:00" both fire at 07:00
and both apply `scene.dark_and_wake`. Same time, same scene, two systems.
One is pure redundancy.

**2. Evening-on — 30 minutes apart**

- Node-RED at sunset -> `scene.evening` (on)
- HA `outside_light` at sunset −30 -> on

HA fires first, Node-RED re-applies 30 min later.

**3. Morning-off — 30 minutes apart**

- Node-RED at ~sunrise -> `scene.morning_off` (off)
- HA `turn_off_subrise` at sunrise +30 -> off

**4. Night-off — 5 minutes apart**

- Node-RED at 22:00 -> `scene.morning_off` (off)
- HA `turn_light_off` at 22:05 -> `scene.dark_and_awake` (off)

## Why this breaks the switch

Each of those pairs changes the light without touching the physical switch, so
the switch and bulb fall out of phase. Because
`front_door_switch_to_zigbee` uses **toggle**, the switch then does the
opposite of what you expect until something re-syncs by luck.

They are out of phase right now:

```
binary_sensor.front_door_outside_switch_input_0_input = on
light.front_door_zigbee_light                          = off
```

That is why the light came on at 00:01 local — the switch was flipped while
inverted.

## Fix, in order

1. **Install the toggle fix** (`front_door_switch_to_zigbee.yaml` here).
   Explicit on/off matched to switch position makes inversion impossible,
   regardless of how many other actors exist.

2. **Fix or delete the "Evening 22:00" -> "Morning Off" wire.** If you wanted
   an evening scene at 22:00, point it at `scene.evening`. If you wanted the
   light off at 22:00, keep it but delete HA's `turn_light_off` (22:05) — they
   do the same job 5 minutes apart.

3. **Pick one system per event.** Suggested split — keep Node-RED for
   sun-based events (it already has proper suncalc nodes) and delete the HA
   duplicates:

   | Event | Keep | Delete |
   |---|---|---|
   | 07:00 on | Node-RED "Morning 7:00" | `automation.outside_light_730` |
   | evening on | Node-RED "Evening Scene" | `automation.outside_light` |
   | morning off | Node-RED "Morning Off" | `automation.turn_off_subrise` |
   | night off | Node-RED 22:00 (repointed) | `automation.turn_light_off` |

   Or the reverse — but not both.

4. **`automation.morning_is_here`** (06:00 -> `scene.morning`, light on) has no
   Node-RED counterpart and last ran 2026-04-15. Probably obsolete: it turns
   the light on an hour before the 07:00 actors.

5. **Two coordinate sets** in the flow: `57.57619,12.09111` and
   `57.58015,12.10066`. ~450m apart, so sun times differ by seconds — harmless,
   but worth making consistent.

## Unrelated: dead entities in 12 scenes

Turned up while checking. These scenes reference entities that no longer exist,
so those lines silently do nothing:

| Scene | Dead entities |
|---|---|
| `red_alert` | 6 (`light.wled`, `switch.wled_*`, `light.living_room`, `light.corner_light_2`) |
| `yellow_alert` | 6 (same WLED set, `light.color_light_5`) |
| `on_holiday` | 3 (`switch.tumbler_dryer`, power strip socket, `light.corner_light`) |
| `return_holiday` | 3 |
| `bedtime` | 2 (`remote.harmony_hub`, `select.harmony_hub_activities`) |
| `movie` | 2 (`light.color_temperature_light_8/7`) |
| `back_home` | 2 |
| `leave_the_house`, `upstairs_off` | `switch.shelly_landinglight_switch_0` |
| `enter_house` | `light.corner_light_3` |
| `bathroom_dark` | `light.color_temperature_light_11` — its **only** entity |

`scene.bathroom_dark` is entirely dead, and `automation.turn_on_bathroom_dark`
applies it nightly at 22:00 for no effect. Its 22:00 trigger is what first
looked like the culprit here — coincidence.

The WLED entities appearing dead across two scenes suggests that integration
was removed or renamed.
