# ha-tools

Ed's tools and audit notes for Home Assistant (`ha.edmundwatson.com`, HA 2026.8.1).

Working notes from maintenance sessions: exported automations, conflict maps,
and cleanup plans. Everything here was produced by reading the HA REST and
WebSocket APIs — no config was written by these tools.

## Layout

| Path | What |
|---|---|
| `heating/` | All 19 automations labelled `Heating`, exported as YAML, one file each |
| `heating/_all-heating-automations.yaml` | Same 19 in a single file |
| `heating/_schedules.yaml` | The 5 `Heating`-labelled schedule helpers |
| `heating/new/` | Consolidated replacements + migration and removal notes |
| `frontdoor/` | Front door ZigBee light: conflict analysis across HA and Node-RED |
| `entity-cleanup.md` | Registry audit — which entities are safe to delete |
| `morning.yaml` | Standalone morning TTS automation |

## Key findings so far

**Registry bloat.** Started at 10,052 entities with 64% unavailable. The bulk
was a Glances integration pointed at `localhost` generating a sensor per Docker
`veth` interface — names change on every container restart, so each restart
orphaned hundreds. Down to ~5,200 after deleting Glances and iBeacon.
See `entity-cleanup.md` for the remaining ~3,100 safe deletions.

**Slow dashboards are the logbook, not the database.** Same 24h window:
history API for all entities returns in 0.003s, logbook takes 31s. The logbook
builds human-readable narrative for ~40,000 entries — CPU-bound Python, not
SQL. Recorder runs on MariaDB and is healthy. Fix is on the dashboard: narrow
or remove Logbook cards.

**Stale `device_id` targets silently break automations.** `device_id` does not
survive re-pairing, and HA gives no warning. `floor_heating_on` and
`floor_heating_off` had been firing on schedule and doing nothing for months.
Prefer `entity_id` in automations. `heating/new/REMOVALS.md` lists every
affected automation.

**Ten actors on one light.** The front door light is driven by 6 HA automations
and 4 Node-RED paths, several duplicating each other. Combined with a `toggle`
action, the physical switch drifts out of phase and does the opposite of what
you expect. See `frontdoor/CONFLICTS.md`.

## Conventions

- **Never commit secrets.** `.gitignore` covers `.ha-token`, `secrets.yaml`,
  `*.key`, `*.pem`. API tokens are full-admin — keep them out of the repo.
- Exported automation files carry a header comment with entity_id, alias,
  current state and `last_triggered` at export time.
- `heating/new/` and similar `new/` folders hold proposals, not what is live.
