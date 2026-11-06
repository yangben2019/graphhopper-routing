---
name: graphhopper-road
description: Route cars and trucks over OpenStreetMap with GraphHopper — configure car, truck and hgv profiles, enable vehicle-limit encoded values, choose which highways to ignore, and verify the result. Use when the user mentions 公路路径规划, 自驾路线, 货车导航, car routing, truck routing, hgv, max_height, max_weight, hazmat, 限高限重, or wants to build a road routing service from an .osm.pbf file.
---

# Road routing

Stock GraphHopper has no railway baggage: the graph is large, the profiles are standard, and the main decisions are *which highways to keep* and *which vehicle limits to enable*.

## Pick the profile first

| Profile | Built-in model | Use when |
|---|---|---|
| `car` | `car.json` | Ordinary passenger routing |
| `truck` | `truck.json` | Freight with height/weight/axle limits |
| `hgv` | via `truck.json` + encoded values | Heavy goods, hazmat restrictions |
| `bike` / `foot` | own models | Not covered here |

Start with one profile. Every additional profile in `profiles_ch` multiplies both import time and memory.

## Minimal config

```yaml
graphhopper:
  datareader.file: region.osm.pbf
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
```

`import.osm.ignored_highways` is the single biggest lever on graph size. Dropping footways, cycleways and paths typically removes a large share of the edges for a car-only service.

## Making truck routing actually work

Truck restrictions live in encoded values, so they must be enabled **before import**:

```yaml
graph.encoded_values: car_access, car_average_speed, max_width, max_height,
                      max_weight, max_weight_except, hgv, max_axle_load, hazmat
```

Then a custom model can exclude edges the vehicle cannot use:

```json
{
  "distance_influence": 0.1,
  "priority": [
    { "if": "max_height < 4.0", "multiply_by": "0" },
    { "if": "max_weight < 40", "multiply_by": "0" }
  ]
}
```

Two traps:

- **Missing values mean "no limit", not "blocked"** in most GraphHopper parsers. Decide deliberately which default you want.
- **Encoded values cannot be added later.** Adding `max_height` after an import means re-importing the whole graph.

## Options worth knowing

| Setting | Effect | Cost |
|---|---|---|
| `max_speed_calculator.enabled: true` | Fills `max_speed` from country defaults when OSM has no `maxspeed` | Needs `country` in encoded values; re-import |
| `graph.urban_density.threads: N` | Classifies edges as rural / residential / city | Slow import |
| `graph.elevation.provider: srtm` | Adds elevation, enables ascend/descend | Extra data + import time |
| `profiles_lm` | Landmark prep; slower queries than CH but more flexible | Memory per profile |

`max_speed_calculator` matters most for estimating travel time: without it, roads with no `maxspeed` fall back to class-based defaults that can be far off.

## Verify

```bash
curl "http://localhost:8989/route?point=39.9042,116.4074&point=39.95,116.45&profile=car&points_encoded=false&instructions=true&locale=en"
```

- GET uses `point=lat,lon`; POST uses `[lon, lat]`. Swapping them silently produces a route in the wrong hemisphere — GraphHopper snaps to the nearest road and returns something plausible.
- `points_encoded=false` gives readable coordinates. Use it whenever the output looks wrong.
- `details=street_name,max_speed` shows what the router actually used.

## Troubleshooting

### `Cannot find point` / no route
Check in this order: coordinates inside the extract's bounding box → the point snapped to a valid edge (`points_encoded=false` shows the snapped position) → `import.osm.ignored_highways` did not remove the needed road class → `prepare.min_network_size` did not drop the component as an island.

### Route is much longer than the straight-line distance
Usually `import.osm.ignored_highways` removed a road class the destination depends on, or the profile excludes a ferry that is the only connection.

### Truck route ignores a height restriction
The encoded value was not enabled at import time, or the restriction is tagged in a form the parser does not read. Re-import with the value enabled; there is no way to add it afterwards.

### Import runs out of memory
A road import's graph is close to the pbf size and CH preparation can double the peak. Size `-Xmx` from the pbf, not from the number of profiles.

## Rules

- Adding an encoded value requires a full re-import. Say it before the user changes a config and waits.
- Prefer CH for production queries; use `ch.disable=true` only when a request needs a custom model or flexible weighting.
- Do not silently widen `bind_host`. Localhost is the safe default.
- Verify with a real request. A route that returns 200 with a 5,000 km path is still a failure.
