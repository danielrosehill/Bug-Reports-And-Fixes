<!-- Destination: follow-up comment on home-assistant/supervisor#7190 -->
A second trigger for the same behaviour, which needs no hardware fault at all: an ordinary reboot where an add-on starts before a service it depends on on another host.

On 2026-11-13 (Supervisor 2026.11.0, HAOS 18.4), the HA VM and the host running our MQTT broker rebooted together. Zigbee2MQTT 2.14.3-1 started first, got `ECONNREFUSED` from the broker, and exited. The watchdog made its 10 restarts between 19:13:54 and 19:14:43, and the broker came up at 19:17:29, 2¾ minutes after the final exit. The app then sat in `error` for 2 hours with nothing in Repairs, until it was started by hand.

So the throttle doesn't only bite on long outages. Any dependency that takes more than about a minute to come up after boot is enough, and there's no way for an add-on to declare a dependency on an external service to avoid the race.

The re-check after the throttle window (or any backoff that keeps retrying) would have recovered this within 30 minutes. A repair issue would at least have said that Zigbee was down.

Logs and timeline: https://github.com/danielrosehill/Bug-Reports-And-Fixes/tree/main/zigbee2mqtt/2026-11-13-broker-not-ready-at-boot-stale-online-state
