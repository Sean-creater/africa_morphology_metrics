# Africa Urban Morphology Metrics

Extracts building footprints and road networks for African cities and
computes building-form, street-network, connectivity, and
spatial-distribution morphology metrics, aggregated onto a 250 m
hexagonal grid. Covers the ~8,393 city/town agglomerations in the
AFRICAPOLIS inventory across 54 African countries.

This is a companion pipeline to
[`africa_hex_grid_workflow`](https://github.com/Sean-creater/africa_hex_grid_workflow),
which builds the 250 m hexagonal grid itself and attaches building
height/volume (WSF3D), population (WorldPop), and land cover (Copernicus)
statistics. This pipeline **consumes that grid as an input** — specifically
its per-city `{City}_grid_250m_hex_enriched.shp` and
`{City}_grid_boundary.geojson` outputs — and extends it with building
morphology, street-network, connectivity, and spatial-distribution metrics
derived from Google Open Buildings and OpenStreetMap, before running its
own multi-stage data-cleaning/QC process to produce one standardized
attribute table + shapefile per city. See that repository's README for how
the grid itself is built, its CRS-handling rationale, and its AFRICAPOLIS/
Natural Earth boundary-file sourcing — this README does not repeat that
material.

The grid is a required **data input** to the published code. This repository
does not execute an external grid-generation script; prepare the companion
grid outputs before running it.

## Pipeline stages

1. **Open Buildings extraction** (`gcs_buildings.py`) — download building
   footprint polygons for a city's AOI from Google Open Buildings v3, via
   direct Google Cloud Storage access.
2. **OSM network extraction** (`osm_network.py`) — download a country's OSM
   road network from Geofabrik (once per country, cached), clip to each
   city boundary.
3. **Morphology metrics** (`morphology_metrics.py`) — per-building
   dimension, shape, complexity, and 3D (height/volume-derived) metrics via
   momepy, aggregated to grid cells.
4. **Street network metrics** (`streetnetwork_metrics.py`) — street
   density, width, orientation, and building-to-street-alignment metrics
   from the clipped OSM network.
5. **Connectivity metrics** (`connectivity/`) — graph-theoretic street
   network metrics (centrality, meshedness, intersection typology, COINS
   street continuity) computed per grid cell.
6. **Spatial distribution metrics** (`spatial_distribution_metrics.py`) —
   building-to-building alignment, inter-building distance, and
   neighbor-count statistics via a parallelized, zero-copy k-nearest-
   neighbor pass.
7. **Aggregation to grid** — every metric family above is joined onto the
   250 m hex grid inherited from the companion pipeline.
8. **Data cleaning / QC** (`cleaning/`) — an 11-script sequence that brings
   every city's output to a common 172-column schema, fixes known
   country-specific data quirks, runs QC checks, and joins the cleaned
   attribute table onto grid geometry to produce the final SHP output. See
   `cleaning/README.md` for the full script-by-script run order.

The orchestrator (`pipeline.py`) drives stages 1-7 per city, with
checkpointing (`checkpoint.py`) and coverage-gap tracking (`trackers.py`)
across large batch runs, and optional email notifications
(`notifications.py`).

## Installation

```bash
conda env create -f environment.yml
conda activate africa_morphology_metrics
# or: pip install -r requirements.txt
pip install -e .          # optional, for the console entry point
pip install -e geoworkflow/  # the vendored extraction backend (see below)
```

Python 3.10+ is required. `python-igraph` is optional but strongly
recommended — the connectivity module automatically falls back to a pure
NetworkX implementation if it is not installed (see "Why three
connectivity implementations" below).

### The vendored `geoworkflow` package

`geoworkflow/` at the repository root is a general-purpose geospatial
extraction/processing library (Cornell University, MIT licensed — see
`geoworkflow/LICENSE`) that this pipeline's Open Buildings extraction code
builds on directly: `gcs_buildings.py` subclasses
`geoworkflow.processors.extraction.open_buildings_gcs.OpenBuildingsGCSProcessor`.
It is vendored here (rather than referenced as an external dependency)
because it is not currently published to PyPI or a public repository. It
carries its **own MIT license, distinct from the rest of this pipeline**
(see "License" below).

A small number of other pieces of logic that are conceptually similar to
`geoworkflow` utilities, but small and genuinely self-contained rather than
part of the larger processor framework, are implemented directly in this
package's own modules instead of importing the vendored copy:
`geofabrik.py` (PBF caching), `s2_coverage.py` (S2-cell tiling), and
`osm_clipping.py` (AOI clipping, including a tiled parallel path for
country-scale networks).

`geoworkflow/` is vendored as a complete, coherent library rather than
trimmed to only the pieces this pipeline calls, since it is independently
versioned and maintained. This means it includes code this pipeline does
not use — most notably an Earth Engine-based building-extraction path
(`processors/extraction/open_buildings.py`, `utils/earth_engine_utils.py`)
that is an alternative to, and entirely separate from, the GCS-based
extraction (`open_buildings_gcs.py`) this pipeline actually calls. It
requires no credentials to import and contains none; it reads Earth Engine
credentials only from a user-supplied service-account key path or the
standard `GOOGLE_APPLICATION_CREDENTIALS` environment variable, exactly
like the GCS path does for its own credentials.

## Data source requirements

None of the following are bundled in this repository.

| # | Source | Used for | Access pattern |
|---|--------|----------|-----------------|
| 1 | **Google Open Buildings v3** | Building footprint polygons | Public, anonymous Google Cloud Storage access via the `google-cloud-storage` SDK (`storage.Client.create_anonymous_client()` — no credentials needed). Bucket: `gs://open-buildings-data/v3/polygons_s2_level_6_gzip_no_header`, gzip-compressed CSV partitioned by S2 level-6 cell, downloaded in 10 MB chunks. This is roughly 2-3x faster than the equivalent `gcsfs`-based access pattern for the same data. |
| 2 | **OpenStreetMap, via Geofabrik** | Road/highway network | Country-level `.osm.pbf` extracts downloaded from `https://download.geofabrik.de/africa/{region}-latest.osm.pbf`, cached locally under `~/.geoworkflow/osm_cache/` (one PBF per country, with a JSON sidecar recording download date; a 30-day-old cache is used with a warning rather than forcing a re-download). Extraction from the PBF tries `pyogrio` first (GDAL-backed, bbox-filtered — roughly 10-50x faster and dramatically lower memory than the alternative; e.g. Tanzania's 668 MB PBF: 8+ minutes and ~24 GB RAM with the fallback path vs. 2-5 seconds and 100-500 MB with pyogrio), falling back to `pyrosm` (which loads the full country network into memory) only if pyogrio fails or returns no results. |
| 3 | **Companion 250 m hex grid** (`africa_hex_grid_workflow`) | The base grid this pipeline's metrics are aggregated onto | See that repository. Provides `{City}_grid_250m_hex_enriched.shp` and `{City}_grid_boundary.geojson` per city. |
| 4 | **AFRICAPOLIS agglomeration boundaries** and **country/administrative boundaries** | City AOI definition, city-to-country lookup, phased-processing city lists | Same sources as the companion grid pipeline — see its README for details. |

## Output schema

Every metric below is joined onto the 250 m hex grid inherited from the
companion pipeline (which already carries `GridID`, `Area_m2`, land cover,
WSF3D height/volume, and population columns — see that repository's
README for those). Full short-name-to-description mapping is in
`data/reference/col_name_mapping.csv`; a fresh copy reflecting exactly what
the current codebase computes is regenerated at
`data/reference/column_name_mapping.csv` at the end of every pipeline run
(`morphology_metrics.generate_column_mapping_with_categories`).

### Building morphology (`morphology_metrics.py`)

Computed per building via momepy, then aggregated to grid cells as
mean/max/std (dimension/shape metrics) or mean/std (most others):

- **Dimension**: Area, Perimeter, CourtyardArea, LongestAxisLength.
- **Shape**: CircularCompactness, Squareness, EquivalentRectangularIndex,
  Elongation, CompactnessWeightedAxis, Convexity, Rectangularity,
  SquareCompactness, CentroidCorners, CourtyardIndex, FacadeRatio,
  FractalDimension, ShapeIndex.
- **3D metrics** (require the WSF3D height/volume rasters —
  `height_raster_path` / `volume_raster_path`): **FormFactor**
  (momepy 3D compactness), **H2F_Ratio** (height / sqrt(footprint area)),
  **FloorARat** (estimated floor count from height, assuming 3 m floors
  with a 2 m minimum-remainder threshold, aggregated to floor-area ratio),
  **VolDe** / VolumeDens (building volume density from the WSF3D volume
  raster, independent of the grid's own inherited `V_*` columns — see
  "Design notes" below).
- **Complexity**: Corners (mean/max/std).
- **Intensity**: BldgDens, CoverRatio, OpenRatio, PerimDens, CourtDens,
  ShareDens (building-share-of-cell-area statistics).
- **Diversity**: Std_* variants of the above (within-cell heterogeneity).

### Street network (`streetnetwork_metrics.py`)

StreetDens, NodeDens, StreetsPN, SegmentDen, PlotDens, BldgDensN,
PlotFrag, StrWidth/StrOpen/StrWidthD, StrLinM/StrLinS (street linearity),
StrOrientM/StrOrientS (street orientation), BldgStrAM/BldgStrAS
(building-to-street alignment). Intersection-topology metrics
(DeadendDen, InterDens, Ratio3way, Ratio4way, CuldesacR) are deliberately
**not** computed here even though they appear in the shared column
mapping — they are computed once by the connectivity module (which builds
its own per-cell subgraphs) to avoid being computed twice and potentially
disagreeing.

### Connectivity (`connectivity/`)

31 metrics per grid cell, computed from the clipped street network graph:
node centrality (BtwCentM/X/S, ClsCentM/X/S, StrCentM/X/S — betweenness,
closeness, and straightness centrality, mean/max/std), clustering
(ClusterM/S), node density (NdNbDenM/S), graph topology (MeanDeg, StdDeg,
MeanNdDst, Cyclomat, EdgeNdRat, GammaIdx, Meshedness), edge centrality
(EdgeBtwM, EdgeClsM, EdgeStrM), intersection typology (DeadendDen,
InterDens, Ratio3way, Ratio4way, CuldesacR, CuldesacL), and **COINS**
street-continuity metrics (COINSCnt, COINSLen — see below).

### Spatial distribution (`spatial_distribution_metrics.py`)

Building-to-building alignment, inter-building distance, and k-nearest-
neighbor count statistics, computed via a zero-copy shared-memory
parallel pass (buildings are passed to the worker pool once; workers
receive only small index lists per task).

## Why three connectivity implementations coexist

`connectivity/` ships three implementations of `calculate_connectivity_metrics`,
selected automatically at import time:

```python
try:
    import igraph as ig
    try:
        from .connectivity_igraph_primary import calculate_connectivity_metrics  # primary
    except ImportError:
        from .connectivity_igraph_buffered import calculate_connectivity_metrics
except ImportError:
    from .connectivity_networkx import calculate_connectivity_metrics  # igraph unavailable
```

The fallback is triggered purely by whether `import igraph` succeeds — not
by any runtime performance check.

- **`connectivity_networkx`** — pure NetworkX/momepy. Its core
  contribution is a **per-cell subgraph cache**: rather than rebuilding
  each grid cell's local street subgraph separately for the centrality
  pass, the topology pass, and the edge pass, it builds each cell's
  subgraph once (optionally pickled to disk) and reuses it across every
  metric family in one worker pass. Uses a 6-neighbor cell buffer so a
  cell's subgraph includes some adjacent-cell context. This is also the
  sole source of `get_connectivity_column_mappings`, the shared output
  schema used by all three backends.
- **`connectivity_igraph_buffered`** — same buffered-subgraph-cache
  strategy as the NetworkX backend, but stores each cached subgraph in
  both NetworkX and igraph form, routing the three centrality
  computations (betweenness, closeness, straightness) and edge
  betweenness through igraph's C implementation. Node-degree/density,
  cyclomatic complexity, edge-node ratio, gamma index, meshedness, and
  cul-de-sac length stay on NetworkX/momepy, since those are already
  linear-time and not worth converting.
- **`connectivity_igraph_primary`** (primary) — builds on the buffered
  igraph backend's hybrid approach but removes the 6-neighbor cell buffer
  entirely: each cell's subgraph contains only nodes physically within
  that cell, with no adjacency-graph computation. To compensate for the
  lost context, it splits metrics into a *topology* subgraph (counts all
  edges touching a cell-internal node, even if the edge extends outside
  the cell — used for Cyclomat, EdgeNdRat, GammaIdx, Meshedness) and a
  *geometry* subgraph (cell-internal edges only, avoiding distortion from
  long edges that extend outside the cell — used for MeanNdDst,
  EdgeBtwM/ClsM/StrM, CuldesacL, centrality). It also computes
  intersection-degree counts (DeadendCnt/InterCnt/3-way/4-way) from each
  node's *global* graph degree rather than its subgraph degree, since
  subgraph extraction can otherwise truncate a node's true degree. This
  backend additionally computes clustering coefficient and COINS street
  continuity, neither present in the other two backends.

**COINS** ("Continuity in Street Networks") is momepy's angle-based
street-continuity algorithm: it chains individual street segments into
longer continuous "strokes" whenever consecutive segments meet within an
angular deviation threshold (30 degrees here), approximating how a person
would perceive a street as "going straight" rather than turning at every
intersection. It is skipped for cells with more than 5,000 edges, or
entirely via `skip_coins=True`, since the stroke-building algorithm does
not scale linearly.

## Non-obvious design notes

- **City-specific UTM projections** (`projection.py`): every city is
  processed in its own locally-accurate UTM zone rather than one
  continental projection, looked up from `data/reference/projection_mapping.json`
  (a city-name-override table layered over a per-country default,
  covering all 54 countries and thousands of individual city overrides for
  cities that straddle a UTM zone boundary), falling back to
  ESRI:102022 (Africa Albers Equal Area Conic) only if a city cannot be
  resolved. This mirrors the companion grid pipeline's own per-city-UTM
  rationale.
- **3D metrics use two independently-sampled WSF3D rasters**: building
  height (`height_raster_path`) and volume (`volume_raster_path`) are
  sampled independently at each building's centroid (not as a polygon
  average), each with its own plausible-range clip and default-if-missing
  value. `VolumeDens`/`VolDe` derived from the volume raster is therefore
  **not** the same quantity as the grid's own inherited `V_mean`/`V_max`/
  `V_sum` columns (those come from the companion grid pipeline's separate,
  coarser raster sampling). Because sampling is centroid-based, small
  buildings near a raster cell boundary can be sensitive to exactly where
  their centroid falls.
- **"No buffer zone" is a deliberate, cross-cutting simplification**, not
  isolated to connectivity: `spatial_distribution_metrics.py` and
  `connectivity_igraph_primary.py` both analyze only buildings/nodes whose
  centroid falls within a cell's own boundary, rather than including
  neighbor-cell context. This is simpler, faster (no grid-adjacency
  computation), and consistent between the two modules.
- **Zero-copy worker pools**: both `spatial_distribution_metrics.py` and
  the metric-calculation passes in `morphology_metrics.py` pass large
  GeoDataFrames to a multiprocessing worker pool exactly once via a pool
  initializer, storing them as a module-level global in each worker
  process; individual tasks then pass only small index lists, avoiding
  per-task serialization of the full dataset.
- **Strict-mode batch processing** (`pipeline.py`): by default, a batch
  run halts immediately on the first city that fails for a reason other
  than a recognized, expected coverage gap (no Open Buildings data, AOI
  too small, no OSM highways after clipping), rather than silently
  skipping it. This trades convenience for a stronger guarantee: a batch
  run that finishes without halting is *known* to have produced a result
  for every requested city. Checkpointing (`checkpoint.py`) makes
  restarting after a fix resume from the last successful city rather than
  reprocessing everything.
- **Automatic country exclusion** (`trackers.py`): if a country
  accumulates 10 consecutive no-Open-Buildings-data cities counting from
  the *first* city processed in that country (and has never had a single
  successful city), it is excluded from the remainder of the run. A
  country that has had even one successful city is permanently exempt
  from this exclusion, since uneven coverage within one country (observed,
  for example, in Egypt) is common and should not stop processing the
  rest of that country.

## Data cleaning / QC stage

See `africa_morphology_metrics/cleaning/README.md` for the full 11-script
run order, with each script's purpose, inputs, outputs, and the data
quirk it addresses (e.g. a country-specific population-resampling issue
for Ethiopia, WSF3D no-coverage handling for desert/arid cities, bringing
cities that stalled at 114/121/165 columns up to the common 172-column
schema). The stage ends with `csv_to_shp.py`, which joins the cleaned CSV
attribute table onto grid geometry read from each city's working-output
directory (`{data_root}/{Country}/{City}/Output/*.shp` — the same
per-city geometry the main pipeline itself produces; this path is a
required CLI argument, not bundled data) to produce the final
`SHP_cleaned/` deliverable.

`data/reference/excluded_cities.md` documents 136 cities excluded from
building/plot-morphology-dependent analyses because more than 50% of
their grid cells have NaN values in those metrics (typically because the
city has very few mapped buildings relative to its grid extent). Cities
and columns unrelated to building/plot morphology are unaffected.

## Data layout

```
africa_morphology_metrics/   # package code (this repo)
geoworkflow/                  # vendored extraction backend (own MIT license)
data/
    reference/                 # small, checked-in reference/lookup data
        col_name_mapping.csv        # CSV column name -> Shapefile-safe (<=10 char) column name
        city_inventory.csv          # per-city cleaning status (cleaned/excluded + reason)
        quality_summary.csv         # per-city column-count tracking, read by cleaning/derive_165_column_metrics.py
        excluded_cities.md          # 136 cities excluded from building/plot-morphology analyses
        projection_mapping.json     # per-country/per-city UTM CRS lookup table
        studyarea/                  # 48-country study-area boundary (reference only)
    example/                   # real Ndioum input bundle and historical output
```

No bulk data is bundled. A full pipeline run reads/writes a large working
cache (per-city building extracts, OSM highway clips, checkpoint state)
under `data/{Country}/{City}/` at the repository root during execution —
this directory is not part of the published repository and is expected to
be gitignored/regenerated per run.

## Example output

`data/example/Ndioum_Senegal/` contains the real (not synthetic) input
bundle and historical cleaned output for Ndioum, Senegal. The one-command
two-project reproduction runs `africa_hex_grid_workflow` to rebuild the
49-column grid, then this project's metric calculators to produce the
172-column CSV and Shapefile. See `data/example/README.md` for input
provenance, the command and the observed SHA-256 match.

## Environment / setup

Tested with Python 3.10+ and the geospatial stack pinned in
`environment.yml` (geopandas, shapely, fiona, rasterio, momepy, networkx,
optional python-igraph) plus GCS/OSM-specific dependencies (pyogrio,
pyrosm, google-cloud-storage, gcsfs, s2sphere, pydantic). This mirrors and
cleans up the original pipeline's `requirements.txt`; entries specific to
the separate downstream deep-learning/clustering stack are not included
here, since that stage is out of scope for this repository.

## License

No license file was found in the source project this pipeline was derived
from. Until a license is chosen, all rights are reserved by default; add a
`LICENSE` file before distributing this code publicly. The vendored
`geoworkflow/` dependency is the one exception: it carries its own MIT
license (Copyright 2025 Cornell University — see `geoworkflow/LICENSE`),
distinct from the rest of this pipeline.
