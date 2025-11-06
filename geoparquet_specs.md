# GeoParquet v1.1.0 Specification

This document outlines how HF-EOLUS applies the GeoParquet v1.1.0 standard across every workflow listed in the dataset inventory. It starts by summarising the mandatory requirements of the specification and then records, per dataset family, how those rules are implemented in practice so readers can navigate the deliverables with a consistent mental model.

## 1. GeoParquet fundamentals

### 1.1 Why GeoParquet for HF-EOLUS

HF-EOLUS handles millions of radar echoes, SAR wind vectors, buoy observations, aggregated grid nodes, and ANN inference rows. GeoParquet combines Apache Parquet’s columnar compression with an interoperable geospatial metadata model, allowing the project to store these high-volume assets efficiently, query them with cloud SQL engines, and move between Python, R, Spark, and DuckDB without bespoke import routines. Because the geometry payloads live alongside the numeric attributes, downstream teams can join spatial and temporal keys without reprojecting or reformatting the source files.

### 1.2 Core compliance requirements

All public and internal GeoParquet assets comply with the GeoParquet v1.1.0 specification:

- Geometry columns sit at the top level of the schema. Each file declares a single primary geometry column and may expose additional optional geometries when required (for example, the range-arc lines in the HF range diagnostics).
- Geometries are encoded as Well-Known Binary (WKB) produced by PyArrow/GeoArrow writers. The encoding is explicitly declared in file metadata so that readers do not need to infer it.
- Coordinate Reference Systems are embedded through PROJJSON definitions. Assets with explicit projections retain their CRS description; otherwise the CRS defaults to OGC:CRS84 (longitude, latitude).
- Every file embeds a `geo` metadata block describing the GeoParquet version, the name of the primary geometry column, the per-column encoding, CRS, and bounding box, and any optional fields (polygon orientation, edge interpretation, epoch) required by that dataset.
- The partition layouts exposed in the dataset inventory align with Hive-compatible folder keys so that Athena, DuckDB, and Spark can prune files without reading the Parquet footers.

### 1.3 Metadata pattern used in HF-EOLUS

Each GeoParquet file shares the same metadata structure even though the payloads vary:

- `version` is set to `"1.1.0"` to pin the exact GeoParquet release implemented by the writers.
- `primary_column` points to the geometry field leveraged for spatial filtering (e.g., `geometry` for point datasets or `range_geometry` for the range diagnostics). Downstream catalogs echo this value via the STAC Table extension.
- `columns.<name>.encoding` is always `"WKB"`; `columns.<name>.geometry_types` lists the concrete geometry classes (`Point`, `MultiLineString`, etc.) present in the file; `columns.<name>.crs` either references OGC:CRS84 or holds a PROJJSON object describing the CRS supplied by the producer.
- `columns.<name>.bbox` captures the file-level spatial extent to help clients that rely on metadata-driven spatial indexing.
- Optional keys `orientation` and `edges` appear whenever polygons or great-circle segments are present (for instance, SAR-derived polygons keep `orientation: counterclockwise` and `edges: spherical` so directional statistics remain unambiguous).

This shared metadata template means that any GeoParquet-compliant reader can infer the spatial semantics of the HF-EOLUS deliverables without looking up external documentation, yet the dataset-specific sections below still summarise how each workflow uses these rules.

## 2. Dataset families and GeoParquet profiles

The following subsections align with the workflows enumerated in the dataset inventory. They act as the bridge between the generic GeoParquet rules above and the concrete artefacts produced by each processing line, summarising schemas, geometry definitions, partition layouts, and CRS metadata captured in `dataset_inventory.md`.

### 2.1 HF-Radar radial metrics ingestion

HF-radar radial metrics are rewritten from CODAR LLUV spectra into three GeoParquet families that always declare GeoParquet v1.1.0 metadata and WKB encodings:

