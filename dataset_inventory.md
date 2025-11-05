# Dataset Inventory Template

This inventory centralizes the traceability requirements defined in the internal documentation update plan. The table below enumerates each external HF-EOLUS workflow, pre-populating the common fields requested for the first documentation milestone (workflow name, DOI references, STAC catalog root, and current partition layout). Additional columns for GeoParquet deliverables, critical columns, and STAC extensions are included as placeholders so subsequent tasks can enrich them without having to reformat the document.

- **hf_radial_metrics_aws_ingestion**
  - DOI references: workflow 10.5281/zenodo.17096796; dataset/STAC 10.5281/zenodo.16892223.
  - STAC catalog root: `hf_radial_metrics_aws_ingestion/hf_ingestion/catalog.json` (station sub-catalogs under `VILA/` and `PRIO/`, delivered as compressed archives on Zenodo).
  - Partition scheme: `pos_bragg={positive|negative}/timestamp=<ISO8601 slot>` (timestamps percent-encode `:`) so each Parquet shard retains the Bragg peak polarity and the original LLUV acquisition time.
  - GeoParquet deliverables:
    - `radial_metrics/` — per-echo GeoParquet tables with point geometries derived from LLUV `LOND/LATD`, split by Bragg peak polarity and timestamp; files embed Glue/Athena statistics and are the primary inputs to all downstream analytics.
    - `header/` — key/value metadata rows from every LLUV header, each stamped with the acquisition time and site code plus a point geometry at the radar location so header provenance can be spatially filtered.
    - `rng_info/` — range-bin diagnostics where each row aggregates MUSIC statistics for a radial ring and stores its great-circle arc as a multiline geometry, enabling QC of noise-floor and Bragg-window parameters by distance.
  - Critical columns:
    - `timestamp` (UTC) drives partition folders and STAC `datetime`; required by Athena for temporal filtering.
    - `pos_bragg` indicates positive (`1`) or negative (`0`) Bragg peak; governs partition pruning and ties every Item to the correct asset.
    - Echo-level metrics kept by default: `Pwr`, `VELO`, `RNGE`, `BEAR`, `LOND`, `LATD`, plus derived `geometry` (WKB). Header tables expose `metakey/metavalue`, while `rng_info` highlights `SPRC`, `RNGC`, `NVAC`, and Bragg window columns (`ALM1`…`ALM4`).
  - STAC extensions: Collections and Items declare the **Table Extension v1.2.0** to publish `table:columns`, `table:primary_geometry`, and `table:row_count`; they also expose custom `hf_site:*` and `radial_metrics:*` properties plus `sci:doi`/`sci:citation` fields for citation traceability, mirroring the JSON bundled under `hf_ingestion/`.

- **hf_eolus_sar_ingestion**
  - DOI references: workflow 10.5281/zenodo.17096926; dataset/STAC 10.5281/zenodo.17100125.
  - STAC catalog root: `hf_eolus_sar_ingestion/hf_eolus_sar/collection.json` (daily Items under `items/`, delivered as a compressed archive on Zenodo).
  - Partition scheme: `assets/date=YYYY-MM-DD/part-*.parquet`, aligned with the Athena partition column loaded via `MSCK REPAIR`.
  - GeoParquet deliverables:
    - `assets/date=YYYY-MM-DD/YYYY-MM-DD.parquet` — one GeoParquet shard per observation day, storing every OWI vector as a CRS84 point with deterministic `rowid` and embedded GeoParquet metadata (WKB encoding, polygon orientation counterclockwise, spherical edges). Files inherit Glue/Athena stats via PyArrow’s dataset writer so downstream SQL engines can prune by `date`.
    - `lineage.json` — maps each `date` partition to the Sentinel-1 OCN SAFE archives processed that day, enabling the Processing extension lineage strings to be regenerated.
    - `columns.sql` — Athena DDL snippet emitted alongside the dataset so the external table can be recreated with the exact column typing used during CTAS (timestamp/radial components stored as DOUBLE, `rowid` coerced to BIGINT).
  - Critical columns:
    - `firstMeasurementTime` / `lastMeasurementTime` define the Item `datetime` bounds and feed Athena predicates; they remain TIMESTAMP(UTC) columns inside every Parquet shard.
    - `owiLon`, `owiLat`, and `geometry` (WKB Point, CRS84) power spatial filtering and constitute the declared GeoParquet primary geometry; lon/lat are kept explicitly for analytical joins even though geometry exists.
    - `owiWindSpeed`, `owiWindDirection`, `owiMask`, `owiInversionQuality`, `owiHeading`, `owiWindQuality`, and `owiRadVel` encapsulate the physical measurements plus ESA quality flags; analytics and ANN training consume them directly.
    - `rowid` (BIGINT) is a deterministic hash of time/lon/lat/speed, used as a stable foreign key into aggregated datasets.
    - `date` (STRING) is the Hive/Athena partition column, mirroring the `assets/date=...` folder layout and the STAC Item identifier.
  - STAC extensions:
    - Collections publish the **Table Extension v1.2.0** (`table:tables`, `table:row_count`, and `table:primary_geometry=geometry`) and reuse the Scientific metadata block (`sci:doi`, `sci:citation`, `providers`) injected via `stac_properties_collection.json`.
    - Items enable both the **Table Extension v1.2.0** and the **Processing Extension v1.1.0**; the latter carries `processing:lineage` strings built from `lineage.json`, while the former lists the OWI schema so STAC clients can introspect Parquet columns without reading the files.

