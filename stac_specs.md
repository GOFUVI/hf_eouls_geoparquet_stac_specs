# STAC Specification

## STAC Overview

The SpatioTemporal Asset Catalog (STAC) specifications provide a standardized, lightweight framework for representing and discovering geospatial asset metadata. At its heart, STAC treats a “spatiotemporal asset” as any resource that captures information about the Earth at a specific location and time.

STAC’s core remains intentionally minimal, with most functionality delivered through extensions. This modular approach has enabled its evolution into a mature standard, adopted by many production systems.

By following STAC, data providers avoid inventing bespoke metadata formats and interfaces, while data consumers benefit from a common set of conventions and tooling. This shared foundation reduces the effort required on both ends: publishers can export metadata in a well-defined structure, and clients can ingest it using existing libraries rather than crafting custom parsers or APIs.

The Item, Catalog, and Collection specifications define the essential JSON objects. Because these objects form a simple hierarchy and use hypermedia links, a STAC catalog can be hosted as a purely static set of interlinked JSON files, making it trivial to browse or serve from object storage.

## Item Overview

A STAC Item represents a single data record—typically one scene captured at a given time and place—and is modeled as a GeoJSON Feature. This ensures compatibility with GIS software and geospatial libraries. In addition to the GeoJSON geometry, each Item includes:

- Temporal information for when the data is valid.
- A thumbnail or preview image.
- Asset links for direct download or streaming of the data.
- Relationship links for navigating between related Items or other resources.

Items may also include custom properties or extension-defined fields to support search, discovery, and domain-specific metadata.

## Catalogs vs Collections

A STAC Catalog is simply a container that links to Items or to other Catalogs, analogous to a folder in a filesystem hierarchy.

A STAC Collection shares Catalog’s linking behavior but adds descriptive fields—such as license, spatial and temporal extents, providers, keywords, and summaries—that apply uniformly to all Items within it. Each Item points back to its parent Collection, making Collection-level metadata (like licensing) easily discoverable.

## Collection Overview

A Collection inherits Catalog’s core attributes (id, description, stac_version, and links) and augments them with license and extent information by default. Many other common properties appear in the core specification, and additional metadata can be provided through STAC extensions (e.g., ISO 19115 references).

Since Collections encapsulate Catalog’s functionality, they can nest arbitrarily under other Catalogs or Collections and contain child Items, Catalogs, or Collections. Items should always link to the most specific (i.e., closest) Collection they belong to, because an Item may only reference one Collection.

Standalone Collection documents are also useful: they describe a grouping of assets without necessarily exposing a deeper hierarchy. This pattern is common when software operates on whole data layers or coverages as single units.

## Catalog Overview

A Catalog document can link to both STAC Items and other Catalogs, providing a simple recursive structure for organizing datasets. Implementers are free to structure catalogs to meet their own requirements.

### Self-contained Catalogs

Self-contained catalogs prioritize portability by employing only relative URLs for structural links (root, parent, child, item, collection). This makes it possible to copy or move the entire catalog without breaking internal navigation. Links that point to external resources—such as sci:doi, derived_from, or license—may remain absolute.

### Self-contained Catalogs with Embedded Assets

In this variant, asset files themselves are included in the same directory hierarchy and referenced with relative URLs. This ensures that both metadata and raw data are available offline, simplifying distribution and local use.

### Relative Published Catalog

A relative published catalog builds on the self-contained pattern by adding a single absolute self link at the root to establish its online location. All other hyperlinks remain relative, so the catalog retains its offline portability while also providing a stable, authoritative URL for published deployments.

## Especificaciones para HF-EOLUS

### Radial Metrics

En nuestras especificaciones para Radial Metrics tenemos, para cada timestamp, un fichero geoparquet para los radial metrics provenientes del pico de bragg positivo y otro para el negativo. Estos son nuestros assets. Con el fin de facilitar su ingestion en una bese de datos (por ejemplo Athena en AWS) Todos los assets de una estacion se almacenan en un mismo directorio assets. Un item incluye los dos ficheros geoparquet de cada timestamp (el que contiene los radial metrics del pico positivo y el que contiene los radial metrics del pico negativo). Los items de una estacion se organizan en colecciones. Una coleccion son todos los radial metrics de una estacion procesados un mismo Antenna Pattern. Por tanto todos los items de una coleccion tienen en comun el UUID del pattern empleado para procesarlos. Las colecciones de periodos de antenna pattern se organizan en estaciones mediante un catalogo. Un catalogo de estaciones es la raiz del catalogo STAC.
