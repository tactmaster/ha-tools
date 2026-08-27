# Front Door ZigBee Light — why it does unexpected things

Device `a05e47787934f341f677855462d69eb3` = **Front Door ZigBee Light**
(IKEA TRADFRI E27 white spectrum, 1000 lm, area **Outside**, via MQTT /
Zigbee2MQTT). 12 entities, the relevant one being
`light.front_door_zigbee_light`.

## Three automations touch it

| Automation | Trigger | Action |
|---|---|---|
| `outside_light` ("Outside Light Evening On") | sunset −30min, before 22:00 | turn_on 100% |
| `turn_off_subrise` ("Outside Light Morning Off") | sunrise +30min | turn_off |
| `front_door_switch_to_zigbee` | physical switch, **both** edges | **toggle** |

## The bug: toggle causes phase inversion

`front_door_switch_to_zigbee` fires on both the `powered` and `not_powered`
device triggers and runs `type: toggle`. Toggle flips whatever the light
happens to be, so switch position and bulb state stay in sync only until
something *else* changes the light. Then the physical switch is inverted —
flipping it "on" turns the bulb off — until chance re-syncs them.

**They are out of phase right now:**

```
binary_sensor.front_door_outside_switch_input_0_input = on
light.front_door_zigbee_light                          = off
```

### Logbook evidence

```
2026-08-20 09:35:59  on   <- Front Door Switch to Zigbee   (daytime, 9:35am)
2026-08-20 16:58:35  off  <- Front Door Switch to Zigbee
2026-08-20 18:12:41  on   <- Outside Light Evening On       (correct, sunset)
2026-08-20 20:00:00  off  <- user 59f25b98... (NOT an automation)
2026-08-20 22:01:27  on   <- Front Door Switch to Zigbee   (10pm)
2026-08-20 22:02:17  off  <- Front Door Switch to Zigbee   (50s later)
2026-08-21 07:15:03  on   <- Front Door Switch to Zigbee
```

Read the 20:00 line together with 22:01. The cloud/user action at 20:00 turned
the light off while the physical switch was still in the "on" position. From
that point the switch was inverted, which is why flipping it at 22:01 turned
the light **on** rather than off.

The 09:35 → 16:58 window is the same fault showing as a light left on for
seven hours of daylight.

## Second bug: wifi reconnects toggle the light

The `powered` device trigger fires on any transition **to** `on`, including
`unavailable -> on`. So every time the Shelly drops off wifi and comes back,
the light toggles. Caught twice in history:

```
2026-08-12 06:51:30  binary_sensor unavailable -> off   light -> on
2026-08-22 14:03:23  binary_sensor -> unavailable
2026-08-22 14:06:27  binary_sensor -> on                light -> on  (14:06:29)
```

## Third issue: dead device_id in the trigger

The trigger references `device_id: 8be3085c6027ad399bd7d2942d7aa067`, which is
**not in the device registry**. It still works only because the entity
registry id next to it (`448a7e93…` ->
`binary_sensor.front_door_outside_switch_input_0_input`) resolves. Fragile.

Likewise the action's `entity_id: ca7d1a63…` is an entity *registry id*
resolving to `light.front_door_zigbee_light`.

## The fix

`front_door_switch_to_zigbee.yaml` in this folder:

- `light.turn_on` / `light.turn_off` matched to switch position instead of
  `toggle` — idempotent, so phase drift becomes impossible
- explicit `from: "off"` / `from: "on"` so `unavailable -> on` no longer fires
- plain entity_ids, no dead device_id

Install it, then flip the switch twice. The light should follow the switch
position exactly, in both directions.

### One assumption to confirm

This assumes the Shelly input is a **latching** switch (state follows the
physical position), which the timing supports — gaps of 50 seconds and
7 hours between edges, rather than sub-second button pulses.

If it is actually a **momentary push-button**, this fix would make the light
only stay on while pressed, and the automation should instead toggle on a
single edge:

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.front_door_outside_switch_input_0_input
    from: "off"
    to: "on"