- `radial_metrics/pos_bragg={positive|negative}/timestamp=<ISO8601>/radial_metrics.parquet` holds every MUSIC echo with point geometries derived from `LOND/LATD`. The schema keeps the original LLUV columns (`Pwr`, `VELO`, `RNGE`, `BEAR`, quality flags) plus a derived `geometry` column exposed as the primary geometry.
- `header/.../header.parquet` stores the key/value metadata rows for each acquisition window. Every row inherits the site location as a CRS84 point so header provenance can be filtered spatially.
- `rng_info/.../rng_info.parquet` aggregates range-bin diagnostics (`SPRC`, `RNGC`, `NVAC`, `ALM1`–`ALM4`) and represents each arc as a `range_geometry` MultiLineString to capture the great-circle footprint of the ring being summarised.

**Geometry and CRS.** All three families embed `columns.<name>.crs` entries pointing to OGC:CRS84. `radial_metrics` and `header` set `primary_column=geometry` and restrict `geometry_types` to `Point`. `rng_info` switches `primary_column` to `range_geometry`, declares `geometry_types=["MultiLineString"]`, and populates `columns.range_geometry.edges="spherical"` so readers know the arcs follow geodesics. Bounding boxes are recorded at the shard level to accelerate metadata-driven filtering.

**Partitioning and metadata.** Folder keys follow `pos_bragg=<0|1>/timestamp=<ISO8601 slot>` (with RFC 3339 timestamps URL-encoded) so Athena and DuckDB can prune shards without scanning Parquet footers. The `pos_bragg` column is stored as a tinyint flag inside every file to keep partition provenance within the schema. Each shard reiterates the GeoParquet version (`"1.1.0"`) and retains Glue statistics so scans remain predicate-pushdown friendly.

**Schema highlights.** `timestamp` remains the temporal spine of all tables and is mirrored in STAC `datetime`. Echo tables expose the entire LLUV payload together with derived coordinates; header tables reduce to `metakey`, `metavalue`, `site_code`, `timestamp`, and `geometry`; `rng_info` concentrates on MUSIC-derived aggregates and includes the great-circle geometry. These consistent schemas allow downstream aggregations, QA dashboards, and STAC catalogs to reason about both positive and negative Bragg peaks without custom adapters.

### 2.2 Sentinel-1 SAR ingestion

The Sentinel-1 workflow produces one GeoParquet shard per observation day under `assets/date=YYYY-MM-DD/YYYY-MM-DD.parquet`; lineage and schema metadata are surfaced through STAC (Processing/Table extensions) rather than through extra files inside the dataset bundle.

**Geometry and CRS.** Each shard declares `primary_column=geometry`, with `geometry_types=["Point"]`, `encoding="WKB"`, and `crs` pointing to CRS84. Optional polygon footprints, when exported, reuse the same metadata block but add `orientation="counterclockwise"` and `edges="spherical"` so swath analyses retain directional context. Longitude and latitude columns stay in the schema for compatibility with analytics engines that prefer scalar coordinates.

**Partitioning and metadata.** The `date` Hive partition is materialised both in the folder path and as a STRING column to accelerate Athena repairs. `firstMeasurementTime` and `lastMeasurementTime` are persisted verbatim as TIMESTAMP(UTC) columns and echoed into the STAC items’ temporal extent. The `geo` metadata advertises the bounding box of each day, and the ingestion pipeline writes the `processing:lineage` strings directly into the Items without producing intermediate JSON sidecars.

**Schema highlights.** The core OWI measurements (`owiWindSpeed`, `owiWindDirection`, `owiMask`, `owiHeading`, `owiRadVel`, `owiWindQuality`, `owiInversionQuality`) remain DOUBLE/INT columns following the CTAS typing documented in the ingestion scripts. `rowid` is a deterministic BIGINT hash used as a foreign key by the aggregation toolkit. Every file keeps ESA quality bits untouched so downstream consumers can filter by reliability without revisiting the SAFE products.

### 2.3 Puertos del Estado buoy ingestion

Puertos del Estado time series are flattened into one GeoParquet snapshot per buoy (e.g., `assets/VILANO.parquet`). There are no directory partitions: each file is a self-contained chronology that STAC references directly.

