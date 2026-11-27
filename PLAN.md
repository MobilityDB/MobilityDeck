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
| Viewers | MobilityDeck (browser, deck.gl), MOVE (QGIS), and one for each other environment: MobilityOpenLayers, MobilityLeaflet, MobilityGeoServer, Jupyter notebooks (`notebook/` of MobilityDeck) | One application each; the engine they show is a setting of MobilityAPI, not code |

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

## The viewer of each environment

The ecosystem has a viewer for each environment its users work in. MobilityDeck in the browser and
MOVE in QGIS are the two the steps of this repository build. The other repositories of the
organization that display MobilityDB data use neither MEOS.js, MobilityAPI nor the catalog, each
with its own server or its own SQL. A viewer of another environment joins the stack in its own
repository (step 8); a viewer MOVE subsumes in QGIS is archived:

| Repository | Environment | What it holds | On the stack |
|---|---|---|---|
| MobilityQGIS | Scripting in the Python console of QGIS, where a user animates trips with code of their own | The experiments of a ULB MA1 project: six scripts that, on each frame of the temporal controller, compute the position of each trip with `valueAtTimestamp` of MobilityDB-python (an archived driver whose place PyMEOS holds) and add it to a memory layer, over `atPeriod` and `period`, which MobilityDB does not declare (its names are `atTime` and `tstzspan`) | Subsumed by MOVE, the viewer of the same environment, which builds its layers from any query and sets their temporal properties (`QgsVectorLayerTemporalProperties`) for the temporal controller; its way of computing a frame, MEOS per feature and frame in the client, is the first arm step 4 measures for MOVE. Archived, its README pointing to MOVE and its report cited beside MOVE-bench |
| MobilityOpenLayers | Browser GIS applications combining many sources and OGC services | An OpenLayers page over a file of trips (`trips.json`) | Its trips read from MobilityAPI and decoded by `meos.js`, its tiles cut by `asMVTGeom` |
| MobilityLeaflet | Lightweight browser maps | A 2023 master's thesis: a Leaflet front end, its own WSGI server and its own SQL functions | Its trips read from MobilityAPI, which takes the place of its server and its SQL functions, and decoded by `meos.js` |
| MobilityGeoServer | Organizations publishing through the OGC services of a GeoServer | A page showing with Leaflet the layers a GeoServer publishes | GeoServer publishes, as plain geometries, what MobilityDB computes (the position of each object at a time, its trajectory over an interval), and reads no MEOS type itself |
| Franchise | A notebook SQL client in the browser or the desktop (Electron) | A fork of HVF/franchise, three commits on its 2019 stack (React 15, Electron 1.7, nwb), whose MobilityDB cell draws a column whose type is `tgeompointseq`, a type MobilityDB does not declare | Subsumed by the notebooks of `notebook/`, which serve the same environment, as the next row states. Archived after the notebooks of step 8, its README pointing to them |
| MobilityDeck `notebook/` | Jupyter notebooks, the SQL notebook environment of the ecosystem | Nothing yet | SQL cells through JupySQL (`%%sql`) over MobilityDB, MobilityDuck or any engine with a Python driver, and a widget drawing a temporal column on a map through `meos.js/deckgl`, the adapter every browser viewer shares; the code of the ecosystem, no fork of another application |

## Steps, one pull request each

1. **The stream example.** The front end of MobilityFlink-Deck becomes `stream/` of this repository,
   its history kept, reading the stream of MobilityAPI (Part 4) through its Flink bridge rather than
   the Flask, Kafka and WebSocket servers of MobilityFlink-Deck. Its data path and its interpolation
   in JavaScript give way to those of MobilityAPI's map (`tutorial-stream/map`), which moves here
   with it: MEOS.js computes each position with `valueAtTimestamp` on one clock, and the replay,
   follow and statistics of MobilityFlink-Deck stay. The README of MobilityAPI's tutorial points
   here. MobilityFlink-Deck is then archived, its README pointing here.
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
6. **The types beyond points.** A `tgeometry` and a `trgeometry` are drawn as the polygon of each
   frame, a `th3index` as the hexagon of each frame (`H3HexagonLayer`), a `tcbuffer` as its circle,
   and a `tfloat` or `tint` attribute of a trip as a chart whose cursor follows the clock of the
   map. The value of each frame is `valueAtTimestamp`, which MEOS declares over each of these types
   and the client computes nothing of; the shape of an instant is the one its type states (step
   interpolation of a `tgeometry` holds its polygon until the next instant).
