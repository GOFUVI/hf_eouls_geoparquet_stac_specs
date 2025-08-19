# STAC Specification

## STAC Overview

The **SpatioTemporal Asset Catalog (STAC)** is a standard for describing geospatial datasets with metadata that is both **machine-readable** and **web-friendly**. STAC defines a common structure for cataloging data like satellite imagery, maps, point clouds, etc., so that these assets can be easily indexed and discovered by location, time, and other properties. The core STAC specification is intentionally minimal \-- it focuses on the essentials needed to locate data in space and time \-- and it relies on a flexible extension mechanism to accommodate domain-specific needs. This design allows STAC to be widely applicable: different projects can extend it (adding additional fields or semantics) without breaking the fundamental compatibility. For a full description of the STAC specifications, please see
[\[1\]](#ref1).

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

-   And it can have keywords, a longer description, etc.

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

-   **Assets**: Each radar measurement interval produces multiple GeoParquet files as data assets. One interval yields two files for the **radial metrics** data: one for the **positive Bragg peak** echoes and one for the **negative Bragg peak** echoes (in HF radar processing, sea echo Doppler spectra have two main peaks \-- approaching and receding waves, processed separately). In addition, each interval produces a small **header** file (containing global metadata from the original file) and a **range info** file (containing summary statistics by range cell). We store these Parquet files in three dedicated subdirectories under each station: `radial_metrics/`, `header/`, and `rng_info/`. This organization by data type keeps all Parquet files for a station grouped and also allows partitioning by attributes (for example, within `radial_metrics/` we further partition by `pos_bragg=0` vs `pos_bragg=1` for negative vs positive peak, and by timestamp)[\[1\]][][\[2\]]. Using a consistent folder structure makes it easy to run queries across all files of a given type for a station (e.g., scanning all radial metrics files in a date range).

-   **Item**: Each STAC Item represents the data from **one time step** (e.g., one 30-minute period) for one station. However, because we have three different data components for each time (radial metrics, header, and range info), we actually create three Item JSONs per time step (one for each component) within the same Collection. The primary **radial metrics Item** (representing the main echo data) has two asset entries (e.g., `"neg_bragg_peak"` and `"pos_bragg_peak"`) pointing to the Parquet files for that interval[\[2\]]. The Item\'s `properties.datetime` is set to the observation time (e.g., the start or center time of the interval), and its geometry is typically a point or small polygon representing the station location or coverage area. In our implementation, we use a representative polygon or point near the station for the geometry and a bounding box covering the station\'s coverage area for the `bbox`.

In addition to the radial metrics item, we create a **header Item** for the same timestamp (containing the header metadata as its asset) and a **range info Item** for that timestamp (containing the range-wise summary asset). These three Items share the same `datetime` and station context, and we interlink them with STAC `links`: the radial metrics Item includes a `describedby` link to the header Item (and the header uses a reciprocal `describes` link back to the radial Item)[\[3\]][][\[4\]]. The radial metrics Item also has a `related` link to the range info Item (and the range info Item links back with `related`)[\[5\]]. This linking ensures that a user can discover the auxiliary information (header or range summary) when looking at a radial metrics item, without cluttering the radial item's assets. Each Item has a unique `id`: we use the station code and timestamp as the base (e.g., `STATIONX_20230101T120000Z`), prefixed for the header and range info items (e.g., `header_STATIONX_20230101T120000Z`) to distinguish them from the main item.

-   **Collection**: We create one Collection per station **per continuous data period or configuration**. If a station's data is homogeneous (same configuration) over the whole archive, it will have a single Collection. If there was a major change (like a new antenna pattern calibration or a long gap), we split into multiple Collections (e.g., \"Station X 2015--2020\" vs \"Station X 2021--Present\"). Each Collection is given an `id` combining the station code and the date range (or config label), and includes metadata common to all items in that set. For example, the Collection's spatial extent covers the station's observation area and its temporal extent spans the first to last timestamps of that period. The Collection description includes the station name/ID and details like the frequency, antenna pattern ID (if applicable), etc. We also utilize the **STAC Table Extension** at the Collection level to document the schema of the tabular data: for instance, listing the columns present in the radial metrics files (bearing, range, velocity, etc.) along with units and descriptions. This allows users to understand the data structure without opening a file. (In cases with multiple item types in one collection, we can include schema information for each type of asset in the collection documentation.) The collection also specifies the data license, keywords (e.g., \"HF-Radar\", \"Ocean currents\"), and the list of data providers or contributors for that station. Each Item in the collection has a `collection` field referencing the Collection `id`.

-   **Station Catalog**: For organizational clarity, each station has a Catalog (a JSON file named `catalog.json` in the station's directory) that groups that station's Collections. The station catalog has an `id` equal to the station code and links (with `rel: child`) to each Collection JSON of that station. It carries a short description (e.g., station location or name) but mainly serves as a container. If a station has only one Collection, the station catalog is somewhat optional, but we include it for consistency so that the root catalog can point to a station folder rather than directly to a collection file.

-   **Root Catalog**: At the top level, a root `catalog.json` lists every station as a child link. The root catalog provides a general description of the HF-EOLUS dataset and perhaps a link to project documentation or a DOI for the dataset collection as a whole. From the root, a user can navigate into a station, then into a collection, then down to items.

Concerning **naming conventions** for files and IDs:

-   Each Parquet data file and corresponding STAC Item file is named consistently. We use the pattern `{station_id}_{timestamp}` for the core identifier. For example, for station **STATIONX** at time **2023-07-15 12:00 UTC**, the radial metrics item JSON might be `STATIONX_20230715T120000Z.json` (prefixed with the data type if not the main radial data). The Parquet files have a similar name plus an indicator of the content (we use a numeric or shorthand tag for the Bragg peak or data type). For instance, `STATIONX_20230715T120000Z_0.parquet` might store the negative Bragg peak radial metrics, and `STATIONX_20230715T120000Z_1.parquet` the positive peak. The header file could be `STATIONX_20230715T120000Z_header.parquet` (or a similar scheme), and the range info file `STATIONX_20230715T120000Z_rng.parquet`. In our implementation, we actually organize these into subfolders (as mentioned above) so the full path conveys the type (e.g., `.../radial_metrics/pos_bragg=1/timestamp=20230715T120000Z/STATIONX_20230715T120000Z_1.parquet`). This also means the same base filename can be used in different subfolders without confusion.

-   Item `id` is set to the station and timestamp (e.g., `"STATIONX_20230715T120000Z"`) for radial metrics items. For header and range info items, we prepend a prefix in the ID (`"header_STATIONX_..."`, `"rnginfo_STATIONX_..."`) to ensure uniqueness within the catalog.

-   Collection `id` combines the station and a descriptor of the period or configuration, for example: `"STATIONX_2015-2020"` or `"STATIONX_2021-2023_apm2"` if needed to denote a second antenna pattern. The exact format can vary; in our examples we use a date range in the ID/title for clarity (e.g., `STATIONX_2015-01-01_to_2020-12-31`). Each collection has a human-readable title as well (which might include the station name and years).

-   Station catalog `id` is simply the station code (e.g., `"STATIONX"`).

With this organization, **discovery and use** of the data becomes straightforward:

-   A user can open the root catalog and see links to each station's data.

-   Navigating into a station folder, they see one or more collections (each representing a distinct subset of that station's data). The station catalog links to these collections.

-   Each collection JSON describes the dataset (station, time range, schema, etc.) and provides links to its Items. In a static catalog, we typically list Item links either directly in the collection (if not too many), or rely on the consumer listing the files in the items directory. In our case, we provide sequential navigation via each Item's `links` (each Item links to the next and previous in time, rather than listing all in one file) to avoid huge lists.

-   Because the files (and item IDs) are named by station and timestamp, it's easy to locate a specific time or perform a time range scan by looking at filenames. Additionally, using STAC-aware tools, one can search the catalog by `datetime` or other properties to find relevant Items without manually navigating the folders.

Finally, we ensure that each Item's `properties` include any needed domain-specific metadata. For HF-Radar, this might include: the radar frequency, the antenna pattern identifier, processing parameters or quality flags, etc. These can be added as custom fields under properties or via a custom extension. For example, we include a property for the antenna pattern unique ID and some filtering parameters (like max range and bearing sectors used) under a namespace (`radial_metrics:filters`), as well as use the `table:row_count` field from the Table extension to indicate how many rows (echoes) are in each Parquet asset. By capturing such details, we make the Items self-explanatory to someone who finds them through a search interface.

In summary, the HF-EOLUS STAC implementation organizes the radial metrics and their associated metadata into a **navigable catalog** where each measurement interval yields a set of STAC Items (radial metrics, header, range summary) with appropriate links and common collection metadata. Collections group the data by station (and configuration period) for clarity, and all metadata needed to interpret and use the data (from file schema to spatial/temporal context) is provided in the STAC JSON. This approach ensures that researchers can discover and access the HF-Radar data using standard tools and integrate it with other STAC-indexed datasets as needed. The combination of **GeoParquet** for storage and **STAC** for metadata yields a powerful, standards-based framework for handling our geospatial data deluge.

##### Example STAC Catalog Files for HF-Radar

Below we present a simplified example of the STAC catalog structure as implemented for the HF-Radar radial metrics data. For clarity, some identifiers and values have been generalized (e.g., using `STATIONX` instead of a real station code, and omitting some repetitive content), and long sections are truncated with `...` for brevity. The example illustrates a scenario with two stations and, for one station, two collections (e.g., two time ranges or configurations), each containing radial metrics data.

**1. Root Catalog (**`catalog.json` **at the dataset root):** This is the top-level catalog listing all stations as its children.

    {
      "type": "Catalog",
      "id": "HF-EOLUS",
      "stac_version": "1.1.0",
      "description": "Root catalog for HF-EOLUS HF-Radar datasets",
      "links": [
        {
          "rel": "self",
          "href": "./catalog.json",
          "type": "application/json",
          "title": "HF-EOLUS HF-Radar Root Catalog"
        },
        {
          "rel": "child",
          "href": "STATIONX/catalog.json",
          "type": "application/json",
          "title": "STATIONX – HF-Radar Station"
        },
        {
          "rel": "child",
          "href": "STATIONY/catalog.json",
          "type": "application/json",
          "title": "STATIONY – HF-Radar Station"
        }
      ]
    }

The root catalog points to each station's own catalog. In this example, we show two stations, **STATIONX** and **STATIONY**.

**2. Station Catalog (**`STATIONX/catalog.json`**):** Inside each station's directory, a catalog file lists that station's collections. Here, station **STATIONX** has two collections (perhaps split by time range or configuration).

    {
      "type": "Catalog",
      "id": "STATIONX",
      "stac_version": "1.1.0",
      "description": "Catalog for HF-Radar station STATIONX",
      "links": [
        {
          "rel": "root",
          "href": "../catalog.json",
          "type": "application/json",
          "title": "HF-EOLUS HF-Radar Root Catalog"
        },
        {
          "rel": "child",
          "href": "STATIONX_2015-01-01_to_2015-12-31/collection.json",
          "type": "application/json",
          "title": "STATIONX 2015-01-01 to 2015-12-31 – Radial Metrics"
        },
        {
          "rel": "child",
          "href": "STATIONX_2016-01-01_to_2020-12-31/collection.json",
          "type": "application/json",
          "title": "STATIONX 2016-01-01 to 2020-12-31 – Radial Metrics"
        }
      ]
    }

Here, **STATIONX** has two collections (covering 2015 and 2016-2020). Each `href` is a subdirectory named after the collection (often using the date range). The `rel: root` link points back up to the root catalog.

**3. Collection (**`STATIONX_2016-01-01_to_2020-12-31/collection.json`**):** Each collection contains metadata describing the dataset and (optionally) links to the item files. We illustrate one collection JSON below. For brevity, we show only a few item links and use `...` to indicate that many similar links are present but omitted.

    {
      "type": "Collection",
      "id": "STATIONX_2016-01-01_to_2020-12-31",
      "stac_version": "1.1.0",
      "description": "HF-Radar radial metrics for station STATIONX (2016–2020)",
      "keywords": ["HF-Radar", "Ocean Currents", "Radial Metrics"],
      "license": "CC-BY-4.0",
      "providers": [
        {
          "name": "Example Institute of Oceanography",
          "roles": ["producer", "host"],
          "url": "https://example.edu"
        },
        {
          "name": "University of Example - Ocean Physics Group",
          "roles": ["processor"],
          "url": "https://example.org"
        }
      ],
      "extent": {
        "spatial": {
          "bbox": [[-10.5, 42.0, -8.5, 44.0]]
        },
        "temporal": {
          "interval": [["2016-01-01T00:00:00Z", "2020-12-31T23:59:00Z"]]
        }
      },
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "summaries": {
        "station": ["STATIONX"],
        "radial_metrics:antenna_pattern": ["APM_v2"],
        "table:columns": [
          {
            "name": "timestamp",
            "type": "datetime",
            "description": "Observation time (UTC)"
          },
          {
            "name": "bearing",
            "type": "number",
            "description": "Azimuth angle of measurement (degrees)"
          },
          {
            "name": "range",
            "type": "number",
            "description": "Range distance of measurement (km)"
          },
          {
            "name": "doppler_velocity",
            "type": "number",
            "description": "Radial velocity (cm/s)"
          },
          {
            "name": "snr",
            "type": "number",
            "description": "Signal-to-noise ratio (dB)"
          },
          ...
        ]
      },
      "links": [
        {
          "rel": "self",
          "href": "./collection.json",
          "type": "application/json"
        },
        {
          "rel": "parent",
          "href": "../catalog.json",
          "type": "application/json",
          "title": "STATIONX"
        },
        {
          "rel": "root",
          "href": "../../catalog.json",
          "type": "application/json",
          "title": "HF-EOLUS HF-Radar Root Catalog"
        },
        {
          "rel": "item",
          "href": "items/radial_metrics_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Radial Metrics STATIONX 2018-06-24T23:30:00Z"
        },
        {
          "rel": "item",
          "href": "items/header_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Header STATIONX 2018-06-24T23:30:00Z"
        },
        {
          "rel": "item",
          "href": "items/rng_info_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Range Info STATIONX 2018-06-24T23:30:00Z"
        },
        ... (additional item links for other timestamps) ...
      ]
    }

In this collection JSON, we include fields like `extent` (with spatial bounding box and temporal range for all data in the collection), `providers` (who created and processed the data), and `summaries` (here we show an example where the Table extension is used to list some columns in the dataset schema). The `links` section has `self`, `parent`, `root` links and a few examples of `item` links.

**4. Item Examples (**`items/*.json`**):** Within the collection's `items/` subdirectory, each time step produces three item files in our setup: one for the radial metrics, one for the header, and one for the range info. Below we show a representative example of each type of Item JSON.

-   **Radial Metrics Item (primary data at a given timestamp):**

<!-- -->

    {
      "type": "Feature",
      "stac_version": "1.1.0",
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "id": "STATIONX_2018-06-24T23:30:00Z",
      "geometry": {
        "type": "Polygon",
        "coordinates": [[ [ -9.0, 43.1 ], [ -9.0, 43.5 ], [ -9.8, 43.5 ], [ -9.8, 43.1 ], [ -9.0, 43.1 ] ]]
      },
      "bbox": [ -9.8, 43.1, -9.0, 43.5 ],
      "properties": {
        "hf_site:code": "STATIONX",
        "hf_site:location": [ -9.21, 43.16 ],
        "radial_metrics:filters": {
          "max_range": 150.0,
          "bearing_ranges": [ [ 0.0, 40.0 ], [ 220.0, 360.0 ] ]
        },
        "table:primary_geometry": "geometry",
        "table:columns": [
          { "name": "timestamp", "description": "Observation time (UTC)", "type": "datetime" },
          { "name": "bearing",   "description": "Bearing of echo (degrees)", "type": "number" },
          { "name": "range",     "description": "Range of echo (km)", "type": "number" },
          { "name": "velocity",  "description": "Radial velocity (cm/s)", "type": "number" },
          { "name": "snr",       "description": "Signal-to-noise ratio (dB)", "type": "number" },
          { "name": "geometry",  "description": "Echo location geometry (WKB)", "type": "binary" }
        ],
        "datetime": "2018-06-24T23:30:00Z",
        "table:row_count": 317
      },
      "links": [
        {
          "rel": "collection",
          "href": "../collection.json",
          "type": "application/json",
          "title": "STATIONX_2016-01-01_to_2020-12-31"
        },
        {
          "rel": "describedby",
          "href": "header_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Header for STATIONX 2018-06-24T23:30:00Z"
        },
        {
          "rel": "related",
          "href": "rng_info_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Range Info for STATIONX 2018-06-24T23:30:00Z"
        },
        {
          "rel": "prev",
          "href": "radial_metrics_STATIONX_2018-06-24T23:00:00Z.json",
          "type": "application/geo+json",
          "title": "Radial Metrics STATIONX 2018-06-24T23:00:00Z"
        },
        {
          "rel": "next",
          "href": "radial_metrics_STATIONX_2018-06-25T00:00:00Z.json",
          "type": "application/geo+json",
          "title": "Radial Metrics STATIONX 2018-06-25T00:00:00Z"
        }
      ],
      "assets": {
        "neg_bragg_peak": {
          "href": "../../radial_metrics/pos_bragg=0/timestamp=2018-06-24T23:30:00Z/STATIONX_2018-06-24T23:30:00Z_0.parquet",
          "type": "application/x-parquet",
          "title": "Radial Metrics – Negative Bragg Peak",
          "roles": [ "data" ]
        },
        "pos_bragg_peak": {
          "href": "../../radial_metrics/pos_bragg=1/timestamp=2018-06-24T23:30:00Z/STATIONX_2018-06-24T23:30:00Z_1.parquet",
          "type": "application/x-parquet",
          "title": "Radial Metrics – Positive Bragg Peak",
          "roles": [ "data" ]
        }
      }
    }

This **radial metrics item** has two assets (negative and positive Bragg peak Parquet files). The `geometry` is a polygon (in this example, a simple rectangle) approximating the coverage area, and the `bbox` is provided accordingly. The properties include some station info and filter settings used (e.g., max range), and a snippet of the table schema (via `table:columns`) for clarity. We include `table:row_count` to indicate how many rows (echoes) are in this time's data. The `links` section links to the parent collection, and crucially connects this item to its corresponding header and range info items (`describedby` and `related` links). It also has `prev` and `next` links to adjacent radial metrics items in time for easy navigation along the timeline.

-   **Header Item (metadata for the same timestamp):**

<!-- -->

    {
      "type": "Feature",
      "stac_version": "1.1.0",
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "id": "header_STATIONX_2018-06-24T23:30:00Z",
      "geometry": {
        "type": "Point",
        "coordinates": [ -9.21, 43.16 ]
      },
      "bbox": [ -9.21, 43.16, -9.21, 43.16 ],
      "properties": {
        "hf_site:code": "STATIONX",
        "hf_site:location": [ -9.21, 43.16 ],
        "radial_metrics:filters": {
          "max_range": 150.0,
          "bearing_ranges": [ [0.0, 40.0], [220.0, 360.0] ]
        },
        "table:primary_geometry": "geometry",
        "table:columns": [
          { "name": "timestamp", "description": "Record timestamp (UTC)", "type": "datetime" },
          { "name": "metakey",   "description": "Metadata key", "type": "string" },
          { "name": "metavalue", "description": "Metadata value", "type": "string" }
        ],
        "datetime": "2018-06-24T23:30:00Z",
        "table:row_count": 37
      },
      "links": [
        {
          "rel": "collection",
          "href": "../collection.json",
          "type": "application/json",
          "title": "STATIONX_2016-01-01_to_2020-12-31"
        },
        {
          "rel": "describes",
          "href": "radial_metrics_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Radial Metrics STATIONX 2018-06-24T23:30:00Z"
        },
        {
          "rel": "prev",
          "href": "header_STATIONX_2018-06-24T23:00:00Z.json",
          "type": "application/geo+json",
          "title": "Header STATIONX 2018-06-24T23:00:00Z"
        },
        {
          "rel": "next",
          "href": "header_STATIONX_2018-06-25T00:00:00Z.json",
          "type": "application/geo+json",
          "title": "Header STATIONX 2018-06-25T00:00:00Z"
        }
      ],
      "assets": {
        "header": {
          "href": "../../header/timestamp=2018-06-24T23:30:00Z/STATIONX_2018-06-24T23:30:00Z_header.parquet",
          "type": "application/x-parquet",
          "title": "LLUV Header Metadata",
          "description": "Original file header metadata",
          "roles": [ "data" ]
        }
      }
    }

The **header item** contains the metadata records from the original radar file (essentially a key-value list of settings and diagnostics). We represent it as a point at the station's location (since the header isn't spatially extensive). Its properties include the same station identifier and filters (repeated for completeness), and a small table schema (here just a few columns: a timestamp and key/value pairs). It links back to the radial metrics item via a `describes` relationship (the radial metrics item had `describedby` pointing to this). The asset is the Parquet file containing the header information. Note that `id` here is prefixed with `"header_"` to distinguish it.

-   **Range Info Item (summary statistics for the same timestamp):**

<!-- -->

    {
      "type": "Feature",
      "stac_version": "1.1.0",
      "stac_extensions": [
        "https://stac-extensions.github.io/table/v1.2.0/schema.json"
      ],
      "id": "rng_info_STATIONX_2018-06-24T23:30:00Z",
      "geometry": {
        "type": "Polygon",
        "coordinates": [[ [ -9.5, 42.5 ], [ -9.5, 43.8 ], [ -10.5, 43.8 ], [ -10.5, 42.5 ], [ -9.5, 42.5 ] ]]
      },
      "bbox": [ -10.5, 42.5, -9.5, 43.8 ],
      "properties": {
        "hf_site:code": "STATIONX",
        "hf_site:location": [ -9.21, 43.16 ],
        "radial_metrics:filters": {
          "max_range": 150.0,
          "bearing_ranges": [ [0.0, 40.0], [220.0, 360.0] ]
        },
        "table:primary_geometry": "geometry",
        "table:columns": [
          { "name": "timestamp", "description": "Observation time (UTC)", "type": "datetime" },
          { "name": "range_cell", "description": "Range cell index", "type": "integer" },
          { "name": "range_km",   "description": "Range distance (km)", "type": "number" },
          { "name": "echo_count", "description": "Number of echoes in this range", "type": "integer" },
          { "name": "avg_velocity", "description": "Average radial velocity (cm/s)", "type": "number" },
          ... 
        ],
        "datetime": "2018-06-24T23:30:00Z",
        "table:row_count": 21
      },
      "links": [
        {
          "rel": "collection",
          "href": "../collection.json",
          "type": "application/json",
          "title": "STATIONX_2016-01-01_to_2020-12-31"
        },
        {
          "rel": "related",
          "href": "radial_metrics_STATIONX_2018-06-24T23:30:00Z.json",
          "type": "application/geo+json",
          "title": "Radial Metrics STATIONX 2018-06-24T23:30:00Z"
        },
        {
          "rel": "prev",
          "href": "rng_info_STATIONX_2018-06-24T23:00:00Z.json",
          "type": "application/geo+json",
          "title": "Range Info STATIONX 2018-06-24T23:00:00Z"
        },
        {
          "rel": "next",
          "href": "rng_info_STATIONX_2018-06-25T00:00:00Z.json",
          "type": "application/geo+json",
          "title": "Range Info STATIONX 2018-06-25T00:00:00Z"
        }
      ],
      "assets": {
        "rng_info": {
          "href": "../../rng_info/timestamp=2018-06-24T23:30:00Z/STATIONX_2018-06-24T23:30:00Z_rng.parquet",
          "type": "application/x-parquet",
          "title": "Range-wise Summary",
          "description": "Echo count and stats per range cell",
          "roles": [ "data" ]
        }
      }
    }

The **range info item** provides summary metrics for each range cell (distance bin) at the given time. We use a polygon approximating the full coverage area of the radar as the geometry (since these summaries pertain to the whole area observed at that time). The properties include the station info and filters, and a subset of the columns (range cell index, range in km, counts of echoes, etc. -- actual implementations might list all relevant fields from the summary). The link with `rel: related` points to the main radial metrics item. The asset is the Parquet file containing the range summary table. The `id` is prefixed with `"rng_info_"` to distinguish it.

All three items share the same `datetime` timestamp and belong to the same collection (`STATIONX_2016-01-01_to_2020-12-31`). Through the links, one can move between the main data and its ancillary info, or iterate over time using the `prev`/`next` links on each series of items (radial metrics series, header series, range info series).

This example demonstrates how the STAC files are structured and interlinked for our HF-Radar radial metrics. In practice, a user starting at the root catalog can drill down to a station, pick a collection, then find items of interest by time. Once an item (radial metrics) is found, its links lead to the associated header and range info if needed for complete context. Because we adhere to STAC and its extensions, this whole catalog can be loaded with standard libraries, enabling programmatic search and retrieval of HF-Radar data in the HF-EOLUS project.

## References

\<ol\> \<li id=\"ref1\"\>STAC Contributors. (2024). SpatioTemporal Asset Catalog (STAC) specification (Version 1.1.0). https://stacspec.org\</li\> \</ol\>

[\[1\]][] [\[2\]][] [\[3\]] radial\_metrics\_VILA\_2018-06-24T23\_30\_00.json

<https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/radial_metrics_VILA_2018-06-24T23_30_00.json>

[\[4\]] header\_VILA\_2018-06-21T19\_00\_00.json

<https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/header_VILA_2018-06-21T19_00_00.json>

[\[5\]] rng\_info\_VILA\_2018-06-28T05\_00\_00.json

<https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/rng_info_VILA_2018-06-28T05_00_00.json>

  [\[1\]]: https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/radial_metrics_VILA_2018-06-24T23_30_00.json#L148-L151
  [\[2\]]: https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/radial_metrics_VILA_2018-06-24T23_30_00.json#L8-L16
  [\[3\]]: https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/radial_metrics_VILA_2018-06-24T23_30_00.json#L124-L132
  [\[4\]]: https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/header_VILA_2018-06-21T19_00_00.json#L98-L106
  [\[5\]]: https://github.com/GOFUVI/hf_radar_radial_metrics_geoparquet_stac_specs/blob/6153ea8d0caf023a0bdee9577ebf1be557077abb/examples/rng_info_VILA_2018-06-28T05_00_00.json#L169-L177
