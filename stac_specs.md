# STAC Specification

## STAC Overview

The **SpatioTemporal Asset Catalog (STAC)** is a standard for describing geospatial datasets with metadata that is both **machine-readable** and **web-friendly**. STAC defines a common structure for cataloging data like satellite imagery, maps, point clouds, etc., so that these assets can be easily indexed and discovered by location, time, and other properties. The core STAC specification is intentionally minimal \-- it focuses on the essentials needed to locate data in space and time \-- and it relies on a flexible extension mechanism to accommodate domain-specific needs. This design allows STAC to be widely applicable: different projects can extend it (adding additional fields or semantics) without breaking the fundamental compatibility. For a full description of the STAC specifications, please s [\[1\]](#ref1).

Concretely, STAC defines a set of **JSON** document types:

-   **Item**: Represents an individual spatiotemporal asset or data capture (analogous to a single scene or observation). A STAC Item is structured as a GeoJSON Feature, meaning it has a geometry (e.g., the footprint of an image or the location of a point measurement) and standard GeoJSON properties like coordinates. Additionally, it includes time information and links to the actual data files (assets).

-   **Catalog**: Provides a way to hierarchically organize Items (and Collections) using links, similar to folders in a file system. A Catalog is essentially a container of links and metadata that can point to either Items or other Catalogs, enabling nesting.

-   **Collection**: A special kind of Catalog that includes extra metadata describing a group of Items with shared characteristics. For example, a Collection might represent a dataset (like \"Landsat 8 Surface Reflectance\") and contain fields such as the license, the spatial and temporal extent of all items, keywords, providers, etc., that apply to the whole set of Items. Each Item can reference which Collection it belongs to.

Using these building blocks, one can create a *static catalog* \-- a set of JSON files stored, say, on a cloud bucket or a website, where each file links to others to form a navigable tree. Because the links can be relative, it\'s possible to host a STAC catalog simply as static files on disk or object storage, making it very portable. This is in contrast to older catalog systems that might require a database or server backend. With STAC, even a large data repository can be exposed through a bunch of JSONs that any client or web browser plugin can traverse.

To illustrate STAC\'s minimalism, the **core fields** in an Item include things like an `id`, a `bbox` (bounding box), a `geometry` (the GeoJSON geometry), a `datetime` (timestamp of the asset), and an `assets` dictionary (with keys for each asset file and values containing links and asset metadata). Additional fields can be added via *extensions* \-- JSON Schema definitions that specify extra properties. For example, there are STAC extensions for electro-optical imagery (adding things like cloud cover percentage), for scientific citation (DOIs, etc.), for raster and vector data specifics, and many more. HF-EOLUS takes advantage of this extensibility as needed (as discussed later, we use the Table extension to describe columns of tabular data).

By adopting STAC, **data providers** (like our project) ensure that we\'re not inventing yet another metadata format for our assets. Instead, we use a common language, which means anyone familiar with STAC can understand our catalog. Likewise, **data consumers** benefit because they can use existing tools \-- e.g., STAC browser apps, libraries like PySTAC in Python, or STAC-index \-- to search and retrieve data from our catalog without custom code. This significantly lowers the barrier to entry: a researcher can, for instance, load our STAC catalog in a Jupyter notebook using PySTAC and programmatically find all HF-Radar files in a given date range and geographic area with just a few lines of code.

In summary, STAC provides a **lightweight, web-ready index** of our geospatial data, ensuring interoperability and ease of use. The following subsections outline the main STAC components and then detail how HF-EOLUS applies them for organizing the radar metrics data.

## Item Overview

A **STAC Item** is the atomic unit of data in a STAC Catalog \-- it corresponds to a single spatiotemporal observation or logical grouping of data that one would consider a single record. In STAC (core 1.0.0), an Item is defined as a JSON that also qualifies as a **GeoJSON Feature**. This means an Item has:

-   A `geometry`: usually the footprint of the data. For imagery, this would be the polygon of the scene\'s coverage. For HF-Radar, this can be a point or small area representing the station location or coverage of that item\'s data.

-   A `bbox`: the bounding box of that geometry (for quick indexing).

-   `properties`: a dictionary of metadata key-values (for example, the timestamp of acquisition is given by `datetime` in properties, plus any other relevant metadata like instrument type or processing info).

-   An `id`: a unique identifier for the item.

-   An `assets` list (or dictionary): each asset entry provides a link (URL or path) to an actual data file plus some metadata like file type, size, or descriptive title.

-   A `links` list: links to related entities (e.g., a link to its parent Collection, links to previous/next in a time series, or a link to license info). These links allow the catalog to be navigable.

Because a STAC Item is a *GeoJSON Feature*, it ensures immediate compatibility with GIS tools. One can take the geometry and properties and plot them on a map, or index them in a spatial database, etc. The use of GeoJSON also means the spatial reference is by default WGS84 geographic (lat/lon), which is typically expected for catalogs (the geometry field in the Item is assumed to be in WGS84 coordinates unless otherwise specified). This is convenient because our HF-Radar data and model data are indeed referenced to WGS84 lat/lon (as indicated by the GeoParquet CRS as well).

An Item usually represents **one snapshot in time** of something. For example:

-   In a satellite context, one Item = one image scene at a certain date/time.

-   In our HF-Radar context, as we will see, one Item = the set of radar measurements (radial metrics) derived from one radar\'s half-hourly spectral data at a given timestamp.

-   If we were cataloging model outputs, one Item could represent one model run output at a certain time (e.g., a forecast for 12:00 UTC on Jan 1, with assets being the output files).

The STAC Item\'s simplicity is by design. Additional domain-specific details (like the fact that an HF-Radar item contains two types of radial data, or that a model output item might have multiple variables) can be captured either in the `assets` (by having multiple asset entries with different keys) or in extension properties. For instance, the **Table Extension** we use will allow us to include the schema of a tabular asset as part of the Item or Collection metadata, so that a user knows what columns exist in the Parquet file without opening it.

## Catalogs vs Collections

Both **Catalogs** and **Collections** are mechanisms to group Items, but they have distinct roles:

-   A **STAC Catalog** is a basic listing mechanism. It has an `id`, a description, and a set of links (some of which are labeled as `child` or `item` or `parent`). Think of a Catalog as a folder that can contain references to Items or to other folders. A Catalog does not enforce any specific metadata beyond identification and linking. It\'s purely structural. You might use nested catalogs to reflect a logical grouping (e.g., group by region, by year, by data type).

-   A **STAC Collection** extends a Catalog by adding dataset-level metadata. In STAC, a Collection is effectively a Catalog with extra required fields like:

-   `license`: the usage license of the data (e.g., CC-BY-4.0, or proprietary, etc.).

-   `extent`: both spatial extent (bounding box covering all items) and temporal extent (time range covered by all items).

-   `providers`: who produced or hosts the data.

-   `summaries`: optional statistical summaries for certain properties (e.g., a list of all unique station IDs in the collection, or min/max values of a parameter).

Every Item can optionally link to a Collection it belongs to (via a `collection` field or a link of type `collection`). This is recommended because it lets a user or client quickly find the overarching context for that item. For example, a Collection might state \"these are HF-Radar radial metrics from Station X, covering 2023, in CSV format, with these variables\" in its description and have the official citation and DOI. Each Item then doesn\'t need to repeat those details; it just points to the Collection.

In our project, we leverage **Collections** to represent groupings like \"all data from a specific radar station and antenna pattern configuration\" (more on that below in HF-EOLUS specifics). By doing so, we capture at the Collection level the things common to that group (e.g., the station location, the period it operated, the coordinate system, etc.), which clients can read once. Items then handle the per-timestep specifics.

In contrast, pure **Catalogs** (non-collection catalogs) we use mainly to build the hierarchy above the collections. For example, we might have a top-level Catalog called \"HF-EOLUS Data Catalog\" with children that are Collections per station. Or we could have an intermediate Catalog grouping stations by region if needed. Catalogs are flexible and impose no rules, which is useful for purely organizing navigation.

## Collection Overview

A STAC **Collection** in HF-EOLUS is used whenever we have a set of Items sharing the same data schema and context. By using Collections, we provide rich metadata once for the whole set:

-   We define the **spatial extent** (the geographic coverage of the collection). For a radar station, this might be the radius or area the station can observe (or simply the station location if we consider each echo\'s location variable).

-   We define the **temporal extent** (earliest and latest timestamps of data in that collection).

-   The **license** for the data is stated at the collection level (so every item implicitly carries that license by reference).

-   We list the **providers** (e.g., the organizations or agencies involved in producing the data).

-   We can include a list of **keywords** (e.g., \"HF-Radar\", \"ocean currents\", \"radial velocity\").

-   If using extensions, the collection can specify which extensions apply to its items (for instance, if all items use the `table` extension and perhaps a custom extension for HF radar, the collection will list those).

The collection document also has a `summaries` section where we can provide summary statistics or enumerations of fields. For example, a collection could list all station IDs it contains (though in our case one station per collection, so not needed), or range of frequencies or antenna patterns.

One key reason to use collections is to facilitate searches. If one is using a STAC API or a static catalog search tool, collections allow filtering at a high level (e.g., getCollections by instrument type). In our static context, it\'s more about organization and metadata clarity.

Collections can themselves be nested under catalogs or even under other collections (since a Collection is a kind of Catalog, STAC allows a hierarchy where collections link to children). However, typically, you have either a Catalog containing Collections or Collections at the top. In HF-EOLUS, we implement a hierarchy where:

-   A root Catalog contains all stations (each station is linked as a child).

-   Each station is a **Catalog** that groups multiple Collections for that station. Why multiple collections per station? Because if the station had a significant configuration change (like a different antenna pattern calibration or a long temporal gap), we treat data before vs after that change as separate collections (since their properties differ, e.g., Antenna Pattern ID or operational period).

-   Each **Collection** under a station covers a specific period/configuration and contains many Items (the individual time steps).

-   Items then link back to their Collection (and by extension, to the station catalog and to the root).

This structure ensures that an **Item always belongs to exactly one Collection**, and that the Collection carries the common metadata for those Items. Meanwhile, the station Catalog and root Catalog just serve to group and connect everything in a browseable tree.

## Catalog Overview

Our top-level **Catalog** (and intermediate station catalogs) follow the concept of **self-contained catalogs** and **relative links** for portability.

A \"self-contained\" STAC catalog is one where all links that tie the structure together (the links to parent, child, item, collection within the dataset) are **relative paths**, not absolute URLs. This means we can zip up the whole set of JSON files and move it somewhere else (or host it on a different domain) and all the references still work. This is great for scenarios like providing the catalog on physical media or as a downloadable package.

However, for a published online catalog, it\'s also useful to have a known entry point URL. We implement a **Relative Published Catalog** pattern, which is essentially a self-contained catalog with one addition: the root Catalog (and/or Collections) includes an absolute `self` link pointing to its canonical URL. For example, if the root catalog is intended to live at `https://hfeolus.example.com/catalog.json`, we put a self link with that URL in the root JSON. All other links (children, items, etc.) remain relative. This hybrid approach means:

-   If a user is browsing the files locally (or we move the files), the internal navigation works via relative links.

-   If a user knows the online endpoint, the self link advertises where the authoritative source is, and tools can use that to identify the catalog.

In practice, we keep our STAC implementation simple: we use JSON files on object storage (like AWS S3 or a web server) and ensure that for any given catalog or collection JSON, the `links` include entries for `child` (pointing to sub-catalogs or collections), `item` (pointing to item files), and `parent`/`root` as appropriate. Each item has a link to its parent (collection) and each collection has a link to root (the top catalog). This forms a complete graph for traversal.

## HF-EOLUS Specifications

Now we describe how the general STAC concepts are applied to the HF-EOLUS data, specifically focusing on the **HF-Radar Radial Metrics** portion of the project. (Meteorological model outputs could be cataloged in a similar fashion; for now, our emphasis is on the radar data as it presents the more complex use case with multiple assets per item and custom metadata.)

#### HF-Radar

For HF-Radar Radial Metrics, we define the following structure in STAC:

-   **Assets**: Each radar measurement interval produces multiple GeoParquet files as data assets. One interval yields two files for the **radial metrics** data: one for the **positive Bragg peak** echoes and one for the **negative Bragg peak** echoes (in HF radar processing, sea echo Doppler spectra have two main peaks -- approaching and receding waves, processed separately). In addition, each interval produces a small **header** file (containing global metadata from the original file) and a **range info** file (containing summary statistics by range cell). We store these Parquet files in three dedicated subdirectories under each station: `radial_metrics/`, `header/`, and `rng_info/`. This organization by data type keeps all Parquet files for a station grouped and also allows partitioning by attributes (for example, within `radial_metrics/` we further partition by `pos_bragg=0` vs `pos_bragg=1` for negative vs positive peak, and by timestamp). Using a consistent folder structure makes it easy to run queries across all files of a given type for a station (e.g., scanning all radial metrics files in a date range).

-   **Item**: Each STAC Item represents the data from **one time step** (e.g., one 30-minute period) for one station. However, because we have three different data components for each time (radial metrics, header, and range info), we actually create three Item JSONs per time step (one for each component) within the same Collection. The primary **radial metrics Item** (representing the main echo data) has two asset entries (e.g., `"neg_bragg_peak"` and `"pos_bragg_peak"`) pointing to the Parquet files for that interval. The Item\'s `properties.datetime` is set to the observation time (e.g., the start or center time of the interval), and its item geometry is typically a polygon representing the station\'s coverage area (with the `bbox` set to the bounding box of that polygon). However, **radial metrics assets** geometry column represent the location (point) of each echo.

In addition to the radial metrics item, we create a **header Item** for the same timestamp (containing the header metadata as its asset) and a **range info Item** for that timestamp (containing the range-wise summary asset). These three Items share the same `datetime` and station context, and we interlink them with STAC `links`: the radial metrics Item includes a `describedby` link to the header Item (and the header uses a reciprocal `describes` link back to the radial Item). The radial metrics Item also has a `related` link to the range info Item (and the range info Item links back with `related`). This linking ensures that a user can discover the auxiliary information (header or range summary) when looking at a radial metrics item, without cluttering the radial item\'s assets. Each Item has a unique `id`: we use the station code and timestamp as the base (e.g., `STATIONX_2023-01-01T12:00:00Z`), prefixed for the header and range info items (e.g., `header_STATIONX_2023-01-01T12:00:00Z`) to distinguish them from the main item.

-   **Collection**: We create one Collection per station **per continuous data period or configuration**. If a station\'s data is homogeneous (same configuration) over the whole archive, it will have a single Collection. If there was a major change (like a new antenna pattern calibration or a long gap), we split into multiple Collections (e.g., \"Station X 2015--2020\" vs \"Station X 2021--Present\"). Each Collection is given an `id` combining the station code and the date range (or config label), and includes metadata common to all items in that set. For example, the Collection\'s spatial extent covers the station\'s observation area and its temporal extent spans the first to last timestamps of that period. The Collection description includes the station name/ID and details such as the antenna pattern ID (if applicable). We also utilize the **STAC Table Extension** at the Collection level to document the schema of the tabular data: for instance, listing the columns present in the radial metrics files (bearing, range, velocity, etc.) along with units and descriptions. This allows users to understand the data structure without opening a file. The collection also specifies the data license, keywords (e.g., \"HF-Radar\", \"Ocean currents\"), and the list of data providers or contributors for that station. Each Item in the collection has a `collection` field referencing the Collection `id`. To help navigation, each collection has three sub-catalogs, one for each item type, containing the actual list of items. 

-   **Station Catalog**: For organizational clarity, each station has a Catalog (a JSON file named `catalog.json` in the station\'s directory) that groups that station\'s Collections. The station catalog has an `id` equal to the station code and links (with `rel: child`) to each Collection JSON of that station. It carries a short description (e.g., station location or name) but mainly serves as a container. If a station has only one Collection, the station catalog is somewhat optional, but we include it for consistency so that the root catalog can point to a station folder rather than directly to a collection file.

-   **Root Catalog**: At the top level, a root `catalog.json` lists every station as a child link. The root catalog provides a general description of the HF-EOLUS dataset and perhaps a link to project documentation or a DOI for the dataset collection as a whole. From the root, a user can navigate into a station, then into a collection, then down to items.

Concerning **naming conventions** for IDs:
-   Item `id` is set to the station and timestamp (e.g., `"STATIONX_2023-07-15T12:00:00Z"`) for radial metrics items. For header and range info items, we prepend a prefix in the ID (`"header_STATIONX_..."`, `"rnginfo_STATIONX_..."`) to ensure uniqueness within the catalog.

-   Collection `id` combines the station and a descriptor of the period or configuration. For example, `"STATIONX_2015-2020"` or `"STATIONX_2021-2023_apm2"` might be used to denote a station\'s data range or a second antenna pattern. The exact format can vary; in our examples we use a date range in the ID/title for clarity (e.g., `STATIONX_2015-01-01_to_2020-12-31`). Each collection has a human-readable title as well (which might include the station name and years).

-   Station catalog `id` is simply the station code (e.g., `"STATIONX"`).

With this organization, **discovery and use** of the data becomes straightforward:

-   A user can open the root catalog and see links to each station\'s data.

-   Navigating into a station folder, they see one or more Collections (each representing a distinct subset of that station\'s data). The station catalog links to these collections.

-   Each Collection JSON describes the dataset (station, time range, schema, etc.) and provides links to its Items. In a static catalog, we typically list Item links either directly in the Collection (if not too many), or rely on the consumer listing the files in an items directory. In our case, we provide sequential navigation via each Item\'s `links` (each Item links to the next and previous in time, rather than listing all in one file) to avoid huge lists.

Finally, we ensure that each Item\'s `properties` include any needed domain-specific metadata. For HF-Radar, this might include the antenna pattern identifier or certain processing parameters. These can be added as custom fields under properties or via a custom extension. For example, we include an antenna pattern unique identifier (as a dedicated property) and some filtering parameters (like max range and bearing sectors used) under a custom `radial_metrics:filters` field, as well as use the `table:row_count` field from the Table extension to indicate how many rows (echoes) are in each Parquet asset. By capturing such details, we make the Items self-explanatory to someone who finds them through a search interface.

In summary, the HF-EOLUS STAC implementation organizes the radial metrics and their associated metadata into a **navigable catalog** where each measurement interval yields a set of STAC Items (radial metrics, header, range summary) with appropriate links and common collection metadata. Collections group the data by station (and configuration period) for clarity, and all metadata needed to interpret and use the data (from file schema to spatial/temporal context) is provided in the STAC JSON. This approach ensures that researchers can discover and access the HF-Radar data using standard tools and integrate it with other STAC-indexed datasets as needed. The combination of **GeoParquet** for storage and **STAC** for metadata yields a powerful, standards-based framework for handling our geospatial data deluge.

## Catalog Structure and Examples

To make these concepts concrete, below we present an example static STAC catalog structure for HF-Radar data, including a root catalog, a station catalog, a collection, and some item examples.

**1. Root Catalog (**`catalog.json`**):** The root catalog provides a top-level entry point. It lists each station as a child link and gives a general description of the dataset collection.

    {
      "type": "Catalog",
      "id": "HF-EOLUS",
      "stac_version": "1.1.0",
      "description": "HF-EOLUS HF-Radar Radial Metrics Catalog - Root",
      "links": [
        {
          "rel": "self",
          "href": "./catalog.json",
          "type": "application/json",
          "title": "HF-EOLUS HF-Radar Root Catalog"
        },
        {
          "rel": "child",
          "href": "VILA/catalog.json",
          "type": "application/json",
          "title": "VILA (Station VILA) Collections"
        },
        {
          "rel": "child",
          "href": "PRIO/catalog.json",
          "type": "application/json",
          "title": "PRIO (Station PRIO) Collections"
        }
      ]
    }

Here, the root catalog (with `id`: \"HF-EOLUS\") links to two station catalogs (VILA and PRIO) as children. Each station is identified by its code and has an associated `catalog.json` in its directory. The root catalog\'s description provides a brief context, and it has a `self` link (to itself) and `child` links for each station.

**2. Station Catalog (**`VILA/catalog.json`**):** Each station folder contains a catalog that groups that station's collections.

    {
      "type": "Catalog",
      "id": "VILA",
      "stac_version": "1.1.0",
      "description": "Station VILA - HF-Radar site collections",
      "links": [
        {
          "rel": "self",
          "href": "./catalog.json",
          "type": "application/json",
          "title": "VILA Station Catalog"
        },
        {
          "rel": "parent",
          "href": "../catalog.json",
          "type": "application/json",
          "title": "HF-EOLUS HF-Radar Root Catalog"
        },
        {
          "rel": "child",
          "href": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00/collection.json",
          "type": "application/json",
          "title": "VILA 2018-06-21T17:30:00Z to 2018-06-30T23:30:00Z Collection"
        },
        {
          "rel": "child",
          "href": "VILA_2011-08-04T00:00:00_2018-06-21T17:00:00/collection.json",
          "type": "application/json",
          "title": "VILA 2011-08-04T00:00:00Z to 2018-06-21T17:00:00Z Collection"
        }
      ]
    }

In this example, station **VILA** has two Collections (splitting its data into two time ranges). The station catalog\'s `links` include a `parent` (pointing back to root) and two `child` links for each collection JSON. The station catalog itself primarily serves as a directory for the station's collections.

**3. Collection (**`VILA_2018-06-21T17:30:00_2018-06-30T23:30:00/collection.json`**):** Each Collection contains metadata describing the dataset and provides links to its sub-catalogs of Items. We illustrate one such Collection JSON below:

\<!\-- \--\>

    {
      "type": "Collection",
      "id": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00",
      "stac_version": "1.1.0",
      "description": "Radial Metrics for station VILA from 2018-06-21T17:30:00Z to 2018-06-30T23:30:00Z within bounding box [-11.0283039, 42.0911473, -8.0724451, 44.4891614].",
      "links": [
        {
          "rel": "child",
          "href": "items/radial_metrics/catalog.json",
          "type": "application/json",
          "title": "Radial Metrics Items"
        },
        {
          "rel": "child",
          "href": "items/header/catalog.json",
          "type": "application/json",
          "title": "Header Items"
        },
        {
          "rel": "child",
          "href": "items/rng_info/catalog.json",
          "type": "application/json",
          "title": "Range Info Items"
        }
      ],
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "table:tables": [
        {
          "name": "radial_metrics",
          "description": "Radial metrics for station VILA"
        },
        {
          "name": "header",
          "description": "Header metadata for station VILA"
        },
        {
          "name": "rng_info",
          "description": "Range info for station VILA"
        }
      ],
      "pattern_uui": [
        "5C42DF92-DBC9-4F39-83E3-969A69A12AAC"
      ],
      "hf_site:location": [
        -9.2108333,
        43.1588833
      ],
      "radial_metrics:filters": {
        "max_range": 150.0,
        "bearing_ranges": [
          [
            0.0,
            38.0
          ],
          [
            217.0,
            360.0
          ]
        ]
      },
      "sci:doi": "10.5281/zenodo.16892223",
      "sci:citation": "Herrera Cortijo, J. L., Fernández-Baladrón, A., Rosón, G., Gil Coto, M., Dubert, J., & Varela Benvenuto, R. (2025). Project HF-EOLUS. Task 2. VILA & PRIO Radial Metrics GeoParquet Dataset. Derived from LLUV datasets and ingested in GeoParquet format. Zenodo. https://doi.org/10.5281/zenodo.16892223",
      "providers": [
        {
          "name": "Instituto Tecnolóxico para o Control do Medio Mariño de Galicia (INTECMAR)",
          "description": "Spectra from INTECMAR's VILA and PRIO HF-Radar stations, between 2011-08-04 and 2023-11-23 have been transferred free of charge by the Observatorio Costeiro da Xunta de Galicia (<https://www.observatoriocosteiro.gal>) for their use. This Observatory is not responsible for the use of these data nor is it linked to the conclusions drawn with them. The Costeiro da Xunta de Galicia Observatory is part of the RAIA Observatory (<http://www.marnaraia.org>).",
          "roles": [
            "producer"
          ],
          "url": "https://intecmar.gal"
        },
        {
          "name": "Grupo de Oceanografía Física de la Universidade de Vigo (GOFUV)",
          "description": "The spectra data have been processed into Radial Metrics and then ingested into these geoparquet files by the Grupo de Oceanografía Física de la Universidade de Vigo (GOFUV) as part of the HF-EOLUS project (TED2021-129551B-I00). HF-EOLUS was financed by MICIU/AEI /10.13039/501100011033 and by the European Union NextGenerationEU/PRTR - BDNS 598843 - Component 17 - Investment I3. Members of the Marine Research Centre (CIM) of the University of Vigo have participated in the development of this repository. Processing details can be found at https://doi.org/10.5281/zenodo.16679609, ingestion details at https://doi.org/10.5281/zenodo.16679610, and the data at https://doi.org/10.5281/zenodo.16892223.",
          "roles": [
            "processor"
          ],
          "url": "https://gofuvi.org"
        }
      ],
      "title": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00 - Radial Metrics",
      "extent": {
        "spatial": {
          "bbox": [
            [
              -11.0283039,
              42.0911473,
              -8.0724451,
              44.4891614
            ]
          ]
        },
        "temporal": {
          "interval": [
            [
              "2018-06-21T17:30:00Z",
              "2018-06-30T23:30:00Z"
            ]
          ]
        }
      },
      "license": "GPL-3.0",
      "keywords": [
        "HF-Radar",
        "Radial Metrics",
        "VILA"
      ]
    }

In this Collection JSON, we include fields like `extent` (with the spatial bounding box and temporal range covering all data in the collection), `providers` (organizations responsible for producing and processing the data), and references to the Table extension. Instead of listing every column in a `summaries` section, this Collection uses a `table:tables` array to enumerate the three tables (radial\_metrics, header, rng\_info) that are part of the dataset. The `links` section provides `child` links to sub-catalogs for each item type (radial metrics data, header data, and range info data) rather than linking directly to individual item files. The Collection's `description`, `title`, and custom properties (`hf_site:location`, `radial_metrics:filters`, etc.) document station-specific context like location and filtering parameters applied. Each Item belonging to this Collection will have a `collection` field referencing this Collection\'s `id`.

**4. Item Examples (**`items/*.json`**):** Within the Collection's `items/` subdirectory, each time step produces three Item files in our setup: one for the radial metrics, one for the header, and one for the range info. Below we show an example of each type of Item JSON (using station VILA at a particular timestamp):

-   **Radial Metrics Item (primary data at a given timestamp):**

\<!\-- \--\>

    {
      "type": "Feature",
      "stac_version": "1.1.0",
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "id": "VILA_2018-06-21T18:00:00",
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [
              -8.0724451,
              42.1192621
            ],
            [
              -8.0724451,
              44.4891614
            ],
            [
              -11.0281618,
              44.4891614
            ],
            [
              -11.0281618,
              42.1192621
            ],
            [
              -8.0724451,
              42.1192621
            ]
          ]
        ]
      },
      "bbox": [
        -11.0281618,
        42.1192621,
        -8.0724451,
        44.4891614
      ],
      "properties": {
        "hf_site:code": "VILA",
        "hf_site:location": [
          -9.2108333,
          43.1588833
        ],
        "radial_metrics:filters": {
          "max_range": 150.0,
          "bearing_ranges": [
            [
              0.0,
              38.0
            ],
            [
              217.0,
              360.0
            ]
          ]
        },
        "table:primary_geometry": "geometry",
        "table:columns": [
          {
            "name": "rowid",
            "description": "Unique row identifier",
            "type": "integer"
          },
          {
            "name": "timestamp",
            "description": "Date and time of observation (UTC)",
            "type": "datetime"
          },
          {
            "name": "Pwr",
            "description": "Signal power of the selected solution (dB)",
            "type": "number"
          },
          {
            "name": "VELO",
            "description": "Radial velocity (cm/s)",
            "type": "number"
          },
          {
            "name": "pos_bragg",
            "description": "Indicator of Bragg peak partition (0 = negative, 1 = positive)",
            "type": "string"
          },
          {
            "name": "geometry",
            "description": "Geometry of the measurement in WGS84 coordinate reference system encoded as WKB",
            "type": "binary"
          }
        ],
        "sci:doi": "10.5281/zenodo.16892223",
        "sci:citation": "Herrera Cortijo, J. L., Fernández-Baladrón, A., Rosón, G., Gil Coto, M., Dubert, J., & Varela Benvenuto, R. (2025). Project HF-EOLUS. Task 2. VILA & PRIO Radial Metrics GeoParquet Dataset. Derived from LLUV datasets and ingested in GeoParquet format. Zenodo. https://doi.org/10.5281/zenodo.16892223",
        "providers": [
          {
            "name": "Instituto Tecnolóxico para o Control do Medio Mariño de Galicia (INTECMAR)",
            "description": "Spectra from INTECMAR's VILA and PRIO HF-Radar stations, between 2011-08-04 and 2023-11-23 have been transferred free of charge by the Observatorio Costeiro da Xunta de Galicia (<https://www.observatoriocosteiro.gal>) for their use. This Observatory is not responsible for the use of these data nor is it linked to the conclusions drawn with them. The Costeiro da Xunta de Galicia Observatory is part of the RAIA Observatory (<http://www.marnaraia.org>).",
            "roles": [
              "producer"
            ],
            "url": "https://intecmar.gal"
          },
          {
            "name": "Grupo de Oceanografía Física de la Universidade de Vigo (GOFUV)",
            "description": "The spectra data have been processed into Radial Metrics and then ingested into these geoparquet files by the Grupo de Oceanografía Física de la Universidade de Vigo (GOFUV) as part of the HF-EOLUS project (TED2021-129551B-I00). HF-EOLUS was financed by MICIU/AEI /10.13039/501100011033 and by the European Union NextGenerationEU/PRTR - BDNS 598843 - Component 17 - Investment I3. Members of the Marine Research Centre (CIM) of the University of Vigo have participated in the development of this repository. Processing details can be found at https://doi.org/10.5281/zenodo.16679609, ingestion details at https://doi.org/10.5281/zenodo.16679610, and the data at https://doi.org/10.5281/zenodo.16892223.",
            "roles": [
              "processor"
            ],
            "url": "https://gofuvi.org"
          }
        ],
        "datetime": "2018-06-21T18:00:00Z",
        "pattern_uui": "5C42DF92-DBC9-4F39-83E3-969A69A12AAC",
        "table:row_count": 3034
      },
      "links": [
        {
          "rel": "collection",
          "href": "../../collection.json",
          "type": "application/json",
          "title": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00"
        },
        {
          "rel": "describedby",
          "href": "../header/header_VILA_2018-06-21T18:00:00.json",
          "type": "application/geo+json",
          "title": "header_VILA_2018-06-21T18:00:00"
        },
        {
          "rel": "related",
          "href": "../rng_info/rng_info_VILA_2018-06-21T18:00:00.json",
          "type": "application/geo+json",
          "title": "rng_info_VILA_2018-06-21T18:00:00"
        },
        {
          "rel": "next",
          "href": "./radial_metrics_VILA_2018-06-21T18:30:00.json",
          "type": "application/geo+json",
          "title": "radial_metrics_VILA_2018-06-21T18:30:00"
        },
        {
          "rel": "prev",
          "href": "./radial_metrics_VILA_2018-06-21T17:30:00.json",
          "type": "application/geo+json",
          "title": "radial_metrics_VILA_2018-06-21T17:30:00"
        }
      ],
      "assets": {
        "neg_bragg_peak": {
          "href": "../../../radial_metrics/pos_bragg=0/timestamp=2018-06-21T18%3A00%3A00/VILA_2018-06-21T18:00:00_0.parquet",
          "type": "application/x-parquet",
          "title": "Radial Metrics - Negative Bragg Peak",
          "roles": [
            "data"
          ]
        },
        "pos_bragg_peak": {
          "href": "../../../radial_metrics/pos_bragg=1/timestamp=2018-06-21T18%3A00%3A00/VILA_2018-06-21T18:00:00_0.parquet",
          "type": "application/x-parquet",
          "title": "Radial Metrics - Positive Bragg Peak",
          "roles": [
            "data"
          ]
        }
      },
      "collection": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00"
    }

This **radial metrics item** has two assets (negative and positive Bragg peak Parquet files). The item `geometry` is a polygon approximating the station's coverage area, and the `bbox` is provided accordingly. The properties include some station info and filtering parameters used (e.g., the maximum range and bearing sectors) and a snippet of the table schema (via `table:columns`) for clarity. We include `table:row_count` to indicate how many rows (echoes) are in this time\'s data. The `links` section links to the parent collection and connects this item to its corresponding header and range info items (via `describedby` and `related` links). It also has `prev` and `next` links to the adjacent radial metrics items in time for easy navigation. *Note:* In the table schema above, the `geometry` column contains the point location of each individual radar echo, whereas the Item's own `geometry` covers the entire measurement area.

-   **Header Item (metadata for the same timestamp):**

\<!\-- \--\>

    {
      "type": "Feature",
      "stac_version": "1.1.0",
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "id": "header_VILA_2018-06-21T18:00:00",
      "geometry": {
        "type": "Point",
        "coordinates": [
          -9.2108333,
          43.1588833
        ]
      },
      "bbox": [
        -9.2108333,
        43.1588833,
        -9.2108333,
        43.1588833
      ],
      "properties": {
        "hf_site:code": "VILA",
        "hf_site:location": [
          -9.2108333,
          43.1588833
        ],
        "table:primary_geometry": "geometry",
        "table:columns": [
          {
            "name": "rowid",
            "description": "Unique row identifier",
            "type": "integer"
          },
          {
            "name": "timestamp",
            "description": "Date and time of metadata entry (UTC)",
            "type": "datetime"
          },
          {
            "name": "metakey",
            "description": "Metadata key",
            "type": "string"
          },
          {
            "name": "metavalue",
            "description": "Metadata value",
            "type": "string"
          },
          {
            "name": "geometry",
            "description": "Station location geometry encoded as WKB",
            "type": "binary"
          }
        ],
        "table:row_count": 37,
        "sci:doi": "10.5281/zenodo.16892223",
        "sci:citation": "Herrera Cortijo, J. L., Fernández-Baladrón, A., Rosón, G., Gil Coto, M., Dubert, J., & Varela Benvenuto, R. (2025). Project HF-EOLUS. Task 2. VILA & PRIO Radial Metrics GeoParquet Dataset. Derived from LLUV datasets and ingested in GeoParquet format. Zenodo. https://doi.org/10.5281/zenodo.16892223",
        "providers": [
          {
            "name": "Instituto Tecnolóxico para o Control do Medio Mariño de Galicia (INTECMAR)",
            "roles": [
              "producer"
            ],
            "url": "https://intecmar.gal"
          },
          {
            "name": "Grupo de Oceanografía Física de la Universidade de Vigo (GOFUV)",
            "roles": [
              "processor"
            ],
            "url": "https://gofuvi.org"
          }
        ],
        "datetime": "2018-06-21T18:00:00Z"
      },
      "links": [
        {
          "rel": "collection",
          "href": "../../collection.json",
          "type": "application/json",
          "title": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00"
        },
        {
          "rel": "describes",
          "href": "../radial_metrics/radial_metrics_VILA_2018-06-21T18:00:00.json",
          "type": "application/geo+json",
          "title": "radial_metrics_VILA_2018-06-21T18:00:00"
        },
        {
          "rel": "prev",
          "href": "./header_VILA_2018-06-21T17:30:00.json",
          "type": "application/geo+json",
          "title": "header_VILA_2018-06-21T17:30:00"
        },
        {
          "rel": "next",
          "href": "./header_VILA_2018-06-21T18:30:00.json",
          "type": "application/geo+json",
          "title": "header_VILA_2018-06-21T18:30:00"
        }
      ],
      "assets": {
        "header": {
          "href": "../../../header/timestamp=2018-06-21T18%3A00%3A00/VILA_2018-06-21T18:00:00_0.parquet",
          "type": "application/x-parquet",
          "title": "LLUV Header",
          "description": "LLUV file header",
          "roles": [
            "data"
          ]
        }
      },
      "collection": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00"
    }

The **header item** contains the metadata records from the original radar file (essentially a list of key--value settings and diagnostics). We represent it as a point at the station\'s location (since the header information isn't spatially extensive). Its properties include the station identifier (repeated for consistency) and a small table schema (columns for timestamp and key/value pairs describing each metadata entry). It links back to the radial metrics item via a `describes` relationship (the radial metrics item had a corresponding `describedby` link to this header). The asset is the Parquet file containing the header information. Note that `id` here is prefixed with `"header_"` to distinguish it. Also, because all header entries pertain to the station itself, the table's `geometry` column is just the station's point location (the same as the Item's geometry).

-   **Range Info Item (summary statistics for the same timestamp):**

\<!\-- \--\>

    {
      "type": "Feature",
      "stac_version": "1.1.0",
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "id": "rng_info_VILA_2018-06-21T18:00:00",
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [
              -8.0724451,
              42.1192621
            ],
            [
              -8.0724451,
              44.4891614
            ],
            [
              -11.0281618,
              44.4891614
            ],
            [
              -11.0281618,
              42.1192621
            ],
            [
              -8.0724451,
              42.1192621
            ]
          ]
        ]
      },
      "bbox": [
        -11.0281618,
        42.1192621,
        -8.0724451,
        44.4891614
      ],
      "properties": {
        "hf_site:code": "VILA",
        "hf_site:location": [
          -9.2108333,
          43.1588833
        ],
        "table:primary_geometry": "geometry",
        "table:columns": [
          {
            "name": "rowid",
            "description": "Unique row identifier",
            "type": "integer"
          },
          {
            "name": "timestamp",
            "description": "Date and time of observation (UTC)",
            "type": "datetime"
          },
          {
            "name": "SPRC",
            "description": "Range cell index (integer)",
            "type": "integer"
          },
          {
            "name": "RNGC",
            "description": "Range cell range (km)",
            "type": "number"
          },
          {
            "name": "NF01",
            "description": "Noise floor for loop 1 (db)",
            "type": "number"
          },
          {
            "name": "NF02",
            "description": "Noise floor for loop 2 (db)",
            "type": "number"
          },
          {
            "name": "NF03",
            "description": "Noise floor for monopole (db)",
            "type": "number"
          },
          {
            "name": "ALM1",
            "description": "Doppler cell index for left first-order limit, negative Bragg peak (integer)",
            "type": "integer"
          },
          {
            "name": "ALM2",
            "description": "Doppler cell index for right first-order limit, negative Bragg peak (integer)",
            "type": "integer"
          },
          {
            "name": "ALM3",
            "description": "Doppler cell index for left first-order limit, positive Bragg peak (integer)",
            "type": "integer"
          },
          {
            "name": "ALM4",
            "description": "Doppler cell index for right first-order limit, positive Bragg peak (integer)",
            "type": "integer"
          },
          {
            "name": "NVSC",
            "description": "Number of MUSIC single solutions in the cell range (integer)",
            "type": "integer"
          },
          {
            "name": "NVDC",
            "description": "Number of MUSIC dual solutions in the cell range (integer)",
            "type": "integer"
          },
          {
            "name": "NVAC",
            "description": "Total number of radial vectors in the cell range (integer)",
            "type": "integer"
          },
          {
            "name": "geometry",
            "description": "Aggregated radial geometry encoded as WKB",
            "type": "binary"
          }
        ],
        "table:row_count": 29,
        "sci:doi": "10.5281/zenodo.16892223",
        "sci:citation": "Herrera Cortijo, J. L., Fernández-Baladrón, A., Rosón, G., Gil Coto, M., Dubert, J., & Varela Benvenuto, R. (2025). Project HF-EOLUS. Task 2. VILA & PRIO Radial Metrics GeoParquet Dataset. Derived from LLUV datasets and ingested in GeoParquet format. Zenodo. https://doi.org/10.5281/zenodo.16892223",
        "providers": [
          {
            "name": "Instituto Tecnolóxico para o Control do Medio Mariño de Galicia (INTECMAR)",
            "roles": [
              "producer"
            ],
            "url": "https://intecmar.gal"
          },
          {
            "name": "Grupo de Oceanografía Física de la Universidade de Vigo (GOFUV)",
            "roles": [
              "processor"
            ],
            "url": "https://gofuvi.org"
          }
        ],
        "datetime": "2018-06-21T18:00:00Z"
      },
      "links": [
        {
          "rel": "collection",
          "href": "../../collection.json",
          "type": "application/json",
          "title": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00"
        },
        {
          "rel": "related",
          "href": "../radial_metrics/radial_metrics_VILA_2018-06-21T18:00:00.json",
          "type": "application/geo+json",
          "title": "radial_metrics_VILA_2018-06-21T18:00:00"
        },
        {
          "rel": "prev",
          "href": "./rng_info_VILA_2018-06-21T17:30:00.json",
          "type": "application/geo+json",
          "title": "rng_info_VILA_2018-06-21T17:30:00"
        },
        {
          "rel": "next",
          "href": "./rng_info_VILA_2018-06-21T18:30:00.json",
          "type": "application/geo+json",
          "title": "rng_info_VILA_2018-06-21T18:30:00"
        }
      ],
      "assets": {
        "rng_info": {
          "href": "../../../rng_info/timestamp=2018-06-21T18%3A00%3A00/VILA_2018-06-21T18:00:00_0.parquet",
          "type": "application/x-parquet",
          "title": "Range Info",
          "description": "Radial metrics stats summary by range cell",
          "roles": [
            "data"
          ]
        }
      },
      "collection": "VILA_2018-06-21T17:30:00_2018-06-30T23:30:00"
    }

The **range info item** provides summary metrics for each range cell (distance bin) at the given time. We use a polygon approximating the full coverage area of the radar as the Item\'s geometry (similar to the radial metrics item). The properties include the station info and the table schema (fields such as range cell index, range distance in km, counts of echoes, and other diagnostics). The `rel: related` link points to the main radial metrics item. The asset is the Parquet file containing the range-wise summary table, and the `id` is prefixed with `"rng_info_"` to distinguish it. Within the range info table, each row's `geometry` is a line representing the arc of that particular range cell (so the Item's polygon covers the whole area, while each data row has a line geometry for its range segment).

All three items share the same `datetime` timestamp and belong to the same Collection (`VILA_2018-06-21T17:30:00_2018-06-30T23:30:00`). Through the interlinking `links`, one can move between the main data and its ancillary info, or iterate over time using the `prev`/`next` links within each series of items (radial metrics series, header series, range info series).

This example demonstrates how the STAC files are structured and interlinked for our HF-Radar radial metrics. In practice, a user starting at the root catalog can drill down to a station, pick a collection, then find items of interest by time. Once a particular Item (e.g., a radial metrics item) is found, its links lead to the associated header and range info items for complete context. Because we adhere to STAC and its extensions, the entire catalog can be loaded with standard libraries, enabling programmatic search and retrieval of HF-Radar data in the HF-EOLUS project.

## Column Definitions for Radial Metrics, Header, and Range Info Items

All HF-Radar data assets are stored as tabular GeoParquet files, each with a defined schema of columns. The radial metrics dataset has a comprehensive set of possible columns (depending on processing; for instance, certain columns appear only if dual-solution processing is used). In practice, a given radial metrics file may contain only a subset of these columns, but it will always include essential fields like `rowid`, `timestamp`, `geometry`, and `pos_bragg` (the latter two are used as partition keys). The header and range info assets have fixed schemas that include all the columns listed for those item types. Below, we list and describe all columns for each item type:

### Radial Metrics Data Columns

  Column           Description
  ---------------- ---------------------------------------------------------------------------------
  **Pwr**          Signal power of the selected solution (dB)
  **LOND**         Longitude and latitude (decimal degrees)
  **LATD**         Longitude and latitude (decimal degrees)
  **VELU**         East and north components of velocity (cm/s)
  **VELV**         East and north components of velocity (cm/s)
  **VFLG**         Vector validity
  **RNGE**         Distance from the antenna (km)
  **BEAR**         Bearing of the vector (counter-clockwise from true north)
  **VELO**         Radial velocity (cm/s)
  **HEAD**         Velocity direction (counter-clockwise from true north)
  **SPRC**         Range cell
  **SPDC**         Doppler bin
  **MSEL**         Selected solution (1 = single, 2 = dual 1, 3 = dual 2)
  **MSA1**         Bearing of the single solution (1440 if invalid)
  **MDA1**         Bearing of the first dual solution (1440 if invalid)
  **MDA2**         Bearing of the second dual solution (1440 if invalid)
  **MEGR**         Ratio of the first and second eigenvalue
  **MPKR**         Signal power ratio
  **MOFR**         Off-diagonal ratio
  **MP13**         Phase between antennas 1 and 3
  **MP23**         Phase between antennas 2 and 3
  **MSP1**         Signal power of the single solution (dB)
  **MDP1**         Signal power of the first dual solution (dB)
  **MDP2**         Signal power of the second dual solution (dB)
  **MSW1**         3-dB width of the DOA function below the peak for the single solution
  **MDW1**         3-dB width of the DOA function below the peak for the first dual solution
  **MDW2**         3-dB width of the DOA function below the peak for the second dual solution
  **MSR1**         Value of DOA function at the peak of the single solution
  **MDR1**         Value of DOA function at the peak of the first dual solution
  **MDR2**         Value of DOA function at the peak of the second dual solution
  **MA1S**         SNR of the self-spectrum of antenna 1 for this Doppler bin
  **MA2S**         SNR of the self-spectrum of antenna 2 for this Doppler bin
  **MA3S**         SNR of the self-spectrum of antenna 3 for this Doppler bin
  **MEI1**         First eigenvalue of the covariance matrix
  **MEI2**         Second eigenvalue of the covariance matrix
  **MEI3**         Third eigenvalue of the covariance matrix
  **MDRJ**         Reason for dual-solution rejection
  **PPFG**         Flags for QARTOD test 103 (1 = pass, 4 = fail)
  **PWFG**         Flags for QARTOD test 104 (1 = pass, 4 = fail)
  **rowid**        Unique row identifier
  **timestamp**    Date and time of observation (UTC)
  **geometry**     Geometry of the measurement in WGS84 coordinate reference system encoded as WKB
  **pos\_bragg**   Indicator of Bragg peak partition (0 = negative, 1 = positive)

### Header Data Columns

  Column          Description
  --------------- ------------------------------------------
  **rowid**       Unique row identifier
  **timestamp**   Date and time of metadata entry (UTC)
  **metakey**     Metadata key
  **metavalue**   Metadata value
  **geometry**    Station location geometry encoded as WKB

### Range Info Data Columns

  Column          Description
  --------------- -------------------------------------------------------------------------------
  **rowid**       Unique row identifier
  **timestamp**   Date and time of observation (UTC)
  **SPRC**        Range cell index (integer)
  **RNGC**        Range cell range (km)
  **NF01**        Noise floor for loop 1 (db)
  **NF02**        Noise floor for loop 2 (db)
  **NF03**        Noise floor for monopole (db)
  **ALM1**        Doppler cell index for left first-order limit, negative Bragg peak (integer)
  **ALM2**        Doppler cell index for right first-order limit, negative Bragg peak (integer)
  **ALM3**        Doppler cell index for left first-order limit, positive Bragg peak (integer)
  **ALM4**        Doppler cell index for right first-order limit, positive Bragg peak (integer)
  **NVSC**        Number of MUSIC single solutions in the cell range (integer)
  **NVDC**        Number of MUSIC dual solutions in the cell range (integer)
  **NVAC**        Total number of radial vectors in the cell range (integer)
  **geometry**    Aggregated radial geometry encoded as WKB

## References

<ol> 
<li id="ref1">STAC Contributors. (2024). SpatioTemporal Asset Catalog (STAC) specification (Version 1.1.0). https://stacspec.org</li\> 
</ol>