**Geometry and CRS.** Every record carries a `geometry` column with the fixed buoy location encoded as a CRS84 point. The `geo` metadata therefore lists a single bounding box per file, `geometry_types=["Point"]`, and `primary_column=geometry`. Because the buoy coordinates never change, these values double as spatial provenance for the entire history.

**Schema highlights.** The schema keeps `timestamp` (TIMESTAMP UTC), `wind_speed` (DOUBLE), `wind_dir` (INT), maintenance quality flags when available, and derived aggregates used by ANN validation. Nullability mirrors the original PdE CSV conventions so missing/calm samples propagate cleanly. Athena CTAS SQL captured alongside the delivery guarantees that downstream re-ingestion can recreate the same typing.

**Metadata considerations.** Rewriting everything into single files allows the `geo` block to remain stable across releases, while `table:row_count` and `table:columns` in the STAC Table extension expose the schema to catalog clients. Citation metadata lives in STAC, so the GeoParquet files focus on CRS declarations and bounding boxes only.

### 2.4 HF-EOLUS geospatial aggregation toolkit

The aggregation toolkit emits several GeoParquet assets keyed by station and grid node:

- `vila_aggregated/pos_bragg={0|1}/0.parquet` and `prio_aggregated/...` contain half-hour summaries of HF echoes projected onto analysis nodes.
- `sar_aggregated/data.parquet` stores Sentinel-1-derived aggregates on the same mesh.
- `grid_nodes_*.parquet` alongside `geo_join_output*.parquet` capture the node definitions and the lookup between raw echoes and grid nodes.

**Geometry and CRS.** Aggregated tables always expose the node centroid as `geometry` (WKB Point, CRS84) and mark it as the primary geometry. Range diagnostics that rely on arcs reuse the same approach as the ingestion workflow by embedding `edges="spherical"`. Grid definition tables add auxiliary planar coordinates when required but keep `geometry` as the canonical spatial column.

**Partitioning and metadata.** HF aggregates stay partitioned by `pos_bragg={0|1}` so consumers can load positive and negative peaks independently. Each shard records the `node_id` domain, timestamp range, and bounding box inside the `geo` metadata so ANN pipelines can preflight extents without scanning the rows. SAR aggregates are single files with the same metadata layout, while node definition tables omit temporal fields because they are static assets.

**Schema highlights.** Common keys across all tables are `timestamp`, `node_id`, `pos_bragg` (when applicable), and `geometry`. HF-specific columns include the `n`, `pwr_*`, and `velo_*` statistic families plus distance/bearing descriptors required downstream. SAR aggregates contribute `owiwindspeed_*`, `owiwinddirection_*`, and sample-count columns. The join artifacts keep `echo_id`, `node_id`, and quality flags so analysts can trace every aggregate back to its raw echoes.

### 2.5 HF-Radar wind inversion toolkit

The wind inversion toolkit republishes the aggregated datasets as ANN-ready GeoParquet tables and captures every inference corpus inside the same standard metadata frame.

**Data families.** `catalogs/data_preparation_pipeline/` stores the pivoted HF/SAR/buoy datasets; `catalogs/*_pipeline/assets/data.parquet` bundles the trained-model inputs per experiment (baseline, KD, SAR-only, grid-offset); `catalogs/*_inference/assets/data.parquet` keeps the prediction outputs for each fold or station.

**Geometry and CRS.** Each file retains the grid `geometry` column from the aggregation toolkit, declared as a CRS84 point and marked as the primary geometry. Additional geometry columns (e.g., buffered footprints) are rare; when present they inherit the same CRS declaration. Bounding boxes reflect the union of all nodes covered by the file so inference consumers can preflight spatial coverage.

**Schema highlights.** Columns fall into four groups: (1) deterministic identifiers (`timestamp`, `node_id`, `fold_id`, `partition_label`), (2) aggregated HF/SAR features (`vila_aggregated__*`, `prio_aggregated__*`, SAR statistical families), (3) reference data (`pde_vilano_*`, maintenance intervals, site bearings), and (4) model outputs (`pred_wind_speed`, `pred_cos_wind_dir`, `prob_range_*`, confidence gates). Every table keeps explicit dtypes (DOUBLE for continuous metrics, INT/BIGINT for counters) and relies on the GeoParquet metadata to preserve CRS context for the node geometry.

