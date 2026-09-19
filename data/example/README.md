# Real, end-to-end Ndioum reproduction

`Ndioum_Senegal/` contains a small **real** case for 33 hexagonal cells in
Ndioum, Senegal. It links the two published code projects:

1. `africa_hex_grid_workflow` builds the 250 m grid and its 49 input columns.
2. `africa_morphology_metrics` adds building, spatial, street and connectivity
   metrics, producing 172 attribute columns and a Shapefile.

The historical source run used
`TIF_Grid_workflow/Workflow_Xinghan/MyScripts/GridWorkflow_hexagon_v6_utm_dynamic.py`
for the grid and `Momepy_workflow/Metrix/run_building_analysis_v15_integrated_osm.py`
for morphology. The two published repositories reorganize those implementations.

## Bundled inputs and provenance

| Input | Source of this real-city subset | Used by |
| --- | --- | --- |
| `inputs/grid_source/Ndioum_aoi.geojson` | Original `Workflow_Xinghan/MyData/01_aoi/Senegal/Ndioum/` AOI, with AFRICAPOLIS identifiers | grid project |
| `inputs/grid_source/Senegal_boundary.geojson` | Senegal feature from the country boundary layer used by the source workflow | grid project country lookup |
| `inputs/grid_source/country_tif/Senegal/*.tif` | Pixel-window crops of all 14 Senegal country rasters in original `Workflow_Xinghan/MyData/00_country_TIF/Senegal/`: WSF3D height/volume, Copernicus land cover, WorldPop 2022 | grid project; height/volume also used by morphology |
| `inputs/grid/Ndioum_grid_250m_hex_enriched.*`, `Ndioum_grid_boundary.geojson` | Historical output of original `Workflow_Xinghan/MyData/03_country_grids/Senegal/Ndioum/` | offline morphology-only alternative |
| `inputs/buildings/open_buildings.*` | Original `Metrix/data/Senegal/Ndioum/Output/` Google Open Buildings extract | morphology project |
| `inputs/highways/Ndioum_highways.geojson` | Original `Metrix/data/Senegal/Ndioum/highways/` OSM extract | morphology project |
| `Ndioum_grid_250m_hex_enriched_metrics_v15.csv` and Shapefile | Historical cleaned result | comparison reference |

The raster crops retain source pixel values, geotransforms and NoData tags;
they were windowed around the city without resampling. These small inputs
make the metric calculation fully offline.

## Third-party data attribution

The input bundle contains clipped or extracted third-party data. Retain these
attributions when redistributing it:

- Building footprints: Google Open Buildings v3, used under CC BY 4.0.
  [Dataset and licence](https://sites.research.google/gr/open-buildings/).
- Roads: © OpenStreetMap contributors, available under ODbL 1.0.
  [Copyright and licence](https://www.openstreetmap.org/copyright).
- Building height and volume: DLR World Settlement Footprint 3D,
  CC BY 4.0. [Product page](https://web.geoservice.dlr.de/web/datasets/wsf_3d).
- Population raster: WorldPop, CC BY 4.0.
  [Redistribution FAQ](https://www.worldpop.org/faq/).
- Land-cover rasters: contains modified Copernicus Service information.
  [Global Land product licence](https://cds.climate.copernicus.eu/licences/Copernicus-Global-Land-product-licence).

The AOI, Senegal boundary and historical grid are retained from the source
research workflow. Their underlying source and redistribution rights should
be verified by the authors before making the bundle public.

Maintainers with the original data can recreate all 28 bundled input files
using `python -m examples.prepare_ndioum_inputs --help`. In the source
workspace, that command regenerated the bundle with identical SHA-256 hashes
for every file.

## Reproduce both projects

Install the environments described by **both** projects, then run from the
`africa_morphology_metrics` repository root using an environment containing
the dependencies of both projects:

```bash
python -m examples.reproduce_ndioum_full \
  --grid-project /path/to/africa_hex_grid_workflow \
  --output-dir outputs/exports/2026-09-19/ndioum_review
```

The script uses the bundled AOI, country boundary and 14 raster crops to run
the published grid project. It then runs the morphology code against that
newly generated grid, bundled building footprints and bundled roads. No
network download or original `Workflow_Xinghan`/`Metrix` code is invoked.
It writes `grid_data/03_country_grids/Senegal/Ndioum/`, plus
`morphology/Ndioum_grid_250m_hex_enriched_metrics_v15.{csv,shp,...}` and
`morphology/verification.json` under the chosen output directory. Use a
current date in the output path for a new run.

To run only the morphology stage against the bundled historical grid:

```bash
python -m examples.reproduce_ndioum \
  --output-dir outputs/exports/2026-09-19/ndioum_morphology_only
```

## Observed check (2026-09-19)

With Python 3.10.19, geopandas 1.1.1, rasterio 1.4.3, momepy 0.9.1,
libpysal 4.13.0 and igraph 1.0.0, the two-project run produced:

- 33 unique grid IDs; the rebuilt grid had the same 49 attribute columns
  and all 47 numeric input columns matched the historical grid.
- 33 output rows and 172 attribute columns; all 170 numeric output columns
  matched the historical CSV at `rtol=atol=1e-8`, including NaN positions.
- The output CSV was **byte-identical** to the historical reference:
  SHA-256 `44074aa687e26a2daffc705f0b5aaf521bfa57c12f8b2773e45ba2cdc3a61439`.
- The Shapefile has 33 geometries and 172 attributes. Its ten-character
  field limit renders `ThreeWayCnt` as `ThreeWayCn`; the CSV keeps the full name.

The verifier records SHA-256 hashes of bundled inputs and reports any
numeric column that differs. These checks demonstrate this city and this
environment; they do not establish byte-for-byte equivalence for every
African city or every dependency version. Ndioum reached 172 columns in the
main metric pass, so this example does not exercise the separate 114/121/165
column recovery scripts or Ethiopia's population correction.
