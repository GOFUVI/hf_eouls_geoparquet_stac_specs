# HF-Radar Radial Metrics GeoParquet & STAC Specifications

This repository documents how the HF-EOLUS project stores high-frequency radar radial metrics in [GeoParquet](geoparquet_specs.md) and describes the [STAC](stac_specs.md) metadata that organizes those assets. For a broader context on the project and data flow, see the [overview](overview.md).

## Table of Contents
- [Overview](#overview)
- [GeoParquet Specification](#geoparquet-specification)
- [STAC Specification](#stac-specification)

## Overview
HF-EOLUS processes large volumes of HF-Radar observations and meteorological model outputs. GeoParquet provides a compact, analysis-ready storage format while STAC supplies standardized metadata so catalogs remain portable and easy to browse. The [overview](overview.md) explains the motivation and how these pieces fit together.

## GeoParquet Specification
The [GeoParquet specification](geoparquet_specs.md) details how geometry columns, coordinate reference systems, and file-level metadata are encoded. It follows GeoParquet v1.1.0, including optional fields like polygon orientation and edge type to ensure interoperability across tools.

## STAC Specification
The [STAC specification](stac_specs.md) describes how radial metrics are published as STAC Items and Collections. Each Item references paired GeoParquet assets for positive and negative Bragg peaks, enabling time-partitioned queries and organized discovery across stations.