**Metadata considerations.** Because the ANN corpora are stored as single shards per experiment, the `geo` block’s `bbox` and `row_count` give consumers enough information to detect station-specific subsets without reading all data. The absence of directory partitions keeps fold assignment inside the schema, simplifying reproducibility.

### 2.6 Wind resource toolkit

The resource toolkit exposes two public GeoParquet snapshots: the ANN inference corpus replicated from the wind-inversion repository and the derived power estimates.

**Geometry and CRS.** Both tables preserve the CRS84 point geometry per node and declare it as `primary_column=geometry`. The inference snapshot mirrors the metadata emitted by the ANN pipeline, while `power_estimates_nodes.parquet` recomputes the bounding box after summarising nodes so spatial tools can trim the dataset quickly.

**Schema highlights.** The inference file repeats every predictor from the ANN stage plus the prediction columns and QA flags (`pred_*`, `prob_range_*`, `range_flag`, `range_flag_confident`). The power estimates table introduces summarised metrics such as `method`, `power_density_w_m2`, `turbine_mean_power_kw`, `capacity_factor`, Kaplan–Meier (`km_shape`, `km_scale`) and Weibull (`weibull_k`, `weibull_lambda`) parameters, bootstrap spreads, empirical QA ratios, and Boolean quality gates. Companion `manifest.json` assets track checksums and `hf_eolus:code_commit` identifiers, but the GeoParquet file itself remains self-describing thanks to the `geo` block.

**Partitioning and metadata.** Public deliveries are single files on purpose so each version tag (e.g., `hf_eolus:version=sar-range-final-20251018`) maps to an immutable asset. Since the datasets can exceed one million rows, `row_group_size` is tuned to keep DuckDB and Athena efficient while still enabling columnar pruning. Bounding boxes and row counts are always set, ensuring catalog clients can estimate footprint and cost before reading the binaries.

### 2.7 MeteoGalicia wind interpolation (standalone branch)

The MeteoGalicia interpolation workflow remains an independent branch that mirrors HF-EOLUS conventions while keeping its catalog separate from the ANN and wind-resource lines.

**Partitioning and assets.** Hourly GeoParquet shards live under `year=/month=/day=/hour=/data.parquet`, each accompanied by JSON sidecars (`metadata.json`) and diagnostic PNGs. Additional collections ship the PdE buoy references used for validation, but none of these assets are linked via `derived_from` to the ANN or resource catalogs yet.

**Geometry and CRS.** Every shard stores node centroids in a `geometry` column tagged as CRS84 plus auxiliary planar coordinates (`x_local`, `y_local`) to encode the MeteoGalicia grid reference frame. The GeoParquet metadata lists `geometry_types=["Point"]`, and when great-circle segments are stored (e.g., interpolation footprints) they inherit the `edges="spherical"` flag.

**Schema highlights.** Mandatory columns include `timestamp`, `node_id`, `x_local`, `y_local`, `is_orig`, `source_model`, `interpolation_source`, the interpolated components (`u`, `v`, `wind_speed`, `wind_direction`, `u_rkt`, `v_rkt`), and diagnostics such as `kriging_var_u/v`, `nearest_distance_km`, `neighbors_used`, and the cross-validation/test statistics. These values are echoed in the per-hour metadata JSON but the GeoParquet schema remains the authoritative source.

**Independence notice.** Although the files comply with GeoParquet v1.1.0 and the Table extension like the rest of the project, their STAC catalog is intentionally standalone. Consumers must treat its versioning and DOI lineage independently until the interpolation outputs are formally wired into the ANN or wind-resource workflows.

## References

1. GeoParquet v1.1.0. https://geoparquet.org/releases/v1.1.0  
2. Efficient Storage and Querying of Geospatial Data with Parquet and DuckDB. https://waterprogramming.wordpress.com/2022/03/07/efficient-storage-and-querying-of-geospatial-data-with-parquet-and-duckdb
