# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test

```bash
make -j          # build all binaries
make test        # full test suite (includes geobuf)
make fewer-tests # faster subset (skips geobuf)
make indent      # clang-format all source
make clean       # remove build artifacts
make install     # install to /usr/local (override with PREFIX=...)
```

Build flags: `BUILDTYPE=Debug` (default Release), custom `CC`, `CXX`, `CFLAGS`, `CXXFLAGS`, `LDFLAGS`.

Unit tests run via `./unit` (built from `unit.cpp`, tests `text.cpp`).

## What This Is

C++11 geospatial toolchain that converts GeoJSON/Geobuf/CSV into Mapbox Vector Tile (MVT) tilesets stored as MBTiles (SQLite) or directory trees. Core insight: at low zoom, drop *least-visible* features rather than simplifying geometry—preserving data density and regional texture.

## Executables

| Binary | Entry point | Purpose |
|--------|-------------|---------|
| `tippecanoe` | `main.cpp` | Main feature→tileset pipeline |
| `tippecanoe-decode` | `decode.cpp` | MBTiles/PBF → GeoJSON |
| `tippecanoe-enumerate` | `enumerate.cpp` | List tiles in an MBTiles |
| `tile-join` | `tile-join.cpp` | Merge/filter/subset MBTiles |
| `tippecanoe-json-tool` | `jsontool.cpp` | JSON sorting/CSV integration |
| `unit` | `unit.cpp` | Unit test runner |

## Architecture

**Processing pipeline** (all in `main.cpp`):
```
CLI parse → read input (geojson/geobuf/csv) → serialize features to temp files
→ build spatial index → traverse_zooms() → write_tile() per zoom
→ encode MVT/PBF → write MBTiles (mbtiles.cpp) or dirtiles (dirtiles.cpp)
```

**Key module responsibilities:**
- `geometry.cpp/hpp` — coordinate transforms, drawing command encoding, simplification
- `tile.cpp/hpp` — zoom traversal, per-tile feature collection
- `mvt.cpp/hpp` — Mapbox Vector Tile PBF encoding
- `mbtiles.cpp/hpp` — SQLite MBTiles I/O
- `serial.cpp/hpp` — feature serialization between pipeline stages
- `evaluator.cpp/hpp` — filter expression evaluation
- `projection.cpp` — Web Mercator and custom projections
- `jsonpull/` — streaming JSON parser (embedded library)
- `geojson.cpp`, `geobuf.cpp`, `geocsv.cpp` — input format parsers

**Limits** (defined in `main.hpp`): `MAX_ZOOM=24`, default tile max 500KB, max 200K features/tile.

**Parallelism**: `-P` flag enables multi-file parallel processing; uses `CPUS` (detected core count) and temp files for overflow.

## Test Structure

`tests/` holds 40+ fixture directories. Each test compares output tiles/JSON against a `stable/` baseline. To add a test, create a directory with input files and update the relevant test target in `Makefile`.