actions:
  - action: light.toggle
    target:
      entity_id: light.front_door_zigbee_light
```

Toggling on **one** edge only is still far better than the current both-edges
version, though a momentary button can't avoid drift entirely.

## SOLVED: the 20:00 turn-off is Node-RED

User `59f25b988b164305a90564a7e1e0878a` = **Supervisor** (a system-generated
account). Calls attributed to Supervisor come from an **add-on**, and you have
the **Node-RED** add-on loaded (`config_entry domain=nodered`, plus 7
`nodered.*` entities).

So a Node-RED flow is turning this light off at **22:00 local**. That is
outside HA's automation store entirely, which is why searching all 134
automations found nothing.

**Find it in Node-RED**: open the add-on UI and look for a flow with a 22:00
inject/schedule node targeting `light.front_door_zigbee_light`. Either remove
it or keep it and drop the HA-side actor — but do not leave both.

### Timezone note

All logbook timestamps are UTC; your HA runs at UTC+2. So the "20:00" event is
**22:00 local**, and the observed sequence reads:

| UTC | Local | What |
|---|---|---|
| 18:12 | 20:12 | Outside Light Evening On (sunset −30) |
| 20:00 | 22:00 | **Node-RED turns it off** |
| 22:01 | 00:01 | switch toggled -> light **on** (inverted) |
| 22:02 | 00:02 | switch toggled -> off |

The light coming on just after midnight is the phase inversion, and Node-RED's
22:00 action is what knocks it out of phase each night.

## Six actors on one light

This is the real problem. The cross-check against scenes found three more
actors my automation-only search had missed, because they reach the light
*via scenes* rather than naming it:

| Actor | When | Effect | Route |
|---|---|---|---|
| `automation.outside_light` | sunset −30, before 22:00 | on 100% | direct |
| `automation.turn_off_subrise` | sunrise +30 | off | direct |
| `automation.front_door_switch_to_zigbee` | switch, both edges | **toggle** | direct |
| `automation.turn_light_off` | 22:05 local | off | -> `scene.dark_and_awake` |
| `automation.outside_light_730` | 07:00, before sunrise+30 | on | -> `scene.dark_and_wake` |
| `automation.morning_is_here` | 06:00, before sunrise | on | -> `scene.morning` |
| Node-RED flow | 22:00 local | off | Supervisor API |

Plus two scenes that touch the light but no automation activates:
`scene.evening` (last applied 2026-08-24 18:30 UTC) and `scene.morning_off`
(2026-08-25 03:59 UTC). Something activates those too — most likely the same
Node-RED instance, or the Touchscreen / Bird Watch user accounts.

Overlaps worth noting:

- **Two morning-off paths**: `scene.morning_off` at 03:59 and
  `turn_off_subrise` at sunrise+30 (04:26)
- **Two evening-on paths**: `scene.evening` at 18:30 and `outside_light` at
  sunset−30 (18:12)
- **Two night-off paths**: Node-RED at 22:00 and `turn_light_off` at 22:05

Every one of those pairs is a chance for the toggle automation to invert.

## Also found: scene.bathroom_dark is broken

`automation.turn_on_bathroom_dark` fires at 22:00 local and applies
`scene.bathroom_dark`, whose only entity is `light.color_temperature_light_11`
— which **does not exist** in the registry or in states. That automation runs
nightly and does nothing.

Its 22:00 trigger is what first made it look like the culprit; the exact
timestamp match with the Node-RED action is coincidence.

## Coverage note

29 of 134 automations could not be fetched over REST (28 `sungrow_inverter_*`
plus `bathroom_floor_heating`), nor could 5 of 18 scripts
(4 `sg_*` Sungrow scripts + `notification_message_plants`). All 20 scenes were
readable. The unreadable ones are YAML-defined; none appear related to this
light, but Node-RED flows are invisible to this method entirely.
