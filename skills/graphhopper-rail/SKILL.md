---
name: graphhopper-rail
description: Route trains over OpenStreetMap with OpenRailRouting — choose among the five shipped profiles (tgv_all, non_tgv, tramtrain, all_tracks, all_tracks_1435), compose the eight custom models, and explain why a connected track still returns no path. Use when the user mentions railway routing, train routing, rail_access, gauge, voltage, electrified, preferred_direction, or asks why a rail route is missing, slower than expected, or avoids the obvious line.
---

# Railway routing

OpenRailRouting is a GraphHopper fork for railways (Geofabrik + SNCF, Apache-2.0). It keeps GraphHopper's routing algorithms and changes the **graph model**: railway-specific encoded values, edge duplication, and geometric turn restrictions.

## Language

Reply in the language the user wrote in. Keep profile names, encoded values, custom model filenames and commands exactly as they are — they are identifiers, not prose. Translate headings and explanations, never identifiers.

## 中文说明

OpenRailRouting 的铁路路由。五个 profile：`tgv_all` / `non_tgv` / `tramtrain` / `all_tracks` / `all_tracks_1435`，由八个 custom model 叠加而成。

最容易踩的三点：

1. **线路连着却算不出路** —— 转角超过 135° 被禁止；`railway=railway_crossing` 只许直穿
2. **时间不对** —— 速度按 `usage`/`service` 推断，没打 `maxspeed` 的路段会落到 25 km/h
3. **改了模型不生效** —— 权重在导入时烘进图里，必须重导

## The five shipped profiles

Each profile = a vehicle plus a stack of custom models. Compose them in order; later files refine earlier ones.

| Profile | Vehicle types | U-turn penalty | Model files | Meaning |
|---|---|---|---|---|
| `tgv_all` | train | 300 s | `rail` + `tgv_all` + `preferred_direction` | High-speed train. Matches 15 kV AC, 25 kV AC, 3 kV DC, 1.5 kV DC. Cap 310 km/h. Excludes yard and spur. |
| `non_tgv` | train | 900 s | `rail` + `non_tgv` + `preferred_direction` | Diesel train. Cap 120 km/h. |
| `tramtrain` | train, tram, light_rail | 180 s | `tramtrain` + `15kv-ac_750v-dc` + `gauge_1435` + `preferred_direction` | Tram-train. Standard gauge, 15 kV AC or 750 V DC. |
| `all_tracks` | train, tram, light_rail, subway | 180 s | `all_tracks` | Any track with rail access, no filtering. |
| `all_tracks_1435` | same | 180 s | `all_tracks` + `gauge_1435` | Any track, standard gauge only. |

`all_tracks` versus `all_tracks_1435` differ by exactly one model file. That is the cheapest way to explain the profile system to a user.

## The eight custom models

| File | What it does |
|---|---|
| `rail.json` | Base filter: drop edges without `rail_access` or with `railway_class != RAIL`. Speed = `rail_average_speed`. |
| `tgv_all.json` | Match power system by voltage and frequency; cap 310. Drop `YARD` and `SPUR`. |
| `non_tgv.json` | Diesel; cap 120; no power-system filter. |
| `tramtrain.json` | Allow `RAIL`, `TRAM`, `LIGHT_RAIL`. |
| `all_tracks.json` | Allow everything with `rail_access`. |
| `gauge_1435.json` | Drop edges where `gauge != 0 && gauge != 1435`. **0 means unknown and is allowed.** |
| `15kv-ac_750v-dc.json` | Power-system filter for 15 kV AC 16.7 Hz or 750 V DC; cap 100. |
| `preferred_direction.json` | Reverse running gets `priority × 0.7`. Soft penalty, not a ban. |

They ship **inside the jar** at `com/graphhopper/custom_models/`. A `custom_models/` folder next to `config.yml` is optional and overrides by filename — that is how you customize without recompiling.

## Railway-specific encoded values

GraphHopper has no concept of these; OpenRailRouting adds them.

| Value | Type | Notes |
|---|---|---|
| `gauge` | integer | Missing = 0 |
| `voltage` | decimal | 100 V precision; missing = 0.0 |
| `frequency` | decimal | 2.5 Hz precision; missing = 0.0 |
| `electrified` | enum | `UNSET`, `NO`, `OTHER`, `CONTACT_LINE`, `RAIL` |
| `railway_class` | enum | `RAIL`, `LIGHT_RAIL`, `TRAM`, `SUBWAY`, `NARROW_GAUGE`, `FUNICULAR` |
| `railway_service` | enum | `NONE`, `SIDING`, `YARD`, `SPUR`, `CROSSOVER` |
| `railway_preferred_direction` | boolean | True when travel matches the tagged direction |

All of them must be listed in `graph.encoded_values`, or the custom models reference values that were never stored.

## Why routes surprise people

Three behaviours that are not documented in the README and explain most "that can't be right" reports.

### 1. A connected track can still be unreachable
Two behaviours in the turn-cost pass:

- Any turn sharper than **135°** is forbidden. Trains cannot take a right-angle curve.
- At `railway=railway_crossing` nodes, **only near-straight-through (165.6°–194.4°)** is allowed. Two tracks crossing geometrically do not connect physically.

So when a user says "the lines touch on the map but there is no route", this is the first thing to check. In practice it usually means the data is missing a connector, or the two ways share a node that is really a crossing.

### 2. Speed is inferred, not read
Few rail ways carry `maxspeed`. The parser derives a base speed from other tags:

```
service=siding    40 km/h
service=yard      25 km/h
service=crossover 60 km/h
usage=main       100 km/h
usage=branch      50 km/h
otherwise         25 km/h
```

If `maxspeed` exists it is used as `maxspeed × 0.9` — a 10% allowance for acceleration and braking distance. Unlabelled track therefore routes at 25 km/h, which makes ETAs look wrong.

### 3. Edges multiply at import
A way with `gauge=1435;1520` becomes two parallel edges, one per value. Power systems multiply the same way. This is why the edge count is higher than the number of rail ways, and why changing a custom model requires a full re-import.

## Verify a rail route

```bash
curl "http://localhost:8989/route?point=39.9042,116.4074&point=31.2304,121.4737&profile=all_tracks&points_encoded=false&instructions=false"
```

Add details to see what the route actually used:

```bash
curl "http://localhost:8989/route?point=39.90,116.40&point=31.23,121.47&profile=all_tracks&details=rail_average_speed&details=gauge&details=voltage&points_encoded=false"
```

If `details` returns nothing useful, the encoded values were probably not enabled in `graph.encoded_values` at import time.

## Choosing a profile

Ask what the user is routing, then map it:

| The user wants | Profile |
|---|---|
| High-speed passenger line | `tgv_all` |
| Freight or diesel passenger | `non_tgv` |
| Tram or light rail | `tramtrain` |
| Anything that runs on rails, no filtering | `all_tracks` |
| Standard gauge only | `all_tracks_1435` |

If none fits, compose a custom model rather than picking the nearest one — a wrong profile silently produces plausible but wrong routes.

## Licensing

The custom models and the engine are **Apache-2.0** (Geofabrik and SNCF). If you copy model files into a project, keep the license notice and credit the origin. See `THIRD_PARTY.md` in this repository.

## Rules

- Never claim the engine was modified unless the user actually built a modified fork. Stock OpenRailRouting is the normal case.
- When a route is missing, check the geometric turn rules **before** suspecting the data is wrong.
- When an ETA looks wrong, check whether the ways carry `maxspeed` or are falling back to the 25 km/h default.
- Changing a custom model means re-importing. Say this before the user spends an hour debugging a change that cannot take effect.
