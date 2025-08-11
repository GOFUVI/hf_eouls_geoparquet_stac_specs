# Overview

HF-Radars are coastal radar systems designed to measure surface currents over wide swaths of ocean. CODAR's SeaSonde systems derive **Radial Metrics** by applying the MUSIC algorithm to Doppler spectra, creating an intermediate product between raw spectral data and the gridded surface-current maps. Each metric records an echo's bearing, range, signal strength, and related metadata.

Because every individual echo is retained, data volumes grow rapidly; even when restricted to spectral bands containing current information, a half-hour sampling interval can yield more than one million echoes in under a week. Efficient storage and organization are therefore essential for long-term analyses.

This repository defines how to store Radial Metrics using the GeoParquet format and how to organize those files with a STAC catalog.

## GeoParquet Format

GeoParquet v1.1.0 extends Apache Parquet with standardized conventions for geospatial data. It describes how geometry columns, coordinate reference systems (CRS), and file metadata must be encoded to ensure interoperability across tools. See the [GeoParquet Specification](geoparquet_specs.md) for the full set of rules and examples.

## STAC Catalog and Item Design

The SpatioTemporal Asset Catalog (STAC) provides a lightweight framework for publishing geospatial metadata. In this repository, each STAC Item represents measurements from a station at a specific timestamp and links to two GeoParquet assets: one for the positive Bragg peak and one for the negative peak. Collections group Items by station and antenna pattern, enabling organized discovery. For details of the catalog structure, refer to the [STAC Specification](stac_specs.md).
