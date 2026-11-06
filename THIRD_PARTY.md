# Third-party notices

This repository contains **no source code copied from any upstream project**. The skills document publicly observable behaviour of tools the user installs themselves.

## OpenRailRouting

- Upstream: https://github.com/geofabrik/OpenRailRouting
- Maintained by: Geofabrik GmbH and SNCF
- License: Apache License 2.0
- Role in this repository: the `graphhopper-rail` skill documents OpenRailRouting's profiles, custom models, encoded values and deployment commands. The engine is **not bundled**; users install or build it themselves.

## GraphHopper

- Upstream: https://github.com/graphhopper/graphhopper
- License: Apache License 2.0
- Role in this repository: the `graphhopper-selfhost` and `graphhopper-road` skills document GraphHopper's HTTP API, configuration format and deployment commands. The engine is **not bundled**.

## OpenStreetMap data

- Upstream: https://www.openstreetmap.org/
- License: Open Data Commons Open Database License (ODbL)
- Role in this repository: referenced as the input format (`.osm.pbf`). No map data is included.

---

If a future change adds bundled configuration files or model files from OpenRailRouting or GraphHopper, the Apache-2.0 license text and attribution must be included at that point.
