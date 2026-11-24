# Plan: MobilityDeck, the deck.gl visualisation of MobilityDB

MobilityDeck is the viewer of the MobilityDB ecosystem: one deck.gl application showing the moving
objects of any of its engines. It holds no code of its own for an engine. Display, like indexing
with MEST, is a concern orthogonal to the engines: its primitives live once in MEOS and reach every
engine through the MEOS catalog, as the MEST engine lives once in MEOS and every binding wires it in.

MOVE, the QGIS plugin of the ecosystem, is the second viewer on the same stack: what MEOS.js and
`meos.js/deckgl` do for MobilityDeck in the browser, PyMEOS does for MOVE in the Python of QGIS.

## The structure

| Layer | Repository | Role |
|---|---|---|
| Model and wire format | MEOS | One data model and one serialization, the same bytes in every engine |
| Engines | MobilityDB, MobilityDuck, MobilitySpark, MobilityFlink, MobilityKafka, MobilityNebula | Compute; each is a backend of MobilityAPI |
| Gateway | MobilityAPI | The one contract a client reads: OGC API Moving Features, collections (Part 1) and streams (Part 4) |
| Browser library | MEOS.js, `meos.js/deckgl` | Decodes MEOS values, computes in the browser, builds the deck.gl layers |
| Desktop library | PyMEOS (`pymeos_cffi`) | Decodes MEOS values and computes in the Python of QGIS, as `meos.js` does in the browser |
| Viewers | MobilityDeck (browser), MOVE (QGIS) | One application each; the engine they show is a setting of MobilityAPI, not code |

MobilityAPI holds the backends of MobilityDB (`backend.go`), MobilityDuck (`backend_duck.go`) and
MobilitySpark (`backend_spark.go`) and the stream bridges of MobilityFlink and MobilityKafka; an
engine reaches MobilityDeck once it is a backend of MobilityAPI. `meos.js/deckgl` turns temporal
points into the data of `TripsLayer` (`tgeompointToTrips`, `tgeompointsToTrips`), restricts and
measures them in the browser (`atGeometry`, `atTime`, `speedSeries`, `tripsWithSpeed`) and makes the
layer, over MapLibre or standalone; MobilityAPI's map (`tutorial-stream/map/`) is built on it.

Two paths bypass the gateway, for speed: the vector tiles MobilityDB serves through `pg_tileserv`,
and files a browser reads with `meos.js`.

MOVE reads MobilityDB alone, through `psycopg`: it builds a materialized view per temporal column
(`atTime`, `instants`, `startTimestamp`, the casts of `tgeogpoint` and `tnpoint` to `tgeompoint`),
one row per piece of a sequence whose M counts the seconds since its start, and its Fast preview
(`move_trajectory_item.py`) interpolates the position of each piece on every frame with NumPy. It
holds no MEOS in the client: its interpolation restates in NumPy what MEOS computes, the
inclusivity of the bounds and step interpolation among it.

MOVE-bench, the benchmark of a ULB master's thesis (2026), measured six ways of animating MobilityDB
trajectories in QGIS, median milliseconds per frame on the STIB buses (17k) and the AIS ships (6k):

| Way | STIB | AIS |
|---|---|---|
| `pymeos_cffi` on raw EWKB, `tpoint_value_at_timestamptz` per feature and frame | 5.5 | 15.4 |
| memory layer and expression | 7.5 | 126 |
| MOVE's Fast preview, NumPy over the pieces | 13.3 | 67 |
| columnar NumPy | 22.9 | 28.6 |
| PyMEOS objects, `value_at_timestamp` per feature and frame | 60.5 | 128 |
| MOVE's PostgreSQL layer and expression | 120 | 550 |

Its numbers hold for PyMEOS 1.2 and an earlier MOVE, the fork MOVE-bench carries in `move_plugin/`.

## The display primitives, in MEOS

| Need | MEOS function (SQL name) |
|---|---|
| Interchange text | `temporal_as_mfjson` (`asMFJSON`) |
| Interchange binary | `temporal_as_wkb`, `asHexWKB` |
| Vector tiles | `tpoint_as_mvtgeom` (`asMVTGeom`) |
| Level of detail | `douglasPeuckerSimplify`, `minDistSimplify`, `maxDistSimplify` |
| Chunks in space and time | `tgeo_space_time_split` (`spaceTimeSplit`), `temporal_time_split` (`timeSplit`) |
| Clipping | `atGeometry`, `atTime` |

