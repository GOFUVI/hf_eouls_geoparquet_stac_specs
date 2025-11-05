# Dataset Inventory Template

This inventory centralizes the traceability requirements defined in the internal documentation update plan. The table below enumerates each external HF-EOLUS workflow, pre-populating the common fields requested for the first documentation milestone (workflow name, DOI references, STAC catalog root, and current partition layout). Additional columns for GeoParquet deliverables, critical columns, and STAC extensions are included as placeholders so subsequent tasks can enrich them without having to reformat the document.

- **hf_radial_metrics_aws_ingestion**
  - DOI references: workflow 10.5281/zenodo.17096796; dataset/STAC 10.5281/zenodo.16892223.
  - STAC catalog root: `hf_radial_metrics_aws_ingestion/hf_ingestion/catalog.json` (station sub-catalogs under `VILA/` and `PRIO/`, delivered as compressed archives on Zenodo).
  - Partition scheme: `pos_bragg={positive|negative}/timestamp=<ISO8601 slot>` (timestamps percent-encode `:`) so each Parquet shard retains the Bragg peak polarity and the original LLUV acquisition time.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

- **hf_eolus_sar_ingestion**
  - DOI references: workflow 10.5281/zenodo.17096926; dataset/STAC 10.5281/zenodo.17100125.
  - STAC catalog root: `hf_eolus_sar_ingestion/hf_eolus_sar/collection.json` (daily Items under `items/`, delivered as a compressed archive on Zenodo).
  - Partition scheme: `assets/date=YYYY-MM-DD/part-*.parquet`, aligned with the Athena partition column loaded via `MSCK REPAIR`.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

- **hf_eolus_pde_buoy_ingestion**
  - DOI references: workflow 10.5281/zenodo.17097949; dataset/STAC 10.5281/zenodo.17098038.
  - STAC catalog root: `hf_eolus_pde_buoy_ingestion/.stac/buoy_ingestion/collection.json` (published as a compressed archive on Zenodo).
  - Partition scheme: one GeoParquet snapshot per buoy, no directory partitions; the buoy identifier maps directly to the asset filename.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

- **hf_eolus_geo_tools**
  - DOI references: workflow 10.5281/zenodo.17104924; dataset/STAC 10.5281/zenodo.17115413.
  - STAC catalog roots: `hf_eolus_geo_tools/vila_catalog/catalog.json`, `hf_eolus_geo_tools/prio_catalog/catalog.json`, and `hf_eolus_geo_tools/sar_catalog/catalog.json` (each released as compressed archives on Zenodo).
  - Partition scheme: `pos_bragg={0|1}/1.parquet` for the HF-radar aggregates per station; SAR-driven aggregates are consolidated into single files under `assets/data.parquet`, inheriting any optional partition columns defined at finalization.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

- **hf-wind-inversion**
  - DOI references: workflow 10.5281/zenodo.17464519; dataset/STAC 10.5281/zenodo.17464583.
  - STAC catalog root: `hf-wind-inversion/catalogs/catalog.json` with sub-catalogs such as `vilano_pipeline/` and `grid_offset_pipeline/` (distributed as compressed bundles on Zenodo).
  - Partition scheme: consolidated GeoParquet assets stored as flat files (`assets/data.parquet`) keyed by `(timestamp, node_id, partition_label)`; cross-validation or domain splits are encoded in columns rather than folder paths.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

- **wind_interpolation**
  - DOI references: pending publication (see `AGENTS.md`).
  - STAC catalog root: `wind_interpolation/catalogs/catalog.json` (independent branch, e.g., `meteogalicia_interpolation/`, delivered as a compressed archive).
  - Partition scheme: `year=/month=/day=/hour=/` partitions under each S3/local prefix, accompanied by metadata sidecars that track the MeteoGalicia `source_model`; remains independent from the wind-resource workflow.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

- **wind_resource**
  - DOI references: pending publication (see `AGENTS.md`).
  - STAC catalog root: `wind_resource/catalogs/sar_range_final_power_estimates/catalog.json` (plus dependent inputs enumerated in `config/stac_catalogs.json`, released as compressed archives on Zenodo).
  - Partition scheme: public GeoParquet deliverables are single-file snapshots (`assets/power_estimates_nodes.parquet`) keyed by node identifiers and timestamps; no directory partitions because the dataset is node-dense yet time-aggregated.
  - GeoParquet deliverables: Pending.
  - Critical columns: Pending.
  - STAC extensions: Pending.

Future tasks tracked in the internal backlog will enrich the placeholder columns with schema descriptions, column highlights, and STAC extension usage notes. This template ensures the shared fields are already normalized across workflows, highlights that the `wind_interpolation` catalog remains a standalone branch, and reminds readers that all STAC/GeoParquet artifacts cited above are delivered as compressed archives (zip or tar.gz) on Zenodo due to storage and file-count limitations.