- **hf_eolus_pde_buoy_ingestion**
  - DOI references: workflow 10.5281/zenodo.17097949; dataset/STAC 10.5281/zenodo.17098038.
  - STAC catalog root: `hf_eolus_pde_buoy_ingestion/.stac/buoy_ingestion/collection.json` (published as a compressed archive on Zenodo).
  - Partition scheme: one GeoParquet snapshot per buoy, no directory partitions; the buoy identifier maps directly to the asset filename.
  - GeoParquet deliverables:
    - `assets/<buoy>.parquet` — consolidated hourly wind time series per buoy (e.g., `Vilano.parquet`), produced via Athena CTAS and rewritten with embedded GeoParquet metadata (`geo` block set to CRS84, WKB Point at the buoy location). Because there are no folder partitions, each file is a complete chronology for that buoy and is the asset exposed in STAC.
    - `ctas.sql` (captured per run) and the ingestion log enumerate the Athena SQL used to recreate the Parquet snapshot, providing provenance for the exact column coercions applied (`wind_speed_str`→DOUBLE, `wind_direction_str`→INT).
  - Critical columns:
    - `timestamp` (`TIMESTAMP`) is the sole temporal dimension; it is parsed from the PdE CSV strings and becomes the STAC `datetime`, enabling hourly slicing without repartitioning.
    - `wind_speed` (`DOUBLE`) and `wind_dir` (`INT`) are the physical measurements consumed by downstream aggregation/ANN workflows; they remain nullable because the source CSV marks calm/invalid samples as blanks.
    - `geometry` (WKB Point) anchors every record to the fixed buoy location and is declared as the GeoParquet primary geometry to keep the dataset spatially discoverable despite its pointwise nature.
  - STAC extensions:
    - The Collection declares the **Table Extension v1.2.0** and exposes `table:columns` describing (`timestamp`, `wind_speed`, `wind_dir`, `geometry`); Items inherit the schema implicitly and stay lean because each asset already contains a single buoy/site.
    - Scientific citation metadata (`sci:doi`, `sci:citation`, `providers`) is injected into both collection and items via the provided properties JSON, so clients consuming the **Scientific Extension** fields can trace the PdE source without inspecting the README.

