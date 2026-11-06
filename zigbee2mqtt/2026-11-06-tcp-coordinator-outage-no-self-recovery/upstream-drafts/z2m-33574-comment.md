<!-- Destination: comment on Koenkk/zigbee2mqtt#33574. CONFIRM FIRST that nothing was touched on 2026-11-04. -->
Another data point, in case it helps: I see the same signature on 2.14.3-1 with a different coordinator, an SLZB-MR3U (EFR32MG24, EmberZNet 7.4.4) over PoE with the Ember adapter.

```
[2026-11-04 10:45:19] error: zh:adapter:transport: TCP socket error: Error: read ECONNRESET
[2026-11-04 10:45:19] error: z2m: Adapter disconnected, stopping
[2026-11-04 11:55:42] error: zh:adapter:transport: TCP socket error: Error: read ECONNRESET
[2026-11-04 11:55:42] error: z2m: Adapter disconnected, stopping
```

It's far less frequent than yours: two in the five days since updating on 2026-11-02. My logs go back to 2026-10-30 and show none in the three days before the update, but that's a short history, so I wouldn't read much into it.
