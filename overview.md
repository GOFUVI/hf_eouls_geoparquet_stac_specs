# Overview

## Introduction

The HF-EOLUS project processes large volumes of data, from HF-Radar observations used for wind inversion to meteorological model outputs used for interpolation. Both workflows require an efficient data format that enables cloud-based processing pipelines and organizes information according to international standards. This project meets those needs by storing data in the GeoParquet format and organizing it with the standards defined by the SpatioTemporal Asset Catalog (STAC).

### HF-Radar

HF-Radars are coastal radar systems designed to measure surface currents over wide swaths of ocean. CODAR's SeaSonde systems derive **Radial Metrics** by applying the MUSIC algorithm to Doppler spectra, creating an intermediate product between raw spectral data and the gridded surface-current maps. Each metric records an echo's bearing, range, signal strength, and related metadata.

Because every individual echo is retained, data volumes grow rapidly; even when restricted to spectral bands containing current information, a half-hour sampling interval can yield more than one million echoes in under a week. Efficient storage and organization are therefore essential for long-term analyses.

## GeoParquet Format

GeoParquet v1.1.0 extends Apache Parquet with standardized conventions for geospatial data. It describes how geometry columns, coordinate reference systems (CRS), and file metadata must be encoded to ensure interoperability across tools. See the [GeoParquet Specification](geoparquet_specs.md) for the full set of rules and examples applied in project HF-EOLUS.

The GeoParquet specifications are applied uniformly to both HF-Radar data and meteorological model outputs.

## STAC

The SpatioTemporal Asset Catalog (STAC) provides a lightweight framework for publishing geospatial metadata. For details of the STAC specifications implemented in HF-EOLUS, refer to the [STAC Specification](stac_specs.md). 

### HF-Radar

When storing Radial Metrics, each STAC Item represents measurements derived from one spectra file recorded by a HF-Radar station at a specific timestamp and links to two GeoParquet assets: one for the Radial Metrics derived from positive Bragg peak and one for the negative peak. Collections group Items by station and antenna pattern, enabling organized discovery. 
