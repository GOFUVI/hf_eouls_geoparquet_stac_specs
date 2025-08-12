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

## HF-EOLUS Specifications

### HF-Radar

For each timestamp we publish two GeoParquet files: one containing radial metrics derived from the **positive** Bragg peak and another from the **negative** peak. These files constitute the data assets referenced by a STAC Item. To streamline ingestion into analytical databases—such as AWS Athena or DuckDB—all assets for a station are stored in a dedicated `assets/` directory so that time-partitioned queries can scan them efficiently.

Each Item bundles the pair of GeoParquet assets for a specific timestamp alongside metadata such as the station identifier, acquisition time, and processing parameters. Items for a station are grouped into Collections. A Collection encompasses all radial metrics from a station processed with a particular Antenna Pattern (APM); consequently every Item in the Collection shares the APM’s UUID, which is recorded as a Collection property.

Collections representing distinct APM periods roll up into a station-level Catalog, and the set of station catalogs is linked together under a single root STAC Catalog. Each Item is stored as a `*.stac.json` file located alongside its assets, enabling the catalog to remain self-contained when copied or hosted from object storage. The Collection uses STAC’s [Table extension](https://github.com/stac-extensions/table) to document the tabular schema—column names, types, and units—so that clients can interpret the GeoParquet data without inspecting every file.

Asset filenames follow a consistent pattern of `{station_id}_{timestamp}_{bragg}.parquet`, where `bragg` is `pos` or `neg` for the positive and negative peaks, respectively. Each Item JSON adopts the same stem—`{station_id}_{timestamp}.stac.json`—so relative links remain stable even when catalogs are copied between storage locations or published online.
Timestamps are encoded in UTC using ISO 8601 so that simple lexical sorting also arranges Items chronologically.
