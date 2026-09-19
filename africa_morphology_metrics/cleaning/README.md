# Cleaning stage

This package is the final QC/data-cleaning stage of the Africa urban
morphology metrics pipeline. It takes per-city hexagonal-grid (250m) metric
tables produced by the upstream grid-enrichment pipeline — each city
progressively enriched with land cover, WSF3D building height/volume,
population, building morphology, derived height metrics, and road-network
metrics — and turns them into a verified, uniform, 172-column cleaned
dataset, finally rejoining the cleaned attribute tables onto grid geometry
to produce `SHP_cleaned/`.

Across the ~8,393 cities in the study area, not every city's per-city
Output/ CSV reached the full 172-column schema by the same route: some
stalled at 114 columns (missing both height-derived and road-network
metrics), some at 121 (missing only road-network metrics), and some at 165
(missing only the 7 height-derived metrics, with road-network metrics
already present). The scripts in this directory bring every city up to the
common 172-column schema and verify the result, rather than assuming a
single uniform starting point.

All scripts take explicit CLI arguments for input/output directories —
none of them hardcode a path to a specific person's working data cache.
Point `--data-root`, `--cleaned-dir`, etc. at your own pipeline run's
output directories.

## Run order

First run `python -m africa_morphology_metrics.cleaning.quality_summary`
with `--data-root` and `--output` to produce the `quality_summary.csv`
required by step 4. This ports the original `Metrix/data_quality_check.py`
inventory calculation: column count plus the number of absent or entirely
zero/NaN raster indicators. It is a prerequisite audit, not a data mutation.

The dependency order below was verified by tracing each script's actual
file reads/writes (not just filename/mtime order). It matches the original
numbered listing, with one exception noted under script 6.

1. **`fix_ethiopia_pop2022.py`** — Recomputes Pop2022 for Ethiopian cities
   directly from the WorldPop 2022 raster, because Ethiopia's original
   population-resampling pass used a different code path than the rest of
   the continent and produced unreliable values. Reads each city's grid
   SHP + metrics CSV under `--data-root/Ethiopia/`, and a population TIF;
   overwrites only the Pop2022 column in place. Independent of the other
   scripts — can run at any point before the final QC pass, but is listed
   first since it is a targeted country-specific correction upstream of
   the general column-completion steps.

2. **`check_wsf3d_coverage.py`** — Diagnostic QC utility. Scans every
   city's H_max/H_mean/H_std/H_sum/V_sum/V_mean/V_max/V_std columns and
   flags cities where WSF3D building-height/volume coverage is entirely
   absent (`no_coverage`), partially absent (`partial`), or fine (`ok`).
   Produces a reference report used to decide which cities should be
   excluded for lacking height data (e.g. desert/arid regions with no
   settlement footprint in WSF3D). Reads per-city CSVs; writes a status
   report CSV. Does not modify any per-city data.

3. **`check_lc_lcf_shannon.py`** — Diagnostic QC utility, structurally
   identical in spirit to script 2 but for the land-cover side: flags
   cities where LC_Built, LCF_Built, or LC_Shannon are entirely zero/NaN,
   which indicates the land-cover classification had no usable signal for
   that city. Reads per-city CSVs; writes a status report CSV. Read-only
   with respect to per-city data.

4. **`derive_165_column_metrics.py`** (originally `fix_165col_derived_metrics.py`)
   — Computes the 7 height-derived metrics (FloorARat, Mean_FormF,
   Mean_H2F, Mean_VolDe, Std_FormF, Std_H2F, Std_VolDe) for cities whose
   enrichment output stalled at 165 columns, from building footprints and
   country height/volume rasters via momepy's FormFactor plus custom
   H2F/FloorArea/VolumeDensity derivations, aggregated to grid cells by
   spatial join. Reads a quality-summary CSV (`--quality-summary`, a
   durable per-run reference table of each city's current column count,
   not a throwaway diagnostic artifact) to select cities with
   `n_cols == 165`; updates each city's CSV in place; copies cities that
   pass a quality check into `--cleaned-dir`.

5. **`add_height_metrics.py`** (originally `fix_114col_add_height_metrics.py`)
   — The same 7-column height-derivation logic as script 4, applied to the
   larger set of cities that stalled at 114 columns (missing both
   height-derived and road-network metrics). Brings these cities to the
   121-column tier and updates `--city-inventory`'s `exclusion_reason`
   from `col_114` to `col_121`, recording that only road-network metrics
   remain outstanding. Road-network metrics are deliberately deferred to a
   separate step (script 8) since they depend on OSM data that is fetched
   independently of the raster/vector layers used here.

6. **`check_col121_cities.py`** — Diagnostic QC utility verifying the
   col_121 tier (cities missing only the 51 road-network columns, i.e. the
   union of the outputs of scripts 4 and 5) before road metrics are added:
   confirms each city is missing *exactly* the expected 51 columns and
   that none of the present WSF3D/height-derived/open-buildings columns
   are degenerate. **Note on execution order**: this script reads one
   already-complete 172-column city's CSV header as its reference column
   list (`--reference-csv`); that reference city must therefore already be
   fully cleaned, meaning in practice this check is only meaningful to run
   after at least one full pipeline pass (including road metrics) has
   completed for some city. It is placed at position 6 in this listing to
   match its logical role (checking the col_121 tier before road metrics
   are added generally), but has an implicit soft dependency on the
   existence of at least one already-172-column city.

7. **`verify_cleaned_output.py`** (originally `verify_cleaned_7787.py`,
   renamed to drop the specific-run city count from the filename) —
   Coarse pass/fail QC sweep over the entire cleaned-output directory:
   column count, non-degenerate TIF/height/core-road columns, GridID
   uniqueness. This is the first gate a city must pass once it reaches the
   cleaned-output directory, run after scripts 4/5 have populated
   `--cleaned-dir` and before road-network metrics are added, and again
   after road-network metrics are added (script 8 runs this same class of
   check internally as part of its own QC gate).

