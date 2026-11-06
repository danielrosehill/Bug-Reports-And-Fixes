# Zigbee2MQTT does not recover after a network coordinator outage

**Date:** 2026-11-06
**Status:** Confirmed (reproduced on demand)
**Severity:** High. A brief loss of a network coordinator leaves the whole Zigbee network down until someone starts the add-on by hand.
**Upstream:** filed as [Koenkk/zigbee2mqtt#33622](https://github.com/Koenkk/zigbee2mqtt/issues/33622), with a [comment on home-assistant/supervisor#7190](https://github.com/home-assistant/supervisor/issues/7190#issuecomment-6685803456). Posted text is in [`upstream-drafts/`](upstream-drafts/).

## Summary

If a network-attached (`tcp://`) Zigbee coordinator disappears for more than about a minute, Zigbee2MQTT on Home Assistant OS stays stopped after the coordinator comes back. Nothing retries. The coordinator is healthy, the port is open, and the Zigbee network stays down without any notification.

Unplugging one cable is enough. With a PoE coordinator, that cuts both its network and its power.

## Environment

| Component | Version |
|---|---|
| Zigbee2MQTT add-on | 2.14.3-1 (zigbee-herdsman 11.2.0) |
| Home Assistant Core | 2026.10.2 |
| Home Assistant OS | 18.3 |
| Supervisor | 2026.10.5 |
| Coordinator | SMLIGHT SLZB-MR3U, PoE, core firmware v3.3.1 |
| Coordinator radio | EFR32MG24, EmberZNet 7.4.4 |
| Z2M config | `serial.adapter: ember`, `serial.port: tcp://<coordinator-ip>:6638` |
| Add-on options | Supervisor watchdog on, start on boot on, `watchdog` (`Z2M_WATCHDOG`) unset |

## Steps to reproduce

1. Run the Z2M add-on with a `tcp://` coordinator and the Supervisor watchdog enabled.
2. Unplug the coordinator's Ethernet cable and leave it out for at least 2 minutes.
3. Plug it back in and wait for the coordinator to boot.

**Expected:** Z2M reconnects by itself once the coordinator is reachable.

**Actual:** the add-on stays in state `error` until it is started by hand.

## Observed timeline (reproduction, 2026-11-06, UTC+2)

| Time | Event |
|---|---|
| 15:12:58 | `TCP socket error: Error: read ETIMEDOUT`, then `Adapter disconnected, stopping` (exit code 2) |
| 15:12:59 to 15:13:38 | 10 Supervisor watchdog restarts, each ending in `connect EHOSTUNREACH` / `HOST_FATAL_ERROR` about 3.5 s later |
| 15:13:41 | Final exit, code 1. No restart follows; add-on state `error` |
| ~15:16:13 | Coordinator boots after the cable is reconnected |
| 15:16:38 | Coordinator answering, port 6638 open, no client on the Zigbee socket, add-on still `error` |
| 15:21:28 | More than 5 minutes after the coordinator returned: add-on still `error`, no new start attempts, no client on the Zigbee socket |
| 15:21:33 | Workaround automation (off during the test) turned back on |
| 15:25:00 | Automation fires on its 5-minute check and starts the add-on |
| 15:25:04 | `Zigbee2MQTT started!`, 24 devices joined |

The same sequence happened unplanned earlier that day: Z2M stopped at 11:54:18 and the watchdog made 10 restarts by 11:55:27. The coordinator was back at about 13:07, and Z2M stayed down until it was started by hand at 14:55. Zigbee was down for 3 hours, including 1 h 47 min with a healthy coordinator.

Raw evidence, with IP addresses removed:

- [`evidence/z2m-disconnect.log`](evidence/z2m-disconnect.log): the moment the connection dropped
- [`evidence/z2m-last-failed-start.log`](evidence/z2m-last-failed-start.log): the last failed start
- [`evidence/supervisor-watchdog.log`](evidence/supervisor-watchdog.log): every exit and restart
- [`evidence/timeline.jsonl`](evidence/timeline.jsonl): coordinator, add-on and HA state, polled every 2 s or so. It has a gap from 15:12:39 to 15:14:41, when the poller stalled, and the row stamped 15:12:39 holds state read later in that gap. Use the logs for that window.

## Why it happens

Three behaviours combine:

1. **Z2M exits when the adapter connection is lost.** It does the same if the adapter is unreachable at startup. A failed start takes about 3.5 s.
2. **The Supervisor watchdog is rate limited.** `WATCHDOG_THROTTLE_MAX_CALLS = 10` per `WATCHDOG_THROTTLE_PERIOD = 30 min` (`supervisor/apps/const.py`). Fast failures use all 10 in under a minute (42 s in the reproduction).
3. **Nothing retries after that.** The watchdog acts on the container exit event. The 11th exit is throttled, no further event arrives, and the expiry of the 30-minute window does not trigger anything. The Supervisor logged nothing further at its default log level, and no notification or repair issue was raised.

### Z2M's built-in watchdog does not fully cover this

`Z2M_WATCHDOG` (the add-on's `watchdog` option) keeps the process alive and retries. Going by its [documentation](https://www.zigbee2mqtt.io/guide/installation/15_watchdog.html) and `index.js`, not by testing it here:

- Its delays are a finite list, and when it runs out the process exits. A longer list only postpones that.
- The default delays (2 s, then 1, 5, 15, 30, 60 min) mean an outage of 73 minutes is noticed at about minute 111.
- It applies only after a successful first start (`unsolicitedStop` is set after the start succeeds), so a coordinator that is absent when Z2M starts is not covered.

## Workaround

[`workaround-ha-automation.yaml`](workaround-ha-automation.yaml) is a Home Assistant package. It starts the add-on when the bridge has been offline for 2 minutes and the coordinator is reachable, on the coordinator's return and every 5 minutes. It uses `hassio.app_start`, which does nothing if the add-on is already running.

The coordinator check uses an entity from the SMLIGHT integration. With other hardware, use any sensor that shows the coordinator is reachable.

Tested in this reproduction: it recovered Z2M from the real failed state in 4 seconds, on its 5-minute check. It was switched off when the coordinator returned, so its other trigger, the coordinator coming back, has not been exercised. It also does nothing while the bridge is up, and its action is a no-op on a running add-on.

## Prior reports

| Issue | State | Notes |
|---|---|---|
| [Koenkk/zigbee2mqtt#21921](https://github.com/Koenkk/zigbee2mqtt/issues/21921) | Closed (stale) | Same request, 2024. No maintainer reply |
| [Koenkk/zigbee2mqtt#29654](https://github.com/Koenkk/zigbee2mqtt/issues/29654) | Closed, not planned | Indefinite retry. Maintainer: use a longer `Z2M_WATCHDOG` list; retrying forever can have unintended consequences |
| [Koenkk/zigbee2mqtt#26429](https://github.com/Koenkk/zigbee2mqtt/issues/26429) | Closed, completed | Startup without adapter, answered with PR #26608 |
| [zigbee2mqtt/hassio-zigbee2mqtt#760](https://github.com/zigbee2mqtt/hassio-zigbee2mqtt/issues/760) | Closed (stale) | SLZB-06, same symptom |
| [home-assistant/supervisor#7190](https://github.com/home-assistant/supervisor/issues/7190) | Open, no replies | Same failure, asks for a notification and backoff |

## Earlier disconnects

Z2M also dropped the coordinator with `read ECONNRESET` on 2026-11-04 at 10:45:19 and 11:55:42, and recovered through the watchdog each time. Cables in the same cabinet were being moved at those times, so these are not evidence of a Z2M fault. They also show the other side of this issue: a short interruption recovers by itself, and only one lasting longer than the watchdog's restarts does not.
