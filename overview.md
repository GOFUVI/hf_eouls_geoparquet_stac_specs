# Overview

## Introduction

The **HF-EOLUS project** processes two major data streams: (1) coastal high-frequency radar measurements (HF-Radar) used for ocean surface wind inversion, and (2) meteorological model outputs used for interpolation and analysis. Both workflows generate **massive volumes of geospatial data** that must be stored efficiently and made accessible for cloud-based processing pipelines. To meet these needs, HF-EOLUS adopts **GeoParquet** [\[1\]](#ref1) as a compact, analysis-ready storage format and uses **STAC** metadata [\[2\]](#ref2) to organize and describe the data according to
international standards.

By using open standards, the project avoids bespoke formats: data
producers can leverage well-defined conventions, and consumers
(analysts, modelers) can use off-the-shelf tools to query and retrieve data. In essence, **GeoParquet** handles the efficient storage of the data itself, while **STAC** provides a standardized, self-describing *catalog* of the data assets for easy discovery and access.


### HF-Radar

**HF-Radars** are shore-based systems that measure ocean surface
currents across wide areas. CODAR's SeaSonde systems derive **Radial Metrics** by applying the MUSIC algorithm to Doppler spectra. These Radial Metrics are an intermediate data product containing individual echo detections with attributes such as timestamp, bearing (angle from the radar), range (distance from the radar), signal strength, radial velocity components, and various quality metrics. Because *every individual echo* is retained rather than averaging into grid cells immediately, the data volume grows rapidly. Even when restricting to frequency bands of interest (the Bragg peaks containing ocean current signals), a week of HF-Radar operation with a half-hour sampling interval can yield over **one million echo records**. Over months, this results in extremely large datasets that are
challenging to manage and analyze in raw form.

Historically, Radial Metrics (and similar HF radar products) have been exported in formats like **LLUV** [\[3\]](#ref3), a custom ASCII text table format developed by CODAR. A Radial Metrics LLUV file consists of: (1) a header with global metadata (station ID, timestamp, processing parameters, etc.), (2) a large table listing each echo's measurements (bearing, range, etc.), and (3) a summary section aggregating echoes statistics by range bin. While human-readable, a plain text or CSV-like format such as LLUV does not scale well for *big data* analytics or long time series -- parsing millions of lines of text is slow and storage is inefficient

Therefore, converting these files into a more compact, binary format optimized for analytical queries became essential.

### Meteorological Model Outputs

In addition to radar data, HF-EOLUS utilizes meteorological model outputs (e.g. wind or atmospheric fields) for interpolation and analysis. These model datasets are typically multi-dimensional arrays (spatial grids over latitude/longitude and possibly altitude, at multiple time steps). Traditionally, such data are stored in formats like **NetCDF** or GRIB, which are designed for multi-dimensional scientific data. However, to **unify the data management strategy**, HF-EOLUS also stores model outputs using the GeoParquet approach where feasible. In practice, model output fields can be transformed into tabular structures (for example, by listing grid point values with their coordinates) so that they too can be stored as Parquet files. By doing so, the project leverages the same tooling -- for instance, using Parquet's efficient compression and the ability to perform SQL-like queries on both radar and model data.

Admittedly, gridded model data are less naturally tabular than point-based radar echoes. NetCDF remains a common choice for such arrays due to its self-describing nature for multi-dimensional grids. Nonetheless, HF-EOLUS demonstrates that with careful structuring, even model data can be made query-friendly. For example, one can store each model time slice or each variable as a separate Parquet table of values, enabling cloud analytics across many files. The **GeoParquet** standard's flexibility with multiple geometry columns or even non-spatial tabular data means both point observations and gridded outputs can reside in a common data lake. The specifics of how model data are transformed and cataloged may differ (e.g., model outputs might use point geometries for grid cell centers or polygons for cells), but the overarching standard ensures consistency across the dataset collection.

In summary, the project's **dual use** of GeoParquet and STAC for both HF-Radar and model data ensures that *all* data is stored in a **cloud-optimized**, analysis-ready format and described with **standard metadata**. The following sections detail the conventions and specifications adopted for GeoParquet files and the STAC catalog.


## GeoParquet Format

GeoParquet v1.1.0 extends Apache Parquet with standardized conventions for geospatial data. It describes how geometry columns, coordinate reference systems (CRS), and file metadata must be encoded to ensure interoperability across tools. See the [GeoParquet Specification](geoparquet_specs.md) for the full set of rules and examples applied in project HF-EOLUS.

The GeoParquet specifications are applied uniformly to both HF-Radar data and meteorological model outputs.

## STAC

STAC provides a lightweight framework for publishing geospatial metadata. For details of the STAC specifications implemented in HF-EOLUS, refer to the [STAC Specification](stac_specs.md). 

### HF-Radar

When storing Radial Metrics, each STAC Item represents measurements derived from one spectra file recorded by a HF-Radar station at a specific timestamp and links to two GeoParquet assets: one for the Radial Metrics derived from positive Bragg peak and one for the negative peak. Collections group Items by station and antenna pattern, enabling organized discovery. 

## End-to-end dataflow

The HF-EOLUS documentation maintains a living [dataset inventory](dataset_inventory.md) that records every GeoParquet deliverable, partition scheme, and STAC extension in use across the external workflows. The overview below contextualizes how those datasets connect across the production chain so readers can trace each downstream output back to its upstream assets.

### Main ingestion → aggregation → ANN/resource loop

1. **Radial metrics ingestion workflow** converts CODAR LLUV spectra into partitioned GeoParquet tables (`radial_metrics`, `header`, `rng_info`) and publishes the authoritative STAC catalog for the VILA and PRIO stations. The ingestion metadata captured in the inventory clarifies how timestamps, Bragg polarity, and header diagnostics remain synchronized across assets.
2. **Sentinel-1 SAR and Puertos del Estado buoy workflows** provide external supervision for later stages. Their GeoParquet shards keep deterministic identifiers (`rowid`, buoy station codes) and dedicated STAC extensions (Table, Scientific, Processing) so that aggregation jobs can join them with radar echoes by space and time.
3. **Geospatial aggregation toolkit** aligns echo-level measurements to the analysis grid used by the wind-inversion models. The resulting GeoParquet files (per-station, per-Bragg polarity, plus a consolidated SAR aggregate) are described in the inventory together with their `node_id` partitions and CRS settings, ensuring they can be cross-referenced with both the ingestion outputs and the ANN-ready tables.
4. **ANN training and inference toolkit** blends the aggregated HF-radar, SAR, and buoy datasets into pivot tables, encodes fold assignments, and emits inference snapshots. Dataset inventory entries explain how columns such as `crc32_partition_key`, `fold`, probabilistic range scores, and maintenance diagnostics are preserved via the Table extension so that downstream resource assessments understand each prediction’s provenance.
5. **Wind-resource estimation toolkit** ingests the ANN inference corpus to compute power-density summaries, Weibull parameters, Kaplan–Meier survival statistics, and QA diagnostics per HF node. Its catalog advertises the `derived_from` links back to the ANN deliverables, while the inventory lists the manifest sidecars and quality flags that document how every resource estimate maps to the training artifacts.

 Together, these stages capture the primary ingestion → aggregation → ANN/resource flow: every downstream file references its parent assets via STAC links, and every schema reference resolves to the dataset inventory for reproducible partitioning and column semantics.

### Parallel MeteoGalicia interpolation branch

The MeteoGalicia interpolation workflow operates as an independent branch that synchronizes model outputs, performs kriging or nearest-neighbor interpolation on dense grids, and repackages the results into its own STAC hierarchy (data, metadata JSON, and diagnostic plots). Its GeoParquet shards partition by year/month/day/hour and annotate each record with `source_model`, interpolation quality metrics, and node descriptors so the catalog can stand alone. As noted in the dataset inventory, these assets are currently distributed separately from the ANN/resource chain: no STAC `derived_from` links connect the interpolation catalog to the ANN or wind-resource toolkits, and validation steps rely on colocated PdE buoy archives instead. Documenting this branch alongside the main loop makes it clear that the interpolation outputs follow the same GeoParquet/STAC conventions yet remain decoupled until the project formally integrates them into the wind-resource assessments.


## References

<ol>
<li id="ref1">GeoParquet. https://geoparquet.org</li>
<li id="ref2">STAC Contributors. (2024). SpatioTemporal Asset Catalog (STAC) specification (Version 1.1.0). https://stacspec.org</li>
<li id="ref3">LLUV File Format. http://support.codar.com/Technicians_Information_Page_for_SeaSondes/Manuals_Documentation_Release_8/File_Formats/File_LLUV.pdf
