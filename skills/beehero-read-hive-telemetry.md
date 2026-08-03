---
name: Read BeeHero hive telemetry
description: Authenticate against the BeeHero API and pull temperature, humidity and bee in/out counts for a set of in-hive sensors over a date range, including resolving which sensors are currently connected to a gateway.
api: openapi/beehero-openapi-original.yml
base_url: https://backend.beehero.io/external
operations:
  - login
  - get_sensors_connected_by_gateway_mac
  - get_sensors_sample_by_mac_address
generated: '2026-08-02'
method: generated
source: openapi/beehero-openapi-original.yml
---

# Read BeeHero hive telemetry

Use this skill to retrieve in-hive sensor readings from BeeHero. Every call in this flow is a `POST`
with a JSON body against `https://backend.beehero.io/external`.

## 1. Authenticate

Call `login` (`POST /login`) with the account email and password:

```json
{ "email": "user@mail.com", "password": "..." }
```

The response body contains `access_token` (it is also set as an `access_token_cookie` cookie). Send
it on every subsequent call as `Authorization: Bearer <access_token>`. Token lifetime is not
documented — treat a `403 Invalid Token` as "re-login and retry once".

A `404 Login credentials do not match` means the email/password pair was rejected; do not retry with
the same credentials.

## 2. Find the sensors you care about (optional)

If you only know the gateway, call `get_sensors_connected_by_gateway_mac`
(`POST /sensors/samples_connected`) with the gateway MAC addresses. It returns
`{ gateway_mac, sensors_conected[] }` — the sensor MAC addresses currently reporting through each
gateway. Note the spelling of `sensors_conected` in the response; it is the published field name.

## 3. Pull the samples

Call `get_sensors_sample_by_mac_address` (`POST /sensors/samples`):

```json
{
  "mac": ["d0:cf:5e:f7:33:1d", "d0:cf:5e:f7:33:1e"],
  "from": "2020-10-18",
  "to": "2020-10-18"
}
```

- `mac` is an array, so batch many hives into one call rather than looping one MAC per request.
- `from` and `to` are `YYYY-MM-DD` dates and are the only range control — there is no pagination,
  no limit and no cursor, so keep windows narrow on large fleets.

Each returned sample carries `sensor_mac_address`, `gateway_mac_address`, `temperature`,
`pcb_temperature_one`, `humidity`, `in_count` (bees entering), `out_count` (bees exiting),
`external_weight`, `bleRssi`, `voltage`, `firmware_version` and `timestamp`. A `message` field is
populated when a requested MAC could not be found — check it before assuming a silent gap.

## Rules

- Reads are modelled as `POST` but have no side effects; they are safe to retry.
- `401 Login required` — no token was sent. `403 Invalid Token` — token expired or invalid, re-login.
- `404 Validation exception` is BeeHero's response to a malformed body, not a missing resource:
  check the MAC format (colon-separated) and the `YYYY-MM-DD` dates before retrying.
- No rate limits are published. Be conservative: batch MACs, keep date ranges tight, and back off on
  repeated failures.
- See `conventions/beehero-conventions.yml` and `errors/beehero-problem-types.yml`.
