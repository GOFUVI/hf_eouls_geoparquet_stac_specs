# HF-EOLUS Project GeoParquet & STAC Specifications

This documentation describes how the **HF-EOLUS** project stores large volumes of high-frequency radar *Radial Metrics* data in the **GeoParquet** format and organizes them with **STAC** (SpatioTemporal Asset Catalog) metadata standards. It serves as a guide for technical users interested in data storage and cataloging strategies for geospatial datasets. While HF-Radar observations are the primary focus, the same principles apply to meteorological model outputs processed in the project. Key benefits include efficient, cloud-optimized storage and internationally standardized metadata for interoperability.

This repository documents how the HF-EOLUS project stores high-frequency radar radial metrics in [GeoParquet](geoparquet_specs.md) and describes the [STAC](stac_specs.md) metadata that organizes those assets. For a broader context on the project and data flow, see the [overview](overview.md).

## Table of Contents
- [Overview](#overview)
- [Processing Lines](#processing-lines)
- [GeoParquet Specification](#geoparquet-specification)
- [STAC Specification](#stac-specification)

## Overview
HF-EOLUS processes large volumes of HF-Radar observations and meteorological model outputs. GeoParquet provides a compact, analysis-ready storage format while STAC supplies standardized metadata so catalogs remain portable and easy to browse. The [overview](overview.md) explains the motivation and how these pieces fit together.

## Processing Lines
HF-EOLUS organizes its deliverables into seven processing lines that share the same GeoParquet and STAC conventions while remaining individually citable through Zenodo DOIs. The summaries below connect each workflow to its role in the project and reference both the workflow documentation DOI and the DOI of the public dataset or catalog whenever available.

### HF-Radar Radial Metrics Ingestion
Workflow DOI: [10.5281/zenodo.17071947](https://doi.org/10.5281/zenodo.17071947) · Dataset DOI: [10.5281/zenodo.16892222](https://doi.org/10.5281/zenodo.16892222)  
This pipeline ingests CODAR SeaSonde LLUV spectra, extracts the radial metrics for the positive and negative Bragg peaks, and rewrites every echo, header, and range diagnostic into partitioned GeoParquet shards with embedded GeoParquet `geo` metadata. Its STAC catalog exposes the per-station hierarchy (`VILA`, `PRIO`) that the rest of HF-EOLUS references, making it the authoritative source for point-level observations used in all downstream analytics.

### Sentinel-1 SAR Ingestion
Workflow DOI: [10.5281/zenodo.17011788](https://doi.org/10.5281/zenodo.17011788) · Dataset DOI: [10.5281/zenodo.17007304](https://doi.org/10.5281/zenodo.17007304)  
This line processes Sentinel-1 OCN products into daily GeoParquet partitions where every OWI vector is spatialized as a CRS84 point and documented through the STAC Table and Processing extensions. The SAR wind fields complement HF-Radar measurements during aggregation and model training, so their schema and lineage metadata are kept compatible with Athena/DuckDB queries as well as with the STAC collections consumed by the ANN and wind-resource pipelines.

### Puertos del Estado Buoy Ingestion
Workflow DOI: [10.5281/zenodo.17097948](https://doi.org/10.5281/zenodo.17097948) · Dataset DOI: [10.5281/zenodo.17098037](https://doi.org/10.5281/zenodo.17098037)  
The buoy ingestion workflow consolidates hourly in-situ records from Puertos del Estado into GeoParquet snapshots per buoy (e.g., Vilano) and republishes them via STAC Items that include Scientific Extension metadata. These assets provide the calibrated reference winds that the aggregation toolkit and the ANN validation steps use to benchmark radar- and SAR-derived winds, ensuring the HF-EOLUS documentation remains traceable back to official PdE observations.

### HF-EOLUS Geo Tools
Workflow DOI: [10.5281/zenodo.17104923](https://doi.org/10.5281/zenodo.17104923) · Dataset DOI: [10.5281/zenodo.17105305](https://doi.org/10.5281/zenodo.17105305)  
This toolkit aligns the per-echo HF-Radar tables with the fixed analysis grid, aggregates velocity and power statistics per `node_id`, and produces a parallel set of SAR aggregates over the same mesh so downstream workflows can combine them explicitly when needed. The resulting GeoParquet derivatives (one shard per station/Bragg polarity plus a consolidated SAR aggregate) fuel every ANN-ready dataset. Its STAC catalogs maintain station-specific namespaces (`hf_site:*`) and Table extension payloads so consumers can inspect schema metadata without reading the binaries.

### HF-Radar Wind Inversion Toolkit
Workflow DOI: [10.5281/zenodo.17369263](https://doi.org/10.5281/zenodo.17369263) · Dataset DOI: [10.5281/zenodo.17131227](https://doi.org/10.5281/zenodo.17131227)  
HF-wind-inversion pivots the aggregated datasets into ANN-ready tables, encodes cross-validation folds, and publishes inference snapshots for multiple training regimes (baseline, fine-tuned, KD, buoy-referenced, grid-offset). Every deliverable remains a single GeoParquet asset with exhaustive Table extension metadata so predictions, probabilistic scores, and maintenance diagnostics can be queried alongside their provenance. These outputs are the direct inputs for the wind-resource assessment toolkit described below.

### Wind Resource Toolkit
Workflow DOI: [10.5281/zenodo.17591545](https://doi.org/10.5281/zenodo.17591545) · Dataset DOI: [10.5281/zenodo.17594220](https://doi.org/10.5281/zenodo.17594220)  
The wind-resource line operates on the ANN inference corpus to compute per-node resource summaries, Weibull/empirical distributions, power-curve statistics, and QA diagnostics. The Zenodo DOIs cited above keep the cataloged GeoParquet snapshots—pivot joins as well as the final `power_estimates_nodes.parquet` bundle—tightly coupled to their manifest sidecars so every estimate can be traced back to the ANN build and to the upstream evidence chain documented across the previous workflows.

### MeteoGalicia Wind Interpolation
Workflow DOI: [10.5281/zenodo.17598353](https://doi.org/10.5281/zenodo.17598353) · Dataset DOI: [10.5281/zenodo.17490872](https://doi.org/10.5281/zenodo.17490872)  
This branch synchronizes MeteoGalicia model outputs, performs kriging/nearest-neighbor interpolation on dense grids, and exposes the hourly GeoParquet shards together with metadata JSON files and diagnostic plots in its own STAC catalog tree. It intentionally remains a standalone branch that does not yet feed the HF-Radar wind-resource chain; nevertheless, the published DOIs ensure the interpolation deliverables adhere to the same GeoParquet/STAC conventions so they can be integrated when downstream consumers are ready.

Cross-document validation of these seven workflows is summarised in the [Cross-Document Validation Checklist](docs/cross_document_validation_checklist.md), which lists the exact sections across README, overview, GeoParquet specs, and STAC specs that describe each processing line.

## GeoParquet Specification
The [GeoParquet specification](geoparquet_specs.md) details how geometry columns, coordinate reference systems, and file-level metadata are encoded. It follows GeoParquet v1.1.0, including optional fields like polygon orientation and edge type to ensure interoperability across tools.

## STAC Specification
The [STAC specification](stac_specs.md) describes how radial metrics are published as STAC Items and Collections. Each Item references paired GeoParquet assets for positive and negative Bragg peaks, enabling time-partitioned queries and organized discovery across stations.
## Acknowledgements

This work has been funded by the HF-EOLUS project (TED2021-129551B-I00), financed by MICIU/AEI /10.13039/501100011033 and by the European Union NextGenerationEU/PRTR - BDNS 598843 - Component 17 - Investment I3. Members of the Marine Research Centre (CIM) of the University of Vigo have participated in the development of this repository.

## Disclaimer
This software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, or in connection with the software or the use or other dealings in the software.

---
<p align="center">
  <a href="https://next-generation-eu.europa.eu/">
    <img src="logos/EN_Funded_by_the_European_Union_RGB_POS.png" alt="Funded by the European Union" height="80"/>
  </a>
  <a href="https://planderecuperacion.gob.es/">
    <img src="logos/LOGO%20COLOR.png" alt="Logo Color" height="80"/>
  </a>
  <a href="https://www.aei.gob.es/">
    <img src="logos/logo_aei.png" alt="AEI Logo" height="80"/>
  </a>
  <a href="https://www.ciencia.gob.es/">
    <img src="logos/MCIU_header.svg" alt="MCIU Header" height="80"/>
  </a>
  <a href="https://cim.uvigo.gal">
    <img src="logos/Logotipo_CIM_original.png" alt="CIM logo" height="80"/>
  </a>
  <a href="https://www.iim.csic.es/">
    <img src="logos/IIM.svg" alt="IIM logo" height="80"/>
  </a>
</p>
