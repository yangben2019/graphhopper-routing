---
name: graphhopper-selfhost
description: Deploy a self-hosted OpenStreetMap routing service from a .pbf file — import the graph, prepare Contraction Hierarchies, start the HTTP API, and verify it with a real request instead of reading logs. Use when the user wants to run GraphHopper or OpenRailRouting on their own machine, mentions 自托管路径规划, OSM pbf, graph-cache, import, serve, 部署 routing service, or asks why an import failed, why the service returns nothing, or how much memory it needs.
---

# Self-hosted OSM routing

Take a user from "I have a .pbf file" to "I can curl a route", and diagnose the three places this normally breaks: the import, the CH preparation, and the first request.

## Language

Reply in the language the user wrote in. Keep commands, config keys, paths, profile names and error text exactly as they appear — translating a YAML key or a stack trace makes it unusable.

## Ask these before touching anything

1. **Which engine?** Stock GraphHopper (roads) or OpenRailRouting (railways)? They are different jars with different configs.
2. **How big is the pbf?** Memories and import time scale with it.
3. **What is `-Xmx` going to be?** If the machine has 8 GB, a national extract will not fit.
4. **Is the jar already built?** If not, they need Maven and (for the web UI) Node.

Do not start an import that cannot finish. Ask first.

## Step 1 — Get a jar

| Engine | Where | Notes |
|---|---|---|
| OpenRailRouting | `github.com/geofabrik/OpenRailRouting` releases, or build from source | Apache-2.0. Build needs `git submodule init && git submodule update && mvn clean install`. Produces `railway_routing-<version>.jar`. |
| Stock GraphHopper | `github.com/graphhopper/graphhopper` releases | `graphhopper-web-<version>.jar`. |

Pick a version consistent with the config you are going to write. OpenRailRouting 1.0.0 tracks GraphHopper `9.1-osm-reader-callbacks`.

## Step 2 — Write the config

Start from the minimal shape below and add only what the user actually needs. A config with five profiles costs five CH preparations; if they only need one vehicle, delete the rest.

```yaml
graphhopper:
  datareader.file: china-latest.osm.pbf
  graph.location: graph-cache

  profiles:
    - name: car
      custom_model_files: [car.json]

  profiles_ch:
    - profile: car
  profiles_lm: []

  graph.encoded_values: car_access, car_average_speed
  graph.dataaccess.default_type: RAM_STORE
  import.osm.ignored_highways: footway,cycleway,path,pedestrian,steps
  prepare.min_network_size: 200

server:
  application_connectors:
    - type: http
      port: 8989
      bind_host: localhost
  admin_connectors:
    - type: http
      port: 8990
      bind_host: localhost

logging:
  level: INFO
  appenders:
    - type: console
```

Two things that bite people:

- **`bind_host: localhost` means "this machine only".** If the user needs to reach it from another machine, set the LAN address deliberately and tell them it is now exposed. Do not set `0.0.0.0` silently.
- **Ports 8989/8990 are upstream defaults.** Change both, and keep the admin port different from the API port.

## Step 3 — Import

```bash
java -Xmx8g \
  -Dgraphhopper.datareader.file=data.osm.pbf \
  -Dgraphhopper.graph.location=./graph-cache \
  -jar graphhopper-web.jar import config.yml
```

For OpenRailRouting the flags use the same names but the action is a positional argument:

```bash
java -Xmx8g \
  -Dgraphhopper.datareader.file=data.osm.pbf \
  -Dgraphhopper.graph.location=./graph-cache \
  -jar railway_routing.jar import config.yml
```

**Set expectations before you start.** Import has two phases and the second one is the surprise:

1. Parsing the pbf — usually the shorter phase
2. Building the graph, then **CH preparation for every profile in `profiles_ch`** — this is where most of the wall clock goes, and where memory peaks

Tell the user roughly how long it will take and that the process can look stuck at a percentage for a long time. Do not let them kill it.

## Step 4 — Verify with a request, not with logs

"Started without errors" is not verification. Start the server and send a real request.

```bash
java -Xmx8g \
  -Ddw.graphhopper.datareader.file=data.osm.pbf \
  -Ddw.graphhopper.graph.location=./graph-cache \
  -Ddw.server.applicationConnectors[0].port=8989 \
  -jar graphhopper-web.jar serve config.yml
```

```bash
curl "http://localhost:8989/route?point=39.9042,116.4074&point=31.2304,121.4737&profile=car&points_encoded=false&instructions=false"
```

A healthy response has `paths[0].distance` and `paths[0].time`. Check that `distance` is plausible for the two points — a 5 km answer for two cities means the coordinates were swapped or the graph is wrong.

`point=` in GET is **latitude,longitude**. POST takes `[longitude, latitude]`. Mixing these up is the single most common "it returns nonsense" cause.

## Troubleshooting

Each entry names where the claim comes from. **未实测** means it is derived from reading the source, not observed on a real run — say so when you use it.

### Import dies with `OutOfMemoryError`
- **Where**: almost always during CH preparation, not during parsing.
- **Why**: CH builds shortcut data for every profile; with several profiles this can dwarf the graph itself.
- **Fix**: raise `-Xmx`, or cut profiles. Removing profiles from `profiles_ch` cuts preparation cost most.
- **Source**: code + upstream docs (未实测)

### Import is slow / looks stuck
- **Why**: CH preparation is O(n log n) per profile and single-threaded by default.
- **Fix**: `prepare.ch.threads: N` if there is spare RAM. For OpenRailRouting, the turn-cost pass over every node adds its own fixed cost — it is not a hang.
- **Source**: code (未实测)

### Config change has no effect
- **Why**: the graph is built at import time. Weights are baked into the graph and into the CH shortcuts.
- **Fix**: delete `graph-cache/` and re-import. There is no hot reload.
- **Source**: by design

### Server starts but requests return 400
- **Why**: `profile` is not one of the names in `profiles`. The list is fixed at import time.
- **Fix**: read the profile names out of the config the graph was imported with.
- **Source**: by design

### Server is reachable locally but not on the network
- **Why**: `bind_host` is `localhost`.
- **Fix**: change it to the LAN address, restart. Mention the exposure.

### `Cannot find point` in the response
- **Why**: the point is outside the imported extract, or snapped too far from any road.
- **Fix**: `snap_prevention` does not help here; check the coordinates are inside the pbf's bounding box. Use `points_encoded=false` to see where it actually snapped.

### Graph is much smaller than the pbf
- Not a bug. The import discards everything the active parsers do not accept. A railway-only import of a national extract can be a small fraction of the pbf. Size `-Xmx` from the graph, not from the pbf.

## Measured reference points

Real numbers from an OpenRailRouting deployment, useful for calibrating expectations:

| Item | Value |
|---|---|
| `china-latest.osm.pbf` | 1,292 MB |
| Railway graph-cache, 5 profiles with CH | **228 MB on disk** |
| …of which edges | 28 MB |
| …of which CH shortcuts (5 profiles) | 17–31 MB each |
| …of which turn costs | 9 MB |

The graph is **far smaller than the pbf** because only `railway=*` ways are kept. For a road import the ratio is much closer to the pbf.

## Rules

- Never run a long import without telling the user how long it will take and that it may look stuck.
- Never change `bind_host` to a public interface without saying so.
- Verify with a request, not with "the log said finished".
- If the user has not built a jar and has no Maven, stop and explain the two options rather than improvising a build.
