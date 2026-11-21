# Thermowatt Smart Boiler for Home Assistant

This add-on bridges the Thermowatt cloud to Home Assistant through MQTT.
Temperature readings and heater commands depend on the vendor cloud; it does
not provide local heater control. Version 1.3.7 adds progressive rate-limit
recovery.

## Installation and configuration

Install and start the Mosquitto MQTT broker, add
`https://github.com/skooterpunk/ha-thermowatt-heater` to the add-on store, then
install Thermowatt Smart Boiler and configure it before starting:

```yaml
email: "your-email@example.com"
password: "your-password"
poll_interval: 300
```

`poll_interval` is the regular status interval per heater in seconds: default
300, allowed 60–3600. Recovery keeps the configured interval. A longer interval
reduces regular traffic and delays updates; authentication, discovery and command
follow-up requests add traffic. Accepted commands schedule a status read after
about 20 seconds. No polling interval is known to be quota-safe.

Back up an existing app and its data before switching, verify the fork's version,
and stop the old bridge before starting this one. Run only one bridge per account.
See the [repository README](../README.md) for installation and troubleshooting.

## Rate-limit recovery in 1.3.7

- Successive daily-limit responses trigger waits of **1, 2, 4, 8, 16, then
  24 hours**, with further waits capped at 24 hours. Each wait starts at the new
  limit response. Other recognized rate limits, including HTTP 429, use increasing
  waits based on the polling interval, capped at 1 hour. A valid longer
  `Retry-After` can extend either policy beyond those caps; shorter waits never
  shorten an existing later deadline.
- The cooldown deadline, reason and daily-limit strike count are saved in
  `/data/thermowatt_config.json` and survive restarts and upgrades with app data
  preserved. An older saved 24-hour deadline remains in force. Restarting does
  not clear a cooldown; deleting app data does not reset vendor limits.
- During an active cooldown the bridge sends **no cloud requests**, including
  login, token refresh, discovery, status reads or commands. Quota errors are
  recognized even in HTTP 200 or 401 responses, before authentication retries.
  All known heaters become unavailable, the remaining polling batch stops,
  incoming commands are rejected, and queued commands and pending follow-up
  polls are discarded. Retained MQTT commands are ignored.
- After startup or any bridge pause, **each heater needs its own newly fetched,
  valid status before any commands resume**, including Off. Expiry, successful
  login/discovery and HTTP 200 alone are insufficient. Failed or malformed status
  responses leave that heater blocked; regular recovery polls keep the configured
  interval. The first valid heater status resets the shared daily-limit strikes
  and clears their saved deadline/reason, but does not unlock other heaters.
  Another pause or restart requires fresh status again.

These waits are the bridge's recovery policy. **Thermann backend quotas, their
scope and reset times are unknown.** Next Cloud Retry is a bridge deadline, not
a guaranteed immediate request or recovery time. Further limits can extend it;
persistent cloud or status failures can leave commands blocked indefinitely.
The bridge cannot suppress traffic from other clients such as the vendor app.

## Monitoring and automations

The Thermowatt Bridge MQTT device exposes Bridge State, Bridge Heartbeat,
Cooldown Reason and Next Cloud Retry. Diagnostics continue during cooldown
without cloud requests; heartbeat normally updates every minute and cooldown
logs about every five minutes. Home Assistant shows the retry timestamp in its
local timezone; it is unavailable outside an active cooldown. A live bridge
does not mean the heaters are reachable.

Failed responses do not overwrite retained valid readings, which may become
stale. Automations should handle unavailable entities and check
`last_successful_update`. Avoid tight command retries and repeated restarts.
Rejected commands are not replayed: send a new command once that heater has
recovered. A valid cloud reading or an accepted command does not prove physical
heating, current backend data or uninterrupted control. See the
[detailed recovery guidance](../README.md#rate-limit-recovery-v137) for limitations.
