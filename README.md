# GraphHopper Routing

[![skills.sh](https://skills.sh/b/yangben2019/graphhopper-routing)](https://skills.sh/yangben2019/graphhopper-routing)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Self-hosted routing from an OSM `.pbf` — deploy it, route trains, route cars and trucks.**

Three Claude Code skills that take you from "I have a map file" to "I can curl a route", and tell you why it broke when it does. The engines (GraphHopper, OpenRailRouting) are not bundled — the skills drive the copies you install.

---

## Skills

| Skill | What it does |
|---|---|
| **graphhopper-selfhost** | The foundation: pick an engine, write a minimal config, run the import, watch for the CH-preparation phase, start the API, and verify with a real request. Includes memory sizing from measured data and a troubleshooting table for the three places imports usually fail. |
| **graphhopper-rail** | Railway routing with OpenRailRouting: the five shipped profiles, the eight custom models and how they compose, the railway-specific encoded values, and the two behaviours that explain most "that route is wrong" reports — geometric turn restrictions and inferred speeds. |
| **graphhopper-road** | Car and truck routing with stock GraphHopper: profile choice, which highways to discard, truck limit encoded values, country default speeds, and how to verify that the route is actually sane. |

### 中文简介

- **自托管部署**：从 pbf 到 `/route` 可用，含内存规划、CH 预计算、排障表。
- **铁路路径规划**：OpenRailRouting 的 5 个 profile、8 个 custom model、轨距/电压/电气化编码值，以及「线路连着却算不出路」的真实原因。
- **公路路径规划**：car / truck / hgv，限高限重、危险品、国家默认限速。

---

## Install

### Option 1 — skills CLI

```bash
npx skills add yangben2019/graphhopper-routing
```

Single skill:

```bash
npx skills add yangben2019/graphhopper-routing --skill graphhopper-rail
```

### Option 2 — Claude Code plugin marketplace

```text
/plugin marketplace add yangben2019/graphhopper-routing
/plugin install graphhopper-routing@graphhopper-routing
```

### Option 3 — manual

```bash
git clone https://github.com/yangben2019/graphhopper-routing.git /tmp/gh
cp -r /tmp/gh/skills/* ~/.claude/skills/
```

---

## Usage

```text
我下载了 china-latest.osm.pbf，想在本机起一个铁路路径规划服务，内存 32G。
```

```text
Import keeps dying with OutOfMemoryError during CH preparation. What do I cut?
```

```text
The two lines touch on the map but the router returns no path. Why?
```

```text
I need truck routing that respects height and weight limits.
```

---

## What these skills are, and are not

**They are:** an operator's manual for the model. Ask a question in plain language; Claude reads the skill, runs the checks, writes the config, and interprets the failures.

**They are not:** a tutorial, an installer, or a replacement for the engines. The import itself — GBs of data, tens of minutes, tens of GB of RAM — still happens on your machine.

---

## Measured reference points

From a real OpenRailRouting deployment, used to calibrate expectations:

| Item | Value |
|---|---|
| `china-latest.osm.pbf` | 1,292 MB |
| Railway graph, 5 profiles with CH | 228 MB on disk |
| — edges | 28 MB |
| — CH shortcuts | 17–31 MB per profile |
| — turn costs | 9 MB |

A railway graph is far smaller than its pbf because only `railway=*` ways survive the import. A road graph is much closer to the pbf size.

---

## Acknowledgement

The skills document [OpenRailRouting](https://github.com/geofabrik/OpenRailRouting) (Geofabrik GmbH and SNCF, Apache-2.0) and [GraphHopper](https://github.com/graphhopper/graphhopper) (Apache-2.0). No upstream source is bundled. See [THIRD_PARTY.md](THIRD_PARTY.md).

---

## License

MIT © 2026 yangben2019
