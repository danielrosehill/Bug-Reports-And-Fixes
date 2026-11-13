# Zigbee2MQTT stays dead after a reboot when the MQTT broker starts after it, and Home Assistant still shows the bridge as online

**Date:** 2026-11-13
**Status:** Confirmed (observed in production; root cause traced in logs and Z2M source)
**Severity:** High. An ordinary host reboot left the whole Zigbee network down for about 2 hours, with no notification, while Home Assistant's own bridge sensor said Zigbee2MQTT was connected.
**Related:** [2026-11-06: no self-recovery after a coordinator outage](../2026-11-06-tcp-coordinator-outage-no-self-recovery/). Same Supervisor watchdog limit, different trigger, plus a second failure that hides it.
**Upstream:** not filed yet.

## Summary

The host rebooted. Home Assistant OS came back before the external Mosquitto broker it uses. Zigbee2MQTT started, could not reach the broker, and exited. The Supervisor watchdog used its 10 restarts in 42 seconds and then gave up. When the broker came up 2¾ minutes later, nothing started Z2M again.

Home Assistant then reconnected to the broker and received the **retained `zigbee2mqtt/bridge/state: online`** message from before the reboot. `binary_sensor.zigbee2mqtt_bridge_connection_state` went to **on** and stayed on for the whole outage. Every Zigbee entity read `unknown`, and the bridge sensor said all was well.

The visible symptom was a wall switch "doing nothing". The first assumption was that a Shabbat-mode automation was overriding it, which is exactly the kind of wrong turn this failure invites.

## Environment

| Component | Version |
|---|---|
| Zigbee2MQTT add-on | 2.14.3-1 |
| Home Assistant Core | 2026.11.1 |
| Home Assistant OS | 18.4 |
| Supervisor | 2026.11.0 |
| MQTT broker | Mosquitto 2.1.2 in Docker (`restart: unless-stopped`) on a separate Ubuntu host, `mqtt://<broker>:1883` |
| Coordinator | SMLIGHT SLZB-MR3U (`tcp://`), Ember. Healthy throughout |
| Add-on options | Supervisor watchdog on, start on boot on |

The HA VM and the broker host went down together, so they came back at the same time and raced.

## Timeline (2026-11-13, UTC+2)

| Time | Event |
|---|---|
| 19:12:31 | Broker host boots (kernel) |
| 19:13:04 | Docker daemon up on the broker host; it starts restarting its ~60 `unless-stopped` containers |
| 19:13:16 | Supervisor attaches to Z2M; first exit (code 255) |
| 19:13:54 to 19:14:39 | Z2M starts, joins the coordinator (24 devices), fails on MQTT with `ECONNREFUSED`, exits code 1. Watchdog restarts it 10 times |
| 19:14:43 | Final exit. Watchdog throttled; add-on state `error` |
| 19:17:29 | Mosquitto container starts and opens port 1883 |
| 19:17:38 | HA's MQTT client reconnects; retained `bridge/state` = `online` arrives; `binary_sensor.zigbee2mqtt_bridge_connection_state` → **on** |
| 19:17 to 21:23 | Z2M dead, add-on `error`, bridge sensor **on**, every Zigbee light and sensor `unknown` |
| 21:23:11 | Add-on started by hand; Z2M connects at once |

The workaround automation from the 2026-11-06 report was installed and enabled, and did **not** fire. Its `last_triggered` was still 2026-11-06. It was gated on the bridge sensor being off.

Evidence (broker IP removed):

- [`evidence/supervisor-watchdog.log`](evidence/supervisor-watchdog.log): every exit and the 10 watchdog restarts
- [`evidence/z2m-last-failed-start.log`](evidence/z2m-last-failed-start.log): the last failed start

## Why it happens

Four behaviours combine. Each is defensible on its own; together they produce an outage that looks like misconfiguration.

1. **Z2M treats a missing broker at startup as fatal, but a missing broker later as routine.** In `lib/mqtt.ts`, `connect()` makes one `connectAsync()` call. If it fails, the error propagates and the controller logs `MQTT failed to connect, exiting...`. After a successful first connect, mqtt.js reconnects automatically and indefinitely. So Z2M rides out a broker restart while running, but not a broker that is a minute late at boot.
2. **The Supervisor watchdog gives up silently.** 10 restarts per 30 minutes (`WATCHDOG_THROTTLE_MAX_CALLS` / `WATCHDOG_THROTTLE_PERIOD`), spent in 42 seconds here, and nothing retries after the window expires. No notification or repair issue is raised. This is the same limit as in the 2026-11-06 report and [home-assistant/supervisor#7190](https://github.com/home-assistant/supervisor/issues/7190).
3. **There is no way to say "start after the broker".** Add-ons can declare a dependency on the Mosquitto *add-on* (`services: mqtt`), but not on an external broker. Boot order between the HA VM and anything else is left to luck.
4. **The bridge's online state is a retained message that only Z2M can clear.** Z2M publishes `bridge/state: online` retained, and registers `offline` as its MQTT last will. A last will only exists for a client that connected. A Z2M process that never reaches the broker leaves no will, so the broker (with persistence) replays the last `online` from before the reboot. Home Assistant's MQTT-discovered `binary_sensor.zigbee2mqtt_bridge_connection_state` therefore reports **on** for a Z2M that is not running.

Point 4 turns a recoverable failure into a misleading one. It defeats any monitoring or recovery that trusts the bridge sensor, including the original version of the workaround automation.

## Workaround

Gate recovery on the Supervisor's view of the add-on, not on MQTT:

- Enable `binary_sensor.zigbee2mqtt_running` (Supervisor integration; **disabled by default**).
- Start the add-on (`hassio.app_start`) when it has been stopped for 2 minutes **or** the bridge has been offline for 2 minutes, and check every 2 minutes. If the broker is still down the start fails harmlessly and the next check tries again, so Z2M effectively waits for the broker.

This is v2 of the workaround in [`../2026-11-06-tcp-coordinator-outage-no-self-recovery/workaround-ha-automation.yaml`](../2026-11-06-tcp-coordinator-outage-no-self-recovery/workaround-ha-automation.yaml), and of [Zigbee2MQTT-Coordinator-Auto-Recovery](https://github.com/danielrosehill/Zigbee2MQTT-Coordinator-Auto-Recovery).

Verified: with Z2M running, triggering the automation with conditions evaluated does nothing. **Not yet verified** against a real reboot.

## What would fix it upstream

- **Z2M:** retry the initial MQTT connection (mqtt.js already supports `reconnectPeriod`) instead of exiting, or at least make the startup behaviour match the runtime behaviour. A broker that is late at boot is the normal case on any multi-host setup.
- **Supervisor:** don't abandon an add-on permanently after the throttle; back off and retry, and raise a repair issue when an add-on with the watchdog on is left in `error` (supervisor#7190).
- **Home Assistant / Z2M discovery:** the bridge connectivity sensor should not be able to read `on` for a process that isn't running. Options: Z2M publishing `offline` (retained) on clean exit paths where it never connected is impossible by definition, so the fix belongs on the consumer side, for example HA combining the retained state with the add-on running state, or Z2M publishing `bridge/state` with a message expiry so a stale `online` ages out.

## Lesson for anyone debugging this

If Zigbee devices all show `unknown` or don't respond, but the Zigbee2MQTT bridge sensor says connected, **check the add-on's actual state** (Settings > Add-ons, or `ha apps info <slug>`). Don't trust the bridge sensor after a reboot.
