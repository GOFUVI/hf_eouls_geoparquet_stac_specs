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

This workflow converts CODAR LLUV spectra into partitioned GeoParquet archives (`radial_metrics`, `header`, `rng_info`). The data is split by station, Bragg polarity, and acquisition timestamp, and every shard stores CRS84 point or multiline geometries so ingestion metadata can be joined spatially downstream. The subsection will capture how the per-asset schemas, geometry types, and `geo` metadata blocks keep the positive and negative Bragg peaks aligned with the STAC catalog for `VILA` and `PRIO`.

### 2.2 Sentinel-1 SAR ingestion

Daily GeoParquet partitions list every OWI vector captured by the Sentinel-1 OCN products. Each file contains CRS84 point geometries combined with deterministic `rowid` identifiers and inherits polygon-orientation metadata for the optional footprint geometries. Here we describe how the SAR workflow encodes its per-day partition folders, lineage sidecars, and Table-extension compatible `geo` metadata so the aggregates can be consumed by DuckDB, Athena, and the ANN training stack.

### 2.3 Puertos del Estado buoy ingestion

Buoy deliverables consolidate entire station chronologies into single GeoParquet snapshots per buoy, each with a fixed CRS84 point geometry. The absence of directory partitions is intentional to keep every buoy asset a self-contained time series; schema consistency is documented through the file-level `geo` block and the Table metadata broadcast through STAC. This section outlines how the ingestion pipeline rewrites the Athena CTAS outputs, applies GeoParquet metadata, and exposes the assets downstream.

### 2.4 HF-EOLUS geospatial aggregation toolkit

The aggregation toolkit emits GeoParquet tables per station and Bragg polarity after snapping raw echoes to the analysis grid, plus consolidated SAR aggregates mapped onto the same mesh. It also stores node-definition tables and join artifacts. All of these tables share CRS84 point geometries keyed by `node_id`, while the HF range statistics inject partition keys for Bragg polarity. This subsection documents how the toolkit maintains GeoParquet compliance across the aggregated outputs that feed ANN preparation.

### 2.5 HF-Radar wind inversion toolkit

Pivoted datasets, ANN-ready folds, and inference corpora are stored as flat GeoParquet files where spatial context (node geometries, station bearings, maintenance metadata) is preserved alongside model targets and predictions. Even though these assets live in different catalog branches (baseline, grid-offset, SAR-only experiments), they inherit the same GeoParquet metadata template so metrics can be compared across experiments. The subsection highlights how the toolkit encodes geometry fields, fold metadata, and partition hashes while keeping the files compatible with Table-extension aware clients.

### 2.6 Wind resource toolkit

Resource deliverables package the ANN inference corpus and the derived power summaries into two GeoParquet snapshots. Both tables expose CRS84 point geometries and embed the provenance metadata that ties every record back to the ANN partitions and corresponding manifest files. This part of the document will describe how the toolkit serialises Kaplan–Meier outputs, Weibull parameters, and QA flags without violating the GeoParquet rules governing geometry placement and CRS declarations.

### 2.7 MeteoGalicia wind interpolation (standalone branch)

The interpolation workflow produces hourly GeoParquet partitions (`year=/month=/day=/hour=/`) containing MeteoGalicia-derived node values plus diagnostic metadata. Each partition carries CRS84 point geometries, interpolation quality metrics, and references to the MeteoGalicia source model. Unlike the other workflows, this branch remains independent: its GeoParquet archives feed analyses within the interpolation toolkit only, and no `derived_from` links connect them to the ANN or wind-resource assets yet. This subsection captures that independence so readers understand that, while the files honour the same GeoParquet conventions, their catalogue stands apart until formal integration occurs.

## References

1. GeoParquet v1.1.0. https://geoparquet.org/releases/v1.1.0  
2. Efficient Storage and Querying of Geospatial Data with Parquet and DuckDB. https://waterprogramming.wordpress.com/2022/03/07/efficient-storage-and-querying-of-geospatial-data-with-parquet-and-duckdb
