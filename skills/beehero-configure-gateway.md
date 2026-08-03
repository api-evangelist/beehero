---
name: Inspect and configure a BeeHero gateway
description: Read gateway telemetry and configuration status, then change gateway settings — skip-remote, movement interrupt and RSSI configuration — on BeeHero field gateways.
api: openapi/beehero-openapi-original.yml
base_url: https://backend.beehero.io/external
operations:
  - login
  - get_gateway_mac_address
  - get_gateways_sample_by_mac_address
  - get_gateway_config_status
  - add_skip_remote_to_gateway_config
  - remove_skip_remote_from_gateway_config
  - movement_interrupt_gateway_config
  - rssi_configuration
generated: '2026-08-02'
method: generated
source: openapi/beehero-openapi-original.yml
---

# Inspect and configure a BeeHero gateway

Gateways are the field devices that relay in-hive sensor data. This skill covers reading their state
and changing their configuration. **The configuration operations change hardware behaviour in the
field — always read state first and confirm the intended change with a human before writing.**

## 1. Authenticate

`login` (`POST /login`) → keep `access_token` → send `Authorization: Bearer <access_token>`.

## 2. Read state (safe)

- `get_gateway_mac_address` (`POST /gateways/samples/get_mac_address`) — resolve the gateway MAC
  address you will operate on.
- `get_gateways_sample_by_mac_address` (`POST /gateways/samples`) — gateway telemetry for a list of
  gateway MACs over a `from`/`to` date range: `humidity`, `modem_rssi`, `latitude`, `longitude`,
  `pcb_temperature_two`, `battery_voltage`, `firmware_version`, `timestamp`.
- `get_gateway_config_status` (`POST /gateways/gateway_config_status`) — the current skip-remote
  configuration status for the gateway.

Check `battery_voltage` and `modem_rssi` before pushing configuration: a gateway on a weak modem
signal or low battery may not pick the change up.

## 3. Write configuration (consequential)

All four are `PUT` operations:

- `add_skip_remote_to_gateway_config` (`PUT /gateways/add_skip_remote`)
- `remove_skip_remote_from_gateway_config` (`PUT /gateways/remove_skip_remote`)
- `movement_interrupt_gateway_config` (`PUT /gateways/movement_interrupt`)
- `rssi_configuration` (`PUT /gateways/rssi_configuration`)

After any write, re-run `get_gateway_config_status` to confirm the change landed.

## Rules

- No idempotency key exists. The operations are `PUT` and so are idempotent by HTTP method, but
  BeeHero publishes no de-duplication or retry contract — never fire-and-forget a retry loop against
  these; confirm with a read instead.
- One gateway serves many hives. A misconfiguration interrupts data collection for a whole yard
  during bloom, when the pollination window cannot be re-run. Escalate to a human for confirmation
  before every write.
- `401 Login required` — no token. `403 Invalid Token` — re-login. `404 Validation exception` — the
  body failed validation, not a missing gateway.
- See `agentic-access/beehero-agentic-access.yml` for the recommended execution contract per
  operation, and `conventions/beehero-conventions.yml`.