8. **`add_road_network_metrics.py`** (originally `fix_road_network_metrics.py`)
   — Adds the 51 road-network/connectivity columns to col_121 cities:
   downloads/clips OSM highway data per country, computes street-network
   and connectivity metrics, merges them into each city's CSV, runs a
   quality check, copies passing cities into `--cleaned-dir`, and updates
   `--city-inventory` (status/exclusion_reason/n_cols). This is the step
   that finally brings a city to the full 172-column schema. It depends on
   upstream pipeline modules (street-network and connectivity metric
   calculators, OSM highway processor, projection helper) from the bundled
   `africa_morphology_metrics` package. Run it with
   `python -m africa_morphology_metrics.cleaning.add_road_network_metrics`
   from the repository root (or install the package first); provide
   `--country-boundaries-file` for the bundled OSM processor. A fixed list
   of cities with pre-existing WSF3D/land-cover issues (`KNOWN_BAD`) still
   get road metrics computed but are excluded from the cleaned-output
   directory regardless.

9. **`full_qc_cleaned.py`** — The most exhaustive QC audit in the
   pipeline, run once the cleaned-output directory is believed complete
   (after script 8). Checks structural integrity (column count/names,
   GridID uniqueness), city-level non-degeneracy per column group,
   per-cell range validity (e.g. CoverRatio in [0,1], LC_Shannon >= 0,
   coordinates within the African bounding box), internal consistency
   (e.g. zero-building cells should have zero building-morphology
   columns), and road-network-specific rules. Produces a summary, a
   detailed issues CSV tagged by severity, and per-column summary
   statistics across the whole cleaned dataset. Read-only with respect to
   per-city data.

10. **`generate_col_mapping.py`** — Generates the CSV-column ->
    SHP-field-name mapping table used by script 11, derived
    programmatically from the canonical 172-column list under the rule
    that ESRI Shapefile field names are capped at 10 characters (only
    `ThreeWayCnt` needs truncation, to `ThreeWayCn`). This is a
    code-generation utility for a small reference table; the generated
    table itself is checked in at `data/reference/col_name_mapping.csv`
    so this script does not need to be re-run for a standard pipeline
    pass — it exists for reproducibility and in case the 172-column schema
    ever changes.

11. **`csv_to_shp.py`** — The final step. Joins each cleaned city's CSV
    attributes back onto grid geometry (by GridID) and writes the result
    as a Shapefile to `SHP_cleaned/`. Geometry is deliberately not carried
    through the earlier CSV-only cleaning steps (re-reading/writing
    geometry at every step would be wasteful, since it never changes
    during cleaning); this script re-attaches it exactly once, at the end.
    **External input requirement**: each city's source grid geometry is
    read from `<source-shp-root>/<country>/<city>/Output/<city><suffix>.shp`
    — the same per-city Output/ directory tree produced by the upstream
    (pre-cleaning) grid-enrichment pipeline that scripts 1, 4, 5, and 8
    also read from. This directory is a large, per-run bulk-data cache and
    is intentionally **not** part of this repository; `--source-shp-root`
    must be pointed at that directory from your own pipeline run. Supports
    multiprocessing (`--workers`) and resumes automatically by skipping
    cities whose output SHP already exists.

## Column-count tiers referenced above

| Tier | Missing relative to 172 | Produced by |
|------|--------------------------|-------------|
| 114  | 7 height-derived + 51 road-network | upstream grid-enrichment run |
| 121  | 51 road-network | script 5 (`add_height_metrics.py`), or upstream run |
| 165  | 7 height-derived | upstream grid-enrichment run |
| 172  | none (complete) | script 8 (`add_road_network_metrics.py`) |

## Reference/data files

Small, checked-in reference data used by these scripts lives in
`data/reference/` (relative to the repository root), not in this
directory:

- `col_name_mapping.csv` — CSV column -> SHP field name mapping (generated
  by script 10).
- `city_inventory.csv` — per-city tracking table (country, city, column
  count, cleaning status, exclusion reason/note) updated in place by
  scripts 5 and 8.
- `excluded_cities.md` — cities with a high (>50%) NaN drop rate in
  building/plot morphology metrics, relevant to any downstream use of
  `SHP_cleaned/` that depends on dense building-morphology coverage.
- `studyarea/study_area_48countries.shp` (+ .shx/.dbf/.prj/.cpg) — the
  48-country study-area boundary shapefile.

## Notes on run-artifact vs. reference data

Per the porting scope for this stage, ad-hoc diagnostic outputs
(`quality_summary_3D_error.csv`, `quality_summary_LCLCFbuiltShannon_error.csv`,
`quality_summary_172_error.csv`, `full_qc_colstats.csv`, `full_qc_issues.csv`,
`full_qc_summary.txt`, `verify_cleaned_summary.txt`, `verify_cleaned_issues.csv`,
`check_col121_issues.csv`, `no_iso3_mapping.csv`) are pure outputs of the
check/verify scripts and are not checked in or read back in by any script
in this pipeline.

One file initially suspected of being a run artifact turned out to be
durable reference data read as an **input**: `quality_summary.csv` (no
suffix) is read by script 4 (`derive_165_column_metrics.py`, via
`--quality-summary`) to select which cities have 165 columns — it is a
per-run city/column-count inventory, distinct from the `*_error.csv`
diagnostic outputs listed above, and is expected to exist before running
script 4. `city_inventory.csv` is similarly durable state: it is both read
and updated in place by scripts 5 and 8, and is checked in under
`data/reference/`.
