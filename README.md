# ha-tools

Ed's collection of Home Assistant blueprints, automations, and scripts.

---

## Blueprints

Blueprints live in `blueprints/automation/`.  
Import them via **Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste the raw GitHub URL.

### IKEA BILRESA Dual Button – Light Control (Matter)

**File:** `blueprints/automation/bilresa_dual_button_light_control.yaml`

Controls a single light with the IKEA BILRESA two-button remote paired via **Matter**.

The BILRESA exposes **two** `event` entities in HA — one per physical button.
Select each entity in the blueprint inputs.

| Action | Result |
|---|---|
| Short press **top** button | Toggle light on / off |
| Short press **bottom** button | Turn light off |
| Long press **top** button | Increase brightness |
| Long press **bottom** button | Decrease brightness |

Configurable inputs: top button event entity, bottom button event entity, target light, brightness step (%).

---

### IKEA BILRESA Scroll Wheel – Light Control (Matter)

**File:** `blueprints/automation/bilresa_scroll_wheel_light_control.yaml`

Controls a single light with the IKEA BILRESA scroll-wheel dimmer paired via **Matter**.  
The wheel emits event type **`step_up`** on clockwise rotation (corresponding to Zigbee/Matter
Level Control command `step_with_on_off`, action code **32768 / 0x8000**).

| Action | Event type | Result |
|---|---|---|
| Press wheel | `single_press` | Toggle light on / off |
| Rotate **clockwise** | `step_up` | Increase brightness |
| Rotate **counter-clockwise** | `step_down` | Decrease brightness |
| Absolute level (fallback) | `move_to_level` | Set brightness directly |

Configurable inputs: scroll wheel event entity, target light, brightness step (%).

---

## Automations

Standalone automations live in `automations/`.  
Copy the file into your `<config>/automations/` directory and reload, or paste into the Automation UI editor.

### HACS – Upgrade All

**File:** `automations/hacs_upgrade_all.yaml`

Automatically installs every pending HACS update each night at 03:00.

- Refreshes all HACS `update` entities first.
- Installs each pending update one by one (errors in one repo don't block others).
- Logs the list of installed updates to the HA system log.
- Can also be triggered manually at any time.

**Requirements:** HA 2022.4+, HACS 1.26+ (must expose `update` entities).
