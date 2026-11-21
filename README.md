# Thermowatt Smart Boiler Bridge for Home Assistant

This fork includes progressive rate-limit recovery in version 1.3.7, described
below. See the [original repair guide](REPAIR.md) for backup, validation and
rollback guidance; its older release instructions describe historical behaviour.

This add-on allows you to integrate Thermowatt-based smart water heaters into Home Assistant using MQTT. It bridges the gap between the Thermowatt cloud and your local Home Assistant instance. Readings and heater commands require the vendor cloud; this is not local heater control.

[![Open your Home Assistant instance and show the add add-on repository dialog with a specific repository URL pre-filled.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fskooterpunk%2Fha-thermowatt-heater)

## Features

- **Polled Monitoring**: Track cloud-reported water temperature (`T_Avg`).
- **Cloud Commands**: Set target temperatures and change operating mode when cloud access and the recovery checks allow it.
- **MQTT Discovery**: Automatically creates a "Water Heater" device in Home Assistant.
- **Diagnostic Sensors**: Monitor errors and system status.

## Installation

1. Prerequisite: Install and start Mosquitto MQTT broker within Home Assistant.
2. Click the **Add Repository** button above, or manually add `https://github.com/skooterpunk/ha-thermowatt-heater` to your Home Assistant Add-on Store.
3. Install the **Thermowatt Smart Boiler** add-on.
4. Configure your Thermowatt account credentials in the **Configuration** tab.
5. Start the add-on.

When switching an existing installation, back up the app and its data first and
confirm that Home Assistant offers this fork's version 1.3.7. Stop the old bridge
before starting the fork; do not run multiple bridges against the same account.

## Configuration

```yaml
email: "your-email@example.com"
password: "your-password"
poll_interval: 300
```

`poll_interval` is the regular status polling interval in seconds for each heater:
default **300** (5 minutes), allowed **60–3600**. Set it in the add-on's
Configuration tab. Recovery preserves this setting; it does not switch to faster
polling. Ordinary failed status reads are retried on the regular schedule unless
a rate-limit response starts a cooldown.

Scheduled status traffic is approximately `device_count * 86400 / poll_interval`
requests per day, plus authentication, discovery and command-related requests.
Accepted commands schedule a real status read after about 20 seconds. A longer
interval reduces regular traffic but delays readings and some recovery attempts;
neither the default nor any other interval is a known quota-safe setting.

## Rate-limit recovery (v1.3.7)

The bridge recognizes daily-limit errors in the cloud response body, including
HTTP 200 responses, as well as HTTP 429 and other recognized limit-exceeded
errors. A quota error takes precedence over an HTTP 401: the bridge pauses rather
than refreshing authentication and replaying the request. A limit reported during
token refresh also prevents replay of the original request.

For successive daily-limit responses without an intervening valid heater status,
the bridge chooses these waits, measured from each new limit response:

| Daily-limit strike | Bridge cooldown |
| --- | --- |
| 1 | 1 hour |
| 2 | 2 hours |
| 3 | 4 hours |
| 4 | 8 hours |
| 5 | 16 hours |
| 6 and later | 24 hours |

Other recognized rate-limit responses use an increasing delay based on
`poll_interval` (starting at twice that interval, capped at 1 hour). This retry
counter is in memory; the daily-limit strike count is persisted. A valid longer
`Retry-After`, expressed as seconds or an HTTP date, can extend either policy,
even beyond 24 hours. Invalid values are ignored. A shorter new pause never
reduces an existing later deadline, including one saved by an older version.
Upgrading therefore does not replace an existing 24-hour pause with a 1-hour wait.

The deadline (`blocked_until`), reason (`cooldown_reason`) and daily-limit strike
count (`daily_limit_strikes`) are saved in `/data/thermowatt_config.json` in the
add-on's persistent data. Restarting preserves them and does not bypass the wait.
Cooldown timing uses the system clock. Keep the app data intact; deleting it
discards this history and does not reset any vendor-side restriction.

While the deadline is active, the bridge suppresses all its cloud requests:
login, token refresh, discovery, status polls and heater commands. A limit also
stops the remaining device requests in the current polling batch and marks all
known heaters unavailable. MQTT diagnostics continue without cloud traffic.
Incoming commands are rejected, and the cooldown loop discards queued commands
and pending command follow-up polls so they are not executed hours later.
Retained MQTT commands are always ignored.

Expiry permits cloud reads again; it does not unlock heater commands by itself.
After startup or any bridge pause, **each heater must receive its own newly
fetched, valid `/status` response** before its commands resume. Authentication,
discovery, HTTP 200 alone and malformed or unsuccessful status payloads do not
satisfy this check. Temperature and mode/status fields must pass validation.
All heater commands are gated, including Off, mode changes and setpoint changes.
A further pause requires a new valid status again.

The first valid heater status resets the shared daily-limit strike count to zero
and clears its saved deadline/reason. That heater can accept commands; other
heaters still awaiting valid status remain blocked. A restart requires fresh
status for every discovered heater again. This is a validated cloud read, not
proof that the physical heater is heating or that the backend data is current.

### What Home Assistant users should expect

These waits are **bridge-imposed recovery timing**, not published Thermann backend
limits or reset times. The request quota, its scope and its reset schedule are
unknown. The bridge retries automatically when its pause and polling schedule
allow; the Next Cloud Retry timestamp is not a promise of an immediate request
or successful recovery. Further limits can extend the wait. Persistent cloud
failures or rejected status data can leave heaters unavailable and commands
blocked indefinitely.

The **Thermowatt Bridge** MQTT device exposes Bridge State, Bridge Heartbeat,
Cooldown Reason and Next Cloud Retry. Heartbeat normally updates every minute
while the bridge loop runs, including during cooldown; logs report the remaining
pause and UTC deadline at connection/pause and about every five minutes.
Home Assistant displays the retry timestamp in its local timezone. Next Cloud
Retry is unavailable outside an active cooldown. An online bridge or heartbeat
does not establish heater availability or successful recovery.

Failed responses do not replace retained successful readings with zero or fake
state, but those retained readings can be stale. Automations should explicitly
handle unavailable entities and check `last_successful_update`, rather than
treating old readings or numeric fallbacks as current temperatures. Commands
rejected during recovery are not replayed; send a new deliberate command after
that heater has recovered. Cloud acceptance still requires a later status read
to show the resulting state and does not prove physical heating.

Avoid repeated restarts, tight command-retry automations and multiple bridges.
Other clients, including the vendor app, may also generate backend traffic; this
bridge cannot suppress their requests. Increasing `poll_interval` can reduce
this bridge's traffic, but cannot guarantee recovery or uninterrupted control.

## Dashboard

Once the add-on is running, a new entity will appear under your MQTT integration. We recommend using the Thermostat Card for the best experience.

## Troubleshooting

Since 1.3.4, Python bridge logs use UTC ISO 8601 timestamps with milliseconds
and a severity level. Match these to Home Assistant history by converting UTC
to the timezone used by your Home Assistant instance.

Each HTTP attempt logs the method, endpoint, HTTP status and elapsed seconds.
Device requests include the boiler name and serial. HTTP 200 means a response
arrived; a following `[STATUS] ... failed` still means its contents were rejected.
Request/response bodies, credentials, tokens and raw request exceptions are not logged.

The following example uses synthetic timestamps and device identifiers:

```code
2000-01-01T12:00:00.936Z INFO [HTTP] Example Boiler (example-serial) GET /status; status=200 duration=0.825s reason=response received
2000-01-01T12:00:00.937Z WARNING [AVAILABILITY] Example Boiler (example-serial): data unavailable
2000-01-01T12:00:00.937Z WARNING [STATUS] Example Boiler (example-serial): failed: unsuccessful or malformed cloud response / invalid value
2000-01-01T12:05:03.419Z INFO [HTTP] Example Boiler (example-serial) GET /status; status=200 duration=0.740s reason=response received
2000-01-01T12:05:03.420Z INFO [RECOVERY] Example Boiler (example-serial): fresh status received; data unavailable for 302.5s
```

Availability/recovery messages appear once per transition, including startup and
MQTT reconnection. The duration starts when this process publishes the first
offline state, uses a monotonic clock, and does not survive a bridge restart.
It measures unavailable bridge data, not a confirmed physical heater outage.
Container service-manager output may keep its own format.

The bridge discovers the account's devices dynamically; no household layout or
device count is assumed. Documentation examples and test fixtures are synthetic.
Runtime logs stay in your installation and include configured device names and
serials for troubleshooting. Redact these identifiers and other private data
before sharing logs publicly.

## Device errors and temporary heating pauses

Version 1.3.6 adds **Device Error Status** and **Device Error Message** diagnostic
sensors for each discovered heater. The same fields are also attributes on the
water-heater entity. They use the existing status poll and normal device availability;
they do not send commands or request extra cloud updates.

`high_ambient_temperature_pause` means the observed protection flag indicates that
heating is temporarily paused while the electronics cool. The heater should resume
automatically when conditions normalize. Do not repeatedly restart it or force heating.
The manufacturer's [owner's guide](https://www.thermann.com.au/thermann-cms/files/Thermann-Smart_Electric-Owners_Guide.pdf)
describes this protective pause under its service guidance.

The protocol mapping is **observed, not vendor documented**: non-volatile flag 0x02
was correlated with the vendor app's high-ambient message on thermostat firmware
`07.00.09.06`. It is not a mapping to the app's E2 error code and does not use a guessed
ambient-temperature cutoff. Other firmware and error bits remain unmapped.

Diagnostic values:

- `ok`: both device error masks are valid and zero; this does not mean heating is on.
- `high_ambient_temperature_pause`: only the observed pause flag is set.
- `unmapped_device_error`: other error bits or unsupported-firmware flags are present;
  check the vendor app for the actual explanation.
- `unknown_error_status`: either mask is missing or malformed; do not assume healthy.

`NonVolatileErrors` and `VolatileErrors` are preserved unchanged. Additional attributes
include `high_ambient_temperature_pause` (true/false, or null when unverified),
`unmapped_non_volatile_errors`, `unmapped_volatile_errors` and `device_error_mapping`.
When known and unknown bits coexist, the status remains `unmapped_device_error` so
the known pause cannot hide an additional problem. Device-error pause/recovery logs
are separate from bridge connectivity, API rate-limit cooldown and heating status.
Automations should honor heater protection and avoid retrying settings during a pause.

## Known to work on:

- **Home Assistant:**
  - _Installation method:_ Home Assistant OS
  - _Core:_ 2025.12.5
  - _Supervisor:_ 2026.01.1
  - _Operating System:_ 16.3
  - _Frontend:_ 20251203.3
- **Mosquitto MQTT Version:** 6.5.2
- **MyThermowatt App Version:** 3.14

Tip: Help others by adding your version here, if it works.

---

_Disclaimer: This project is not affiliated with or endorsed by Thermowatt or Ariston._

---

### Support my work

If this add-on saved you some frustration or made your home a bit smarter or helped someone avoid a cold shower, feel free to [buy me a beer on Ko-fi!](https://ko-fi.com/thermohacker)

[![support](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/thermohacker)


