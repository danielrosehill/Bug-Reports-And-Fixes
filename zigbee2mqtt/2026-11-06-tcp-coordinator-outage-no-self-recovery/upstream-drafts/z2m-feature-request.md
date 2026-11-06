<!-- Destination: Koenkk/zigbee2mqtt, Feature request template -->
**Title:** [Feature request]: Keep retrying a tcp:// coordinator until it returns, including at startup

### Is your feature request related to a problem? Please describe

When a network coordinator is unreachable for more than about a minute, Z2M does not recover once it comes back. Unplugging a PoE coordinator for a couple of minutes is enough, and I can reproduce this on demand.

On Home Assistant OS, Z2M exits and each failed start takes about 3.5 s, so the Supervisor watchdog's limit of 10 restarts per 30 minutes is used up in 42 s. Nothing retries after that. The coordinator is back and its port is open, but Zigbee stays down until someone starts the add-on by hand, with no notification. Today that meant 1 h 47 min of downtime with a healthy coordinator.

I know this has come up before (#21921, #29654) and that `Z2M_WATCHDOG` exists. From the docs and `index.js` (I haven't run it myself), it leaves two gaps for network coordinators:

- The delay list is finite, so a long enough outage still ends in an exit.
- It only applies after a successful first start, so a coordinator that is absent when Z2M starts isn't covered, for example after a power cut when the host boots first.

### Describe the solution you'd like

For `tcp://` adapters, treat connection-level failures (`ECONNREFUSED`, `EHOSTUNREACH`, `ETIMEDOUT`, `ECONNRESET`) as retryable: stay running and keep trying at a capped interval until the coordinator returns.

A smaller change would also solve it:

1. Let `Z2M_WATCHDOG` repeat its last delay indefinitely.
2. Let the watchdog cover connection failures on the first start.

On the concern in #29654 that retrying forever could have unintended consequences: this would be limited to opening a TCP connection to a host that isn't answering, which doesn't touch any hardware. Settings errors and any other failure would still stop Z2M as they do now.

### Describe alternatives you've considered

- A very long `Z2M_WATCHDOG` list. This works for a time, but is still finite and doesn't cover first start.
- A Home Assistant automation that starts the add-on when the bridge is offline and the coordinator is reachable. That's what I run now, but it's specific to HA and every network coordinator user has to discover it after an outage.

### Additional context

Z2M 2.14.3-1, zigbee-herdsman 11.2.0, HAOS 18.3, Ember adapter, SLZB-MR3U over PoE.

Full write-up with logs and a timeline: https://github.com/danielrosehill/Bug-Reports-And-Fixes/tree/main/zigbee2mqtt/2026-11-06-tcp-coordinator-outage-no-self-recovery

Related: home-assistant/supervisor#7190
