# Heating automations

Pulled 2026-08-17 from `ha.edmundwatson.com` via read-only GET.
19 automations + 5 schedules carry the `Heating` label.

One file per automation. `_all-heating-automations.yaml` has everything in one place.

## Index

| Automation | State | Last triggered | Mode | T/C/A |
|---|---|---|---|---|
| [downstairs_ac](downstairs_ac.yaml) | off | 2026-06-25 08:45:38 | single | 1/1/1 |
| [children_heating_off](children_heating_off.yaml) | on | 2026-05-04 05:30:00 | single | 1/1/1 |
| [cooling_downstair_off](cooling_downstair_off.yaml) | on | 2026-08-24 19:13:47 | single | 1/0/1 |
| [cooling_off](cooling_off.yaml) | on | never | single | 1/0/1 |
| [floor_heating_off](floor_heating_off.yaml) | on | 2026-08-25 06:00:00 | single | 1/0/1 |
| [floor_heating_on](floor_heating_on.yaml) | on | 2026-05-04 13:30:00 | single | 1/1/1 |
| [heating_children_evening_on](heating_children_evening_on.yaml) | on | 2026-05-04 15:00:00 | single | 1/1/2 |
| [heating_kitchen_on](heating_kitchen_on.yaml) | on | 2026-05-04 15:00:00 | single | 1/1/2 |
| [heating_kitchen_turn_off](heating_kitchen_turn_off.yaml) | on | 2026-05-04 05:30:00 | single | 1/1/2 |
| [heating_morning_turn_off](heating_morning_turn_off.yaml) | on | never | single | 1/1/1 |
| [heating_nighttime](heating_nighttime.yaml) | on | never | single | 1/1/3 |
| [heating_upstairs](heating_upstairs.yaml) | on | 2026-05-04 16:00:00 | single | 2/1/2 |
| [heating_working_on](heating_working_on.yaml) | on | 2026-05-04 05:30:00 | single | 1/1/2 |
| [new_automation_10](new_automation_10.yaml) | on | 2026-08-25 06:00:00 | single | 1/0/1 |
| [new_automation_3](new_automation_3.yaml) | on | 2026-05-04 13:30:00 | single | 1/1/2 |
| [turn_off_lufty](turn_off_lufty.yaml) | on | 2026-05-04 05:30:00 | single | 1/1/1 |
| [upstairs_aircontioning_summer](upstairs_aircontioning_summer.yaml) | on | 2026-08-24 21:37:04 | single | 1/1/4 |
| [utility_room_heater_off](utility_room_heater_off.yaml) | on | 2026-08-25 06:00:00 | single | 1/0/1 |
| [wake_luftwarmer](wake_luftwarmer.yaml) | on | 2026-05-04 14:30:00 | single | 1/1/4 |

T/C/A = triggers / conditions / actions

## Entities each one controls

**children_heating_off** — Children Heating Off

- `input_boolean.heating`
- `schedule.children_evening_heating`

**cooling_downstair_off** — Cooling Downstair Off

- (no explicit entity_id targets — device_id or template based)

**cooling_off** — Cooling off

- (no explicit entity_id targets — device_id or template based)

**downstairs_ac** — Downstairs AC 

- `input_select.season`
- `sensor.bens_room_bt_sensor_temperature`
- `sensor.bens_room_heater_temperature`
- `sensor.downylufymund_inside_temperature`
- `sensor.easyweatherv1_6_5_indoor_temperature`
- `sensor.ellies_room_bt_sensor_temperature`

**floor_heating_off** — Floor Heating Off

- `schedule.floor_heating`

**floor_heating_on** — Floor Heating On

- `input_boolean.heating`
- `schedule.floor_heating`

**heating_children_evening_on** — Heating - Children Evening on

- `input_boolean.heating`
- `schedule.children_evening_heating`

**heating_kitchen_on** — Heating - Kitchen On

- `input_boolean.heating`
- `schedule.heating_kitchen`

**heating_kitchen_turn_off** — Heating - Kitchen Turn Off

- `input_boolean.heating`
- `schedule.heating_kitchen`

**heating_morning_turn_off** — Heating - Morning Turn off

- `input_boolean.heating`
- `schedule.heating_night`

**heating_nighttime** — Heating - Nighttime 

- `input_boolean.heating`
- `schedule.heating_night`

**heating_upstairs** — Heating Upstairs

- `input_boolean.heating`
- `schedule.upstairs_heating`

**heating_working_on** — Heating - Working On

- `input_boolean.heating`
- `schedule.working`

**new_automation_10** — Utility Room Floor Heating 

- `schedule.utility_room_floor_heating`

**new_automation_3** — Heating - Working Off

- `input_boolean.heating`
- `schedule.working`

**turn_off_lufty** — Heating - Turn off Lufty

- `input_boolean.heating`
- `schedule.wake_up_schedule`

**upstairs_aircontioning_summer** — Upstairs Aircontioning Summer

- `input_select.season`
- `sensor.atc_ed38_temperature`
- `sensor.spare_room_sensor_temperature`

**utility_room_heater_off** — Utility Room Heater Off

- `schedule.utility_room_floor_heating`

**wake_luftwarmer** — Heating - Wake Luftwarmer

- `climate.daikin`
- `input_boolean.heating`
- `schedule.wake_up_schedule`

## Schedules

- `schedule.children_evening_heating` = **off**
- `schedule.heating_kitchen` = **off**
- `schedule.utility_room_floor_heating` = **off**
- `schedule.wake_up_schedule` = **off**
- `schedule.working` = **on**

See `_schedules.yaml`. Time blocks aren't in the REST API — check the UI for those.
