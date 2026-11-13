<!-- Destination: Koenkk/zigbee2mqtt, Feature request template -->
**Title:** [Feature request]: Retry the initial MQTT connection instead of exiting when the broker isn't up yet

### Is your feature request related to a problem? Please describe

If the MQTT broker isn't reachable when Z2M starts, Z2M logs `MQTT failed to connect, exiting... (connect ECONNREFUSED <broker>:1883)` and exits. Once Z2M has connected, a broker outage is handled fine, because mqtt.js reconnects indefinitely. The only fatal case is the broker being late at startup, which is the normal case after a power cut or host reboot whenever the broker runs on another machine or in another container.

#4408 asked for this in 2020 and was closed because a Docker restart policy covered it. On Home Assistant OS that answer doesn't hold. The add-on's restart policy is the Supervisor watchdog, which allows 10 restarts per 30 minutes and then stops for good (home-assistant/supervisor#7190). Z2M fails in about 4 s, so all 10 are spent in under a minute.

This happened to me on 2026-11-13 (Z2M 2.14.3-1, HAOS 18.4, Supervisor 2026.11.0, Mosquitto 2.1.2 in Docker on a second host). Both hosts rebooted together. Z2M made its 10 attempts between 19:13:54 and 19:14:43, and the broker opened port 1883 at 19:17:29. Zigbee stayed down for 2 hours until I started the add-on by hand. The coordinator was healthy throughout and joined fine on every attempt; only MQTT failed.

A second effect made it much harder to spot. The `bridge/state` `online` message is retained, and `offline` is only the last will, which a client that never connects doesn't have. The broker replayed the `online` from before the reboot, so Home Assistant's bridge connection sensor read **connected** for the whole outage, while every device was `unknown`.

### Describe the solution you'd like

Retry the initial MQTT connection on connection-level errors (`ECONNREFUSED`, `EHOSTUNREACH`, `ETIMEDOUT`, `ENOTFOUND`) at a capped interval, the same way mqtt.js already retries after the first connect, logging each attempt. Authentication and TLS errors could still exit as they do now. In `lib/mqtt.ts`, `connect()` currently makes one `connectAsync()` call, so a failure goes straight to the exit path.

If indefinite retry isn't wanted, a configurable startup grace period (for example `mqtt.initial_connect_timeout`) would also solve it.

For the stale state: publishing `bridge/state` with an MQTT 5 message expiry, or documenting that consumers shouldn't trust a retained `online` on its own, would stop it masking outages.

### Describe alternatives you've considered

- A Home Assistant automation that starts the add-on when `binary_sensor.zigbee2mqtt_running` (the Supervisor sensor, disabled by default) has been off for 2 minutes, checking every 2 minutes. This is what I run now. It works, but it's HA-specific and every user has to discover it after an outage. A version that trusted the bridge sensor didn't fire, because of the stale retained state above.
- `Z2M_WATCHDOG`: from the docs and `index.js`, it only applies after a successful first start, so it doesn't cover this.
- Starting the broker first: there's no way to declare a dependency on an external broker from an HA add-on, and ordering across hosts isn't something I can guarantee.

### Additional context

Logs and timeline: https://github.com/danielrosehill/Bug-Reports-And-Fixes/tree/main/zigbee2mqtt/2026-11-13-broker-not-ready-at-boot-stale-online-state

Related: #4408, #33622 (the same "keep retrying" request for network coordinators).
