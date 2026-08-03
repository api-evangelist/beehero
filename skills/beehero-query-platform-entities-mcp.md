---
name: Query BeeHero platform entities over MCP
description: Use the official beehero-mcp server to discover entity types, resolve the relationship path between them, build a BeeHero filter and retrieve groups, farms, orchards, yards, gateways, sensors, inspections and experiments.
api: mcp/beehero-mcp.yml
base_url: null
operations:
  - getAvailableEntities
  - getRelations
  - getBeeheroFilter
  - getEntities
  - getEntity
  - getBeekeeperHivesAndFramesByGroupId
  - getInspectionsByGroupId
  - getExperimentsByName
  - getExperimentPointsByExperimentId
  - getExperimentSerializationsByExperimentPointId
  - getExperimentSerializationByFileName
generated: '2026-08-02'
method: generated
source: mcp/beehero-mcp.yml
---

# Query BeeHero platform entities over MCP

The official `beehero-mcp` package (npm, stdio transport) exposes BeeHero's platform data — a
different surface from the public `/external` REST API, which only carries device telemetry.

## Setup

Install `beehero-mcp` and configure three values: `API_USER_EMAIL`, `API_PASSWORD` and
`API_SERVER_URL`. The server logs in itself and manages the bearer token; you never handle it.

## The four-step query pattern

The server is designed to be driven in this order — following it avoids guessing entity names or
filter syntax.

1. **`getAvailableEntities`** — no arguments. Returns every queryable entity type, grouped by
   category (core, hardware, monitoring, experiments, seasonal). Call this before assuming a name.
2. **`getRelations`** with `fromEntity` and `toEntity` — returns the relationship path between two
   entity types plus the filter fields to use at each hop. Always call this before navigating
   between entities rather than inventing a foreign key.
3. **`getBeeheroFilter`** with `attributes` (and optionally an existing `filter` to extend) —
   returns a filter string of the form `[('att', 'op', 'value'), ...]`. Supported operators:
   `eq, not, in, notin, gt, gte, lt, lte, like, notlike, between, notbetween, or`. Pass the returned
   string through unchanged; do not reformat it.
4. **`getEntities`** with `entity`, the `filter` from step 3, and an optional `limit`
   (default 100) — returns the records.

Example: to get the sensors for a group, call `getRelations('groups', 'sensors')` (which tells you
to filter sensors by `group_id`), then `getBeeheroFilter({group_id: '123'})`, then
`getEntities('sensors', filter)`.

## Direct lookups

- **`getEntity`** with `entityType` and `entityId` — single record. Supported types: `user`,
  `experiment`, `hardware-order`, `inspection`, `orchard`.
- **`getBeekeeperHivesAndFramesByGroupId`** with `group_id` — hive and frame counts for a group.
- **`getInspectionsByGroupId`** with `group_id` — inspections for a group.

## Experiments

Walk the chain: `getExperimentsByName(name)` → `getExperimentPointsByExperimentId(experiment_id)` →
`getExperimentSerializationsByExperimentPointId(experiment_point_id)` →
`getExperimentSerializationByFileName(file_name)`.

## Rules

- Every tool is read-only. There is no write, delete or configuration tool in this server — hardware
  changes go through the REST API (see `skills/beehero-configure-gateway.md`).
- `getEntities` defaults to `limit: 100`. Raise it deliberately; there is no cursor, so a large
  limit is the only way to widen a result set.
- The entity hierarchy: groups contain users, farms, yards, gateways and sensors; farms contain
  orchards; orchards contain yards and orchard-markers. Filter by `group_id`, `farm_id`,
  `orchard_id`, `gateway_id` accordingly. Full graph in `data-model/beehero-data-model.yml`.
- These tools do **not** map to any published OpenAPI operation — see
  `mcp/beehero-tool-crosswalk.yml`.