- **hf_eolus_geo_tools**
  - DOI references: workflow 10.5281/zenodo.17104924; dataset/STAC 10.5281/zenodo.17115413.
  - STAC catalog roots: `hf_eolus_geo_tools/vila_catalog/catalog.json`, `hf_eolus_geo_tools/prio_catalog/catalog.json`, and `hf_eolus_geo_tools/sar_catalog/catalog.json` (each released as compressed archives on Zenodo).
  - Partition scheme: `pos_bragg={0|1}/1.parquet` for the HF-radar aggregates per station; SAR-driven aggregates are consolidated into single files under `assets/data.parquet`, inheriting any optional partition columns defined at finalization.
  - GeoParquet deliverables:
    - `vila_aggregated/pos_bragg={0|1}/0.parquet` and `prio_aggregated/pos_bragg={0|1}/0.parquet` — half-hour aggregates per grid `node_id`, generated after mapping raw echoes to grid nodes and projecting VELO toward a reference point; each partition is compressed into a single GeoParquet shard.
    - `sar_aggregated/data.parquet` — Sentinel-1 OWI wind statistics mapped to the same grid and consolidated (no directory partitions) for ingestion into the ANN workflow.
    - `grid_nodes_*.parquet` snapshots and the `geo_join_output*` mapping tables — node definitions (point geometry per node) plus row-to-node lookups kept as GeoParquet to rerun aggregation or regenerate STAC without recreating the grid.
  - Critical columns:
    - Shared keys: `timestamp`, `node_id`, and `geometry` (WKB/CRS84) appear in every aggregate and function as STAC `table:primary_geometry`.
    - HF-radar stats: `n`, `pwr_*` (mean/median/stddev/min/max/mad/n), and `velo_*` summaries for the VELO projection; `pos_bragg` remains the partition discriminator and is encoded both in folder names and STAC hierarchy.
    - SAR stats: `owiwindspeed_*` and `owiwinddirection_*` families produced by the directional wrapper; magnitude-weighted direction fields are carried alongside sample counts to preserve QC context.
  - STAC extensions: All Collections/Items depend on the **Table Extension v1.2.0** to advertise schema metadata, and reuse the `sci:*` citation block plus license/provider metadata defined in `stac_properties_collection_{hf,sar}.json`; HF catalogs additionally keep the custom `hf_site:*` namespace for station context.

- **hf-wind-inversion**
  - DOI references: workflow 10.5281/zenodo.17464519; dataset/STAC 10.5281/zenodo.17464583.
  - STAC catalog root: `hf-wind-inversion/catalogs/catalog.json` with sub-catalogs such as `vilano_pipeline/` and `grid_offset_pipeline/` (distributed as compressed bundles on Zenodo).
  - Partition scheme: consolidated GeoParquet assets stored as flat files (`assets/data.parquet`) keyed by `(timestamp, node_id, partition_label)`; cross-validation or domain splits are encoded in columns rather than folder paths.
  - GeoParquet deliverables:
    - `catalogs/data_preparation_pipeline/` contains the canonical pivoted HF-radar aggregates (`vila_pivot*.parquet`, `prio_pivot*.parquet`, `pivots_joined/assets/data.parquet`) plus the stratified SAR and buoy partitions (`pivots_sar_valid_{train,test}`, `pivots_sar_buoy_wind_*`, `vilano_buoy_training_*`, both 10 m-adjusted and native heights). Every table is finalised as a single GeoParquet file with deterministic `crc32_partition_key` and `fold` columns so the Athena views used for ANN training can be reconstructed without rerunning the splitters.
    - `catalogs/grid_offset_pipeline/` packages the offset-grid experiments: displaced pivot tables (`*_pivot_offset*`), intermediate joins (`pivots_offset_*`, `pivots_offset_sar_valid_source`), and the downstream inference snapshots (`grid_offset_*_inference`) that quantify how well the ANN transfers when the mesh is shifted away from the original SAR alignment.
    - `catalogs/{vilano_pipeline,sar_pipeline,sar_vilano_pipeline}/` consolidate the production Vilano inference runs and the SAR-only experiments. Each collection publishes `assets/data.parquet` with predictions, range classifiers, probabilistic scores, maintenance indicators, and scenario labels (baseline, fine-tuned, L2-SP, KD, combined, buoy-reference) so consumers can compare operating modes without touching intermediate CSVs. All archives are zipped for publication on Zenodo.
  - Critical columns:
    - Common keys: `timestamp`, `node_id`, and `geometry` (WKB/CRS84) plus the Bragg-peak pivots `vila_aggregated__*` / `prio_aggregated__*`, station geometry descriptors (`*_bearing`, `*_dist_km`), and maintenance metadata (`*_maintenance_interval_id`, `*_maintenance_type`, `*_maintenance_start`, `*_hours_since_last_calibration`).
    - Partition metadata: `crc32_partition_key`, `fold`, `partition_name`, `domain`, `source_partition`, and `cv_seed` appear in every training-ready table so cross-validation folds and test holds can be reproduced byte-for-byte.
    - Supervision targets: aggregated SAR fields (`sar_mean_wind_speed`, `sar_mean_wind_direction`, mask/quality columns), buoy references (`vilano_buoy_wind_speed_[10m|ref]`, `vilano_buoy_cos_dir`, `vilano_buoy_flag`), and derived target labels (`target_range_label`, `target_speed`, `target_dir`) that drive the regression and range heads.
    - Model outputs and QA diagnostics: `pred_wind_speed`, `pred_wind_direction`, `pred_cos_wind_dir`, `pred_sin_wind_dir`, `prob_range_{below,in,above}`, `pred_range_label`, `pred_range_confidence`, `range_flag[_confident]`, `range_near_{lower,upper,any}_margin`, `range_prediction_consistent`, and `domain_shift_label` document the ANN behaviour across every inference snapshot.
  - STAC extensions: All collections declare the STAC **Table Extension v1.2.0** (with `table:tables`, exhaustive `table:columns`, and `table:row_count`), reuse `sci:doi` / `sci:citation` blocks from the DOIs above, and expose detailed `providers` metadata (INTECMAR, ESA, PdE, GOFUV). Inference catalogs share identical schemas so Items can be compared across scenarios, and every Item keeps the GeoParquet asset co-located with the JSON to ease downstream syncing.

