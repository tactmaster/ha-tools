# What can be removed — audit of the 19 Heating automations

All 19 configs re-fetched and every action target resolved against the live
device + entity registries.

## Remove now — replaced by Bathroom Floor Heating

| Automation | unique_id | Working actions |
|---|---|---|
| `automation.floor_heating_on` | 1736024879431 | **0 of 1** |
| `automation.floor_heating_off` | 1736024940221 | **0 of 1** |

Both target only dead device_ids (`6d5793b8…`, `b1b0827d…`). They have been
firing on schedule and doing nothing. Nothing else references them. Delete.

## Remove or fix — dead, does nothing

| Automation | unique_id | Problem |
|---|---|---|
| `automation.wake_luftwarmer` | 1664978776780 | **0 of 4** actions work |

Every action targets device `ed758bd9…` (gone) or `climate.daikin` (does not
exist). Last fired 2026-05-04 and did nothing.

Your actual Daikin climate entities are now named differently:

```
climate.daikin_room_temperature
climate.daikinap10698_room_temperature
```

So this broke when the Daikin integration was re-added under new names.
Either re-point it at the right entity or delete it.

Its partner `automation.turn_off_lufty` still half-works (1 of 2 device
targets alive), so check that one too.

## CONFLICT with the new automation — decide before deleting

These two still work, and they control **the same switch** the new automation
manages (`switch.bathroom_heater_switch_0`):

| Automation | unique_id | Does |
|---|---|---|
| `automation.heating_working_on` | 1760351004302 | `schedule.working` on -> heater **ON** |
| `automation.new_automation_3` ("Heating - Working Off") | 1760351039610 | `schedule.working` off -> heater **OFF** |

They reach the switch through the legacy device-action format, where the
`entity_id` field holds an entity *registry id* (`f8aee990…`) rather than an
entity_id. That registry id resolves to `switch.bathroom_heater_switch_0`, so
the switching still happens even though the `device_id` beside it is dead.

**The clash:** when "Working Off" turns the bathroom heater off while
`schedule.floor_heating` says heat, the new automation's `manual_off` trigger
reverts it back on ~3s later. They will fight.

Three ways out:

1. **Bathroom heater follows floor_heating only** — delete both Working
   automations, or edit them to drop the bathroom-heater action and keep only
   their climate action.
2. **Bathroom heater follows schedule.working** — point the new automation at
   `schedule.working` instead of `schedule.floor_heating`, then delete both.
3. **Both schedules should heat it** — change the new automation's condition to
   an `or` across both schedules, then delete both.

Their climate action (device `9f2b20ef…`) is dead in both cases, so that half
does nothing regardless.

## Keep — these work

| Automation | Note |
|---|---|
| `children_heating_off` | 2 live device targets |
| `cooling_downstair_off` | ok |
| `cooling_off` | ok, but `last_triggered` is null — never fired |
| `downstairs_ac` | area/floor targets |
| `heating_children_evening_on` | ok |
| `heating_kitchen_on` | ok |
| `heating_kitchen_turn_off` | ok |
| `heating_morning_turn_off` | ok, never fired |
| `heating_nighttime` | ok, never fired |
| `heating_upstairs` | legacy device-actions, both resolve |
| `new_automation_10` ("Utility Room Floor Heating") | ok — worth renaming |
| `upstairs_aircontioning_summer` | 3 of 4 work (floor/area targets) |
| `utility_room_heater_off` | ok |

## One fix repairs three automations

Dead device `9f2b20efb917dd436d983286c23118c0` is referenced by
`heating_working_on`, `new_automation_3` and `upstairs_aircontioning_summer` —
in each case a `climate.set_temperature` / `climate.turn_on` that silently
does nothing. Re-point those three at the right climate entity and all three
regain their climate half.

Other dead device_ids: `b1b0827d…` (4 automations), `6d5793b8…` (2),
`ed758bd9…` (2), `47afe5d1…` (1), `ad5cc49e…` (1).

## Never fired

`cooling_off`, `heating_morning_turn_off`, `heating_nighttime` all have
`last_triggered: null` despite valid targets. Their triggers are probably
never satisfied — worth checking the schedules they watch.
