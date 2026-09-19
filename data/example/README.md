# Example output — Ndioum, Senegal

This is a real (not synthetic) output of the pipeline for a single small
city: Ndioum, Senegal — a modest secondary town, chosen because its hex grid
is small enough (33 cells) to check into the repository as a schema
reference without adding bulk data.

Files:

- `Ndioum_grid_250m_hex_enriched_metrics_v15.csv` — the cleaned per-hexagon
  attribute table (no geometry), the tabular deliverable of the data-cleaning
  stage described under `africa_morphology_metrics/cleaning/`.
- `Ndioum_grid_250m_hex_enriched_metrics_v15.{shp,shx,dbf,prj,cpg}` — the
  same attributes joined onto hexagon geometry, the terminal SHP+CSV
  deliverable of the full pipeline (output of `cleaning/join_grid_geometry.py`
  or equivalently-named final step — see the cleaning stage README).

Column meanings are documented in the top-level README's output schema
section and in `data/reference/col_name_mapping.csv`.

## Reproducing this example, or generating one for another city

Run the full pipeline (`africa_morphology_metrics/pipeline.py`) for a single
city in `single_city` mode, then run the `cleaning/` script sequence over
its output. See the top-level README for the full stage list, data source
requirements, and setup instructions.