- **wind_interpolation**
  - DOI references: pending publication (see `AGENTS.md`).
  - STAC catalog root: `wind_interpolation/catalogs/catalog.json` (independent branch, e.g., `meteogalicia_interpolation/`, delivered as a compressed archive).
  - Partition scheme: `year=/month=/day=/hour=/` partitions under each S3/local prefix, accompanied by metadata sidecars that track the MeteoGalicia `source_model`; remains independent from the wind-resource workflow.
  - GeoParquet deliverables:
    - The MeteoGalicia workflow syncs hourly GeoParquet shards to `local_sync/year=/month=/day=/hour=/data.parquet`, with matching `metadata.json` and PNG diagnostics stored under `local_sync_metadata/` and `local_sync_plots/`. `case_study/catalogs/meteogalicia_interpolation/` republishes those folders as three sibling collections (`items/parquet`, `items/metadata`, `items/plots`) so each hour exposes the data asset, the JSON sidecar, and the cross-validation plots via STAC links.
    - `case_study/catalogs/pde_vilano_buoy/` keeps the Vilano buoy GeoParquet time series alongside the interpolation results to simplify validation against in-situ observations.
    - All catalog trees remain standalone and are zipped before uploading to Zenodo so the interpolation branch can evolve independently from the ANN/resource repositories.
  - Critical columns:
    - Spatial descriptors (`x_local`, `y_local`, `is_orig`, `node_id`, `geometry`) identify whether a node belongs to the original WRF grid or to the densified mesh and allow spatial filtering either in projected metres or WGS84.
    - Interpolated winds: `u`, `v`, `wind_speed`, `wind_direction`, `u_rkt`, `v_rkt`, `interpolation_source`, and `source_model` record the chosen MeteoGalicia domain (1 km/1.3 km/4 km) plus whether the value is native or interpolated.
    - Quality metrics: `kriging_var_u/v`, `nearest_distance_km`, `neighbors_used`, `cv_model_{u,v}`, `cv_rsr_{u,v}`, `cv_bias_{u,v}`, `test_model_{u,v}`, `test_rsr_{u,v}`, `test_bias_{u,v}`, and the hold-out cross-validation summaries that power the hourly `metadata.json` sidecars.
  - STAC extensions: Collections and Items rely on the STAC **Table Extension v1.2.0** (per-hour `table:columns`, row counts, and `table:primary_geometry`). Items also expose `properties.source_model` and the collection aggregates the distinct values under `extra_fields.source_models`, mirroring the provenance recorded in every GeoParquet partition. Each Item links to its metadata and plot assets via `describedby` / `related` relationships, while the buoy catalog inherits the `sci:doi` / `sci:citation` references from the PdE ingestion repository. This entire catalog tree is explicitly documented as a self-contained branch that does not yet feed the wind-resource chain.