Each is public in the catalog, so every engine generated from it exposes it, and an engine that does
not is a gap of its parity ledger: MobilityNebula has neither `asMVTGeom` nor `spaceTimeSplit`.

Missing from MEOS: the coordinates and timestamps of each instant of a trip as flat arrays, the form
`TripsLayer` reads, which `meos.js/deckgl` builds today by parsing the MF-JSON text of each trip. It
is an accessor rather than a format: the binary interchange stays `asBinary`, which every type and
engine carries, and the arrays `temporal_timestamps` and `tfloat_values` answer hold distinct values
rather than one per instant.

## Steps, one pull request each

1. **The stream example.** The front end of MobilityFlink-Deck becomes `stream/` of this repository,
   its history kept, reading the stream of MobilityAPI (Part 4) through its Flink bridge rather than
   the Flask, Kafka and WebSocket servers of MobilityFlink-Deck. MobilityFlink-Deck is then archived,
   its README pointing here.
2. **The examples on `meos.js/deckgl` and MobilityAPI.** `mvt/`, `geojsonvt/` and `stream/` move to
   the stack of MobilityAPI's map (deck.gl 9, Vite, MapLibre) and read their trips through
   `meos.js/deckgl`; the types MOVE draws (`tgeompoint`, `tgeogpoint`, `tcbuffer`, `tnpoint`, MOVE #10)
   are drawn too, and the pages of MEOS.js-examples (`trips`, `trajectories`, `compare`) are linked
   or brought in rather than rewritten.
3. **The same display from every engine.** One query through each backend of MobilityAPI answers the
   same MF-JSON, hence the same data of `TripsLayer`; the check compares that data, not pixels.
4. **The frame and load times, measured.** Each choice is measured as MOVE measured its own (MOVE #16
   and #17: from 2848 to 186 ms per frame by holding a layer in chunks of segments, and from about 190
   to 16.8 ms by drawing positions on the canvas rather than through a layer), on the same data at two
   sizes, one like AIS and one like STIB:
   - the trips handed to `TripsLayer` as typed arrays against GeoJSON, and the arrays of the MEOS
     accessor (G121 of MobilityDB #2861) against the MF-JSON `meos.js/deckgl` parses;
   - the tiles of `mvt/` cut by space alone against tiles cut by space and time (`spaceTimeSplit`), so
     that a frame loads only the trips of its time window;
   - trips restricted, simplified and measured by `meos.js` in the browser as they arrive, against
     the same computation in the engine before they are sent;
   - in MOVE, the position of each frame from MEOS per feature (`pymeos_cffi`, the fastest way of
     MOVE-bench) against NumPy over the arrays of the MEOS accessor prepared once per trip, the form
     `TripsLayer` interpolates on the GPU, at both sizes and a larger one, since the first costs a
     call per feature and frame and the second one pass over all features.

   The examples keep the faster of each pair, and the measurements are recorded beside them.
5. **MOVE on the stack.** MOVE reads its trips through MobilityAPI or from files, as MobilityDeck
   does, decodes and restricts them with PyMEOS, and computes each frame the way step 4 keeps; its
   NumPy interpolation of M-encoded pieces goes, and with it the materialized views it builds in
   MobilityDB. PyMEOS installs into the Python of QGIS on Linux, macOS and Windows as part of MOVE.

## Beyond this repository

- **MEOS:** the coordinates and timestamps of each instant as parallel arrays, one public accessor
  in the shape the portable naming plan gives parallel arrays (row G121 of MobilityDB #2861).
- **MobilityAPI:** a backend for MobilityNebula, and `asBinary` as its binary response (row IO-B24
  of MobilityDB #3788).
- **MEOS.js:** `meos.js/deckgl` calling that accessor once per trajectory once MEOS has it.
- **PyMEOS:** the same accessor through `pymeos_cffi`, for MOVE.
- **MOVE-bench:** archived once its author agrees and step 4 has cited its measurements, its README
  pointing to MOVE.
- **MobilityNebula:** `asMVTGeom` and the split functions in its plugin, as its parity ledger states.
