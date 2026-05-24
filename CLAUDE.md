# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Node-RED flow configuration for home automation, primarily integrating with Home Assistant. Companion to [home-assistant-config](https://github.com/billimek/home-assistant-config). There is no build system, test suite, or linting — this repo is pure configuration.

## Repository layout

- `flows.json` — single source of truth for all flows, nodes, and subflows (~316KB flat JSON array)
- `flows_cred.json` — AES-encrypted credentials blob, decrypted by Node-RED at runtime via a `credentialSecret`; never store or commit secrets in cleartext
- `package.json` — minimal; just tells Node-RED the filenames for the two files above

## How flows are authored

Flows are edited in the Node-RED editor UI and exported/committed as snapshots — all recent commits are bulk "Update flow files" exports, not hand-written changes.

`flows.json` is a flat array of node objects. The structural invariants that must be preserved when editing by hand:

- Each node has a unique `id`
- Nodes belong to a tab via their `z` field (matches the tab's `id`)
- Connectivity is defined by `wires`: an array-of-arrays of target node `id`s
- Subflow definitions have `type: "subflow"`; subflow instances have `type: "subflow:<definition_id>"`

A malformed edit (broken `id` reference, invalid JSON) silently breaks wiring at runtime. Always validate after any manual edit:

```
jq . flows.json > /dev/null
```

## Architecture

### Flow tabs (categories)

| Tab | Purpose |
|-----|---------|
| Home Assistant | General HA automations that don't fit a specific category |
| Alarm | House alarm system control and notifications |
| Presence | Arrival/departure detection and reactions |
| Garage | Garage door control and observability |
| zwave | Z-Wave device automations |

Frigate/camera automations (including the `frigate camera` subflow) now live within the remaining tabs rather than a dedicated Cameras tab.

### Subflows (reusable logic)

`open?`, `closed?`, `garage doors closed?`, `turn off lights`, `home?`, `frigate camera`

### Integration node types

- **Home Assistant**: `api-current-state`, `api-call-service`, `trigger-state`, `server-state-changed` — these reference the single `server` config node named "Home Assistant"
- **MQTT**: `mqtt in` / `mqtt out` against broker `10.0.6.50:1883`
- **Notifications**: HA `notify` domain via `api-call-service` (a `pushover-keys` credentials node still exists but is no longer connected to any active nodes)
- **Scheduling**: `bigtimer`, `time-range-switch`, `weekday`, `light-scheduler`
- **Calendar**: `ical-upcoming` (uses `ical-config` node)
- **Logic**: 65 `function` nodes carrying custom JavaScript; `switch`, `trigger`, `delay`, `rbe`

## Inspecting flows with jq

```bash
# List tabs
jq -r '.[]|select(.type=="tab")|"\(.id)\t\(.label)"' flows.json

# List subflows
jq -r '.[]|select(.type=="subflow")|"\(.id)\t\(.name)"' flows.json

# All nodes on a specific tab (replace <tabid>)
jq '[.[]|select(.z=="<tabid>")]' flows.json

# Find a node by id
jq '.[]|select(.id=="<nodeid>")' flows.json

# Count node types
jq -r '[.[].type]|group_by(.)|map({type:.[0],count:length})|sort_by(-.count)[]|"\(.count)\t\(.type)"' flows.json

# Extract JS from all function nodes
jq -r '.[]|select(.type=="function")|"// \(.name)\n\(.func)"' flows.json
```