- **wind_resource**
  - DOI references: pending publication (see `AGENTS.md`).
  - STAC catalog root: `wind_resource/catalogs/sar_range_final_power_estimates/catalog.json` (plus dependent inputs enumerated in `config/stac_catalogs.json`, released as compressed archives on Zenodo).
  - Partition scheme: public GeoParquet deliverables are single-file snapshots (`assets/power_estimates_nodes.parquet`) keyed by node identifiers and timestamps; no directory partitions because the dataset is node-dense yet time-aggregated.
  - GeoParquet deliverables:
    - `catalogs/sar_range_final_pivots_joined/assets/data.parquet` republishes the ANN inference corpus used as the upstream resource input (≈1.16 million rows, 2011–2023). It preserves every pivoted HF-radar feature, SAR aggregation, buoy context, and model prediction so resource metrics can be regenerated without rerunning the ANN.
    - `catalogs/sar_range_final_power_estimates/assets/power_estimates_nodes.parquet` intersects the ANN outputs with empirical QA summaries and power-curve computations to produce one record per HF node together with the processing `manifest.json`. The collection is version-tagged (e.g., `hf_eolus:version=sar-range-final-20251018`) so each delivery can be traced back to the exact code commit and ANN snapshot.
    - `catalogs/pde_vilano_buoy/` (mirroring the PdE DOI) ships alongside the toolkit to keep the ground-truth reference used during validation within the same archive.
  - Critical columns:
    - ANN inference table: `timestamp`, `node_id`, `geometry`, `vila_aggregated__*` / `prio_aggregated__*` statistics for both Bragg peaks, station distance/bearing descriptors, PdE buoy joins, SAR supervision columns, and maintenance intervals. Prediction outputs cover `pred_wind_speed`, `pred_wind_direction`, `pred_cos_wind_dir`, `pred_sin_wind_dir`, `prob_range_{below,in,above}`, `pred_range_label`, `pred_range_confidence`, `range_flag`, `range_flag_confident`, `range_near_{lower,upper,any}_margin`, and consistency flags used by the resource estimators.
    - Resource summary table: `method`, `power_density_method`, `air_density`, `power_density_w_m2`, `turbine_mean_power_kw`, `capacity_factor`, `availability_ratio`, `curtailed_ratio`, `km_shape`/`km_scale` (Kaplan–Meier), `weibull_k`/`weibull_lambda`, bootstrap spread statistics (`bootstrap_mean_speed`, `bootstrap_std_speed`, `bootstrap_ci_lower/upper`), empirical QA ratios (`in_ratio`, `below_ratio`, `uncertain_ratio`), and Boolean quality gates (`sample_bias`, `coverage_bias`, `censoring_bias`, `low_coverage`, `reliable_estimate`). `manifest.json` captures file hashes and `hf_eolus:code_commit` for provenance.
  - STAC extensions: Collections and Items depend on the STAC **Table Extension v1.2.0** (full schema advertised via `table:columns`, row counts, and `table:primary_geometry`). The power-estimate Item links back to the ANN dataset through `derived_from`, carries the custom `hf_eolus:*` version metadata, and publishes the provenance manifest as a secondary asset. Both collections retain the Zenodo `sci:doi`/`sci:citation` blocks and reuse the provider declarations (GOFUV, PdE, HF-EOLUS project) so publication metadata stays aligned with the upstream catalogs.

Future tasks tracked in the internal backlog will enrich the placeholder columns with schema descriptions, column highlights, and STAC extension usage notes. This template ensures the shared fields are already normalized across workflows, highlights that the `wind_interpolation` catalog remains a standalone branch, and reminds readers that all STAC/GeoParquet artifacts cited above are delivered as compressed archives (zip or tar.gz) on Zenodo due to storage and file-count limitations.