7. **One clock for the past and the stream.** The clock of MobilityDeck reads a stored trip through
   MobilityAPI Part 1 and a live one through Part 4, and its slider passes from the one to the
   other; a live trip shows the watermark of the stream it reads, the instant up to which it is
   complete.
8. **The viewer of each environment on the stack.** MobilityOpenLayers, MobilityLeaflet,
   MobilityGeoServer and the notebooks of `notebook/` each take the form the table above states,
   one pull request in each repository, and each shows the trips of step 3's check, so that every
   viewer draws the same data. Franchise is archived after the notebooks, its README pointing to
   them.

## Prerequisites and order

The steps need the following, outside the viewers:

| Row | Repository | What | Needed by |
|---|---|---|---|
| V1 | MobilityDB | The accessor of the coordinates and timestamps of each instant (row G121 of MobilityDB #2861): `double *tpoint_coords(const Temporal *temp, double **ycoords, double **zcoords, TimestampTz **times, int *count)`, the inverse of `tpointseq_make_coords`, whose arrays it answers: the x coordinates returned, the y and z coordinates (none for a point without Z) and the timestamps as out-parameters, so a sequence is made again from them; a binding converts the timestamps to its host's through `timestamptz_to_unix_us`, and `meos.js/deckgl` interleaves the coordinates `TripsLayer` reads. No SQL function, being an accessor for the bindings | steps 4, 5 |
| V2 | MEOS-API, MEOS.js, PyMEOS | V1 in the catalog, the two bindings regenerated and released, the published packages being what the viewers install (`meos.js` 1.0.2 on npm, `pymeos-cffi` 1.3.0 on PyPI) | steps 4, 5 |
| V3 | PyMEOS-CFFI | Windows wheels: PyPI holds wheels for Linux and macOS and the source alone, so the Python of QGIS on Windows cannot install it | step 5 |
| V4 | MobilityAPI | `asBinary` as its binary response (row IO-B24 of MobilityDB #3788) | step 4 |
| V5 | MobilityAPI, MobilityFlink | The watermark of a stream carried through the bridge to each event; no Go source of MobilityAPI carries one | step 7 |
| V6 | MEOS.js | `meos.js/deckgl` drawing polygons, hexagons, circles and series of values; it reads temporal points alone, from their MF-JSON | step 6 |
| V7 | MobilityAPI | A backend for MobilityNebula; step 3 compares three engines without it | step 3, the fourth engine |
| V8 | MobilityQGIS | Archived, its README pointing to MOVE | none |
| V9 | MobilityAPI | A continuous query over the moving features of a whole collection, `/collections/{cid}/tgsequence/queries`, each event carrying the identifier of its feature; a geometry query streams one feature (`/collections/{cid}/items/{fid}/tgsequence/queries`), so a live fleet takes one subscription per object. As the queries of one feature, it claims no OGC class | steps 1, 7 |

Steps 1, 2 and 3, the OpenLayers, Leaflet, GeoServer and notebook viewers of step 8, and V1, V3,
V4 and V9 start at once and in parallel; the fleet of step 1 reads V9. Step 4 follows V1 and V2,
step 5 step 4 and V3, step 6 V6, step 7 V5 and V9.

The risk: MOVE on Windows rests on V3, which the build of PyMEOS-CFFI does not produce.

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

## Left out, and why

- **A new repository for the shared display code.** `meos.js/deckgl` is that code for the browser
  and PyMEOS for QGIS; a third home would copy them.
- **Apache Arrow as the transport.** The binary interchange of every type and engine is `asBinary`
  (IO-B24 of MobilityDB #3788), and the flat arrays `TripsLayer` reads are the MEOS accessor of
  G121. Arrow returns only if step 4 measures a cost these two leave.
- **A GeoServer data store or a WMS `TIME` dimension.** A tile or a map image holds the value of a
  type at one time and loses its interpolation; the tiles come from `asMVTGeom`, and the temporal
  value from MobilityAPI.
- **A QGIS Processing provider.** It would send MOVE's computation back to SQL in MobilityDB alone,
  where step 5 moves it to MEOS in the client for every engine.
- **A conformance suite across viewers.** Step 3 is that suite: the same query through each engine
  gives the same data of each layer, compared as data, not pixels.
