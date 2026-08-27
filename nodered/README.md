# Node-RED flows

Exported flows from the Node-RED add-on. Node-RED runs as an add-on, so its
service calls appear in the HA logbook as the **Supervisor** user with no
`context_entity_id` — which makes them invisible when searching HA automations.
That is why anything odd should be checked here too.

## `lights-flow.json` — tab "Lights"

Four `api-call-service` nodes, all `scene.turn_on`:

| Node | Scene | Triggered by |
|---|---|---|
| "Dark and awake scene" | `scene.dark_and_wake` | inject 07:00, gated sunset..sunrise |
| "Evening Scene" | `scene.evening` | suncalc at sunsetStart, gated 12:00–22:00 |
| "Morning Off" | `scene.morning_off` | suncalc at sunrise **and** inject 22:00 |
| "Bedtime" | `scene.bedtime` | inject 23:55 |

### Known bug: "Evening 22:00" calls the morning scene

```
inject "Evening 22:00"  (cron 00 22 * * *)  ->  "Morning Off"  ->  scene.morning_off
```

The node is labelled evening but applies `scene.morning_off`, which turns the
front door light **off** at 22:00. This is the 22:00 turn-off that took a while
to track down — it does not exist anywhere in HA's automations.

Either repoint it at `scene.evening`, or keep it as an off-switch and delete
HA's `automation.turn_light_off` (22:05), which does the same thing 5 minutes
later.

### Also misleading

The node named **"Dark and awake scene"** calls `scene.dark_and_wake` (turns
the light **on**). A separate scene `scene.dark_and_awake` turns things
**off**. Two scenes one letter apart with opposite effects.

### Minor

Two different coordinate pairs are used: `57.57619,12.09111` and
`57.58015,12.10066` — about 450m apart, so sun times differ by seconds.
Harmless, but worth making consistent.

See `../frontdoor/CONFLICTS.md` for how these overlap with the HA automations.
