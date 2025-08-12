# Overview

## Introduction

El proyecto HF-EOLUS procesa grandes cantidades de datos, ya sean de HF-Radar para la inversion del viento, ya sean salidas de modelos meteorológicos para su interpolación. En ambos casos se requiere de un formato de datos eficiente que permitan la aplicación de pipelines de procesado en la nube, y una organización de los datos de acuerdo a estándares internacionales. Ambos objetivos se abordan en este proyecto mediante la aplicación de formato GeoParquet para el almacenamiento de los datos y de los estándares definidos por el SpatioTemporal Asset Catalog (STAC) para la organización de los mismos.

### HF-Radar

HF-Radars are coastal radar systems designed to measure surface currents over wide swaths of ocean. CODAR's SeaSonde systems derive **Radial Metrics** by applying the MUSIC algorithm to Doppler spectra, creating an intermediate product between raw spectral data and the gridded surface-current maps. Each metric records an echo's bearing, range, signal strength, and related metadata.

Because every individual echo is retained, data volumes grow rapidly; even when restricted to spectral bands containing current information, a half-hour sampling interval can yield more than one million echoes in under a week. Efficient storage and organization are therefore essential for long-term analyses.

## GeoParquet Format

GeoParquet v1.1.0 extends Apache Parquet with standardized conventions for geospatial data. It describes how geometry columns, coordinate reference systems (CRS), and file metadata must be encoded to ensure interoperability across tools. See the [GeoParquet Specification](geoparquet_specs.md) for the full set of rules and examples applied in project HF-EOLUS.

Las especificaciones GeoParquet se aplican por igual a los datos de HF-Radar y a las salidas de modelos meteorológicos.

## STAC

The SpatioTemporal Asset Catalog (STAC) provides a lightweight framework for publishing geospatial metadata. For details of the STAC specifications implemented in HF-EOLUS, refer to the [STAC Specification](stac_specs.md). 

### HF-Radar

When storing Radial Metrics, each STAC Item represents measurements derived from one spectra file recorded by a HF-Radar station at a specific timestamp and links to two GeoParquet assets: one for the Radial Metrics derived from positive Bragg peak and one for the negative peak. Collections group Items by station and antenna pattern, enabling organized discovery. 
