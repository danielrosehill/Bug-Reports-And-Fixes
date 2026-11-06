<!-- Destination: comment on home-assistant/supervisor#7190 -->
I can reproduce this on demand, on Supervisor 2026.10.5 with Zigbee2MQTT 2.14.3-1 and a PoE coordinator, by unplugging the coordinator's cable for about 3.5 minutes.

The add-on exited at 15:12:58, the watchdog made exactly 10 restarts between 15:12:59 and 15:13:38, and the final exit at 15:13:41 was followed by nothing. The coordinator came back at about 15:16, and the app stayed in `error` with no notification or repair issue. An unplanned outage earlier the same day went the same way and left Zigbee down for 3 hours.

That matches `WATCHDOG_THROTTLE_MAX_CALLS = 10` per `WATCHDOG_THROTTLE_PERIOD = 30 min`. Because the watchdog runs on the container exit event, it looks like the throttled exit is dropped and nothing re-checks the app when the window expires. An app that fails in 3 to 4 seconds uses the whole budget in under a minute, so the 30-minute window gives far less time than it suggests.

Two things would solve this, and either would help:

1. Re-check an app left in `error` once the throttle window expires, rather than giving up for good.
2. Raise a repair issue or notification when the watchdog stops, as this report asks.

Logs and timeline: https://github.com/danielrosehill/Bug-Reports-And-Fixes/tree/main/zigbee2mqtt/2026-11-06-tcp-coordinator-outage-no-self-recovery
