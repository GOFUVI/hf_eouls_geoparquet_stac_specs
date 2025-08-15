# HF-Radar Radial Metrics GeoParquet & STAC Specifications

This documentation describes how the **HF-EOLUS** project stores large volumes of high-frequency radar *Radial Metrics* data in the **GeoParquet** format and organizes them with **STAC** (SpatioTemporal Asset Catalog) metadata standards. It serves as a guide for technical users interested in data storage and cataloging strategies for geospatial datasets. While HF-Radar observations are the primary focus, the same principles apply to meteorological model outputs processed in the project. Key benefits include efficient, cloud-optimized storage and internationally standardized metadata for interoperability.

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
