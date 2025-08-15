# STAC Specification

## STAC Overview

The **SpatioTemporal Asset Catalog (STAC)** is a standard for describing geospatial datasets with metadata that is both **machine-readable** and **web-friendly**. STAC defines a common structure for cataloging data like satellite imagery, maps, point clouds, etc., so that these assets can be easily indexed and discovered by location, time, and other properties. The core STAC specification is intentionally minimal -- it focuses on the essentials needed to locate data in space and time -- and it relies on a flexible extension mechanism to accommodate domain-specific
needs. This design allows STAC to be widely applicable: different projects can extend it (adding additional fields or semantics) without breaking the fundamental compatibility. For a full description of the STAC specifications, please see [\[1\]](#ref1).

Concretely, STAC defines a set of **JSON** document types:

- **Item**: Represents an individual spatiotemporal asset or data capture (analogous to a single scene or observation). A STAC Item is structured as a GeoJSON Feature, meaning it has a geometry (e.g., the footprint of an image or the location of a point measurement) and standard GeoJSON properties like coordinates. Additionally, it includes time information and links to the actual data files (assets).

- **Catalog**: Provides a way to hierarchically organize Items (and Collections) using links, similar to folders in a file system. A Catalog is essentially a container of links and metadata that can point to either Items or other Catalogs, enabling nesting.

- **Collection**: A special kind of Catalog that includes extra metadata describing a group of Items with shared characteristics. For example, a Collection might represent a dataset (like "Landsat 8 Surface Reflectance") and contain fields such as the license, the spatial and temporal extent of all items, keywords, providers, etc., that apply to the whole set of Items. Each Item can reference which Collection it belongs to.

Using these building blocks, one can create a *static catalog* -- a set of JSON files stored, say, on a cloud bucket or a website, where each file links to others to form a navigable tree. Because the links can be relative, it's possible to host a STAC catalog simply as static files on disk or object storage, making it very portable. This is in contrast to older catalog systems that might require a database or server backend. With STAC, even a large data repository can be exposed through a bunch of JSONs that any client or web browser plugin can traverse.

To illustrate STAC's minimalism, the **core fields** in an Item include things like an `id`, a `bbox` (bounding box), a `geometry` (the GeoJSON geometry), a `datetime` (timestamp of the asset), and an `assets` dictionary (with keys for each asset file and values containing links and asset metadata). Additional fields can be added via *extensions* -- JSON Schema definitions that specify extra properties. For example, there are STAC extensions for electro-optical imagery (adding things like cloud cover percentage), for scientific citation (DOIs, etc.), for raster and vector data specifics, and many more. HF-EOLUS takes advantage of this extensibility as needed (as discussed later, we use the Table extension to describe columns of tabular data).

By adopting STAC, **data providers** (like our project) ensure that we're not inventing yet another metadata format for our assets. Instead, we use a common language, which means anyone familiar with STAC can understand our catalog. Likewise, **data consumers** benefit because they can use existing tools -- e.g., STAC browser apps, libraries like PySTAC in Python, or STAC-index -- to search and retrieve data from our catalog without custom code. This significantly lowers the barrier to entry: a researcher can, for instance, load our STAC catalog in a Jupyter notebook using PySTAC and programmatically find all HF-Radar files in a given date range and geographic area with just a few lines of code.

In summary, STAC provides a **lightweight, web-ready index** of our geospatial data, ensuring interoperability and ease of use. The following subsections outline the main STAC components and then detail how HF-EOLUS applies them for organizing the radar metrics data.

## Item Overview

A **STAC Item** is the atomic unit of data in a STAC Catalog -- it corresponds to a single spatiotemporal observation or logical grouping of data that one would consider a single record. In STAC (core 1.0.0), an Item is defined as a JSON that also qualifies as a **GeoJSON Feature**. This means an Item has:

- A `geometry`: usually the footprint of the data. For imagery, this would be the polygon of the scene's coverage. For HF-Radar, this can be a point or small area representing the station location or coverage of that item's data.

- A `bbox`: the bounding box of that geometry (for quick indexing).

- `properties`: a dictionary of metadata key-values (for example, the timestamp of acquisition is given by `datetime` in properties, plus any other relevant metadata like instrument type or processing info).

- An `id`: a unique identifier for the item.

- An `assets` list (or dictionary): each asset entry provides a link (URL or path) to an actual data file plus some metadata like file type, size, or descriptive title.

- A `links` list: links to related entities (e.g., a link to its parent Collection, links to previous/next in a time series, or a link to license info). These links allow the catalog to be navigable.

Because a STAC Item is a *GeoJSON Feature*, it ensures immediate compatibility with GIS tools. One can take the geometry and properties and plot them on a map, or index them in a spatial database, etc. The use of GeoJSON also means the spatial reference is by default WGS84 geographic (lat/lon), which is typically expected for catalogs (the geometry field in the Item is assumed to be in WGS84 coordinates unless otherwise specified). This is convenient because our HF-Radar data and model data are indeed referenced to WGS84 lat/lon (as indicated by the GeoParquet CRS as well).

An Item usually represents **one snapshot in time** of something. For example:

- In a satellite context, one Item = one image scene at a certain date/time.

- In our HF-Radar context, as we will see, one Item = the set of radar measurements (radial metrics) derived from one radar's half-hourly spectral data at a given timestamp.

- If we were cataloging model outputs, one Item could represent one model run output at a certain time (e.g., a forecast for 12:00 UTC on Jan 1, with assets being the output files).

The STAC Item's simplicity is by design. Additional domain-specific details (like the fact that an HF-Radar item contains two types of radial data, or that a model output item might have multiple variables) can be captured either in the `assets` (by having multiple asset entries with different keys) or in extension properties. For instance, the **Table Extension** we use will allow us to include the schema of a tabular asset as part of the Item or Collection metadata, so that a user knows what columns exist in the Parquet file without opening it.

## Catalogs vs Collections

Both **Catalogs** and **Collections** are mechanisms to group Items, but they have distinct roles:

- A **STAC Catalog** is a basic listing mechanism. It has an `id`, a description, and a set of links (some of which are labeled as `child` or `item` or `parent`). Think of a Catalog as a folder that can contain references to Items or to other folders. A Catalog does not enforce any specific metadata beyond identification and linking. It's purely structural. You might use nested catalogs to reflect a logical grouping (e.g., group by region, by year, by data type).

- A **STAC Collection** extends a Catalog by adding dataset-level metadata. In STAC, a Collection is effectively a Catalog with extra required fields like:

- `license`: the usage license of the data (e.g., CC-BY-4.0, or proprietary, etc.).

- `extent`: both spatial extent (bounding box covering all items) and temporal extent (time range covered by all items).

- `providers`: who produced or hosts the data.

- `summaries`: optional statistical summaries for certain properties (e.g., a list of all unique station IDs in the collection, or min/max values of a parameter).

- And it can have keywords, a longer description, etc.

Every Item can optionally link to a Collection it belongs to (via a `collection` field or a link of type `collection`). This is recommended because it lets a user or client quickly find the overarching context for that item. For example, a Collection might state "these are HF-Radar radial metrics from Station X, covering 2023, in CSV format, with these variables" in its description and have the official citation and DOI. Each Item then doesn't need to repeat those details; it just points to the Collection.

In our project, we leverage **Collections** to represent groupings like "all data from a specific radar station and antenna pattern configuration" (more on that below in HF-EOLUS specifics). By doing so, we capture at the Collection level the things common to that group (e.g., the station location, the period it operated, the coordinate system, etc.), which clients can read once. Items then handle the per-timestep specifics.

In contrast, pure **Catalogs** (non-collection catalogs) we use mainly to build the hierarchy above the collections. For example, we might have a top-level Catalog called "HF-EOLUS Data Catalog" with children that are Collections per station. Or we could have an intermediate Catalog grouping stations by region if needed. Catalogs are flexible and impose no rules, which is useful for purely organizing navigation.

## Collection Overview

A STAC **Collection** in HF-EOLUS is used whenever we have a set of Items sharing the same data schema and context. By using Collections, we provide rich metadata once for the whole set:

- We define the **spatial extent** (the geographic coverage of the collection). For a radar station, this might be the radius or area the station can observe (or simply the station location if we consider each echo's location variable).

- We define the **temporal extent** (earliest and latest timestamps of data in that collection).

- The **license** for the data is stated at the collection level (so every item implicitly carries that license by reference).

- We list the **providers** (e.g., the organizations or agencies involved in producing the data).

- We can include a list of **keywords** (e.g., "HF-Radar", "ocean currents",
"radial velocity").

- If using extensions, the collection can specify which extensions apply to its items (for instance, if all items use the `table` extension and perhaps a custom extension for HF radar, the collection will list those).

The collection document also has a `summaries` section where we can provide summary statistics or enumerations of fields. For example, a collection could list all station IDs it contains (though in our case one station per collection, so not needed), or range of frequencies or antenna patterns.

One key reason to use collections is to facilitate searches. If one is using a STAC API or a static catalog search tool, collections allow filtering at a high level (e.g., getCollections by instrument type). In our static context, it's more about organization and metadata clarity.

Collections can themselves be nested under catalogs or even under other collections (since a Collection is a kind of Catalog, STAC allows a hierarchy where collections link to children). However, typically, you have either a Catalog containing Collections or Collections at the top. In HF-EOLUS, we implement a hierarchy where:

- A root Catalog contains all stations (likely as child links).

- Each station is a **Catalog** that groups multiple Collections. Why multiple collections per station? Because if the station had a significant change (like a different antenna pattern calibration), we treat data before vs after that change as separate collections (since their properties differ, e.g., Antenna Pattern ID).

- Each Collection under a station covers a specific configuration period (with uniform metadata like antenna pattern ID) and contains many Items (the individual time steps).

- Items then link back to their Collection (and by extension, to the station catalog, and to root).

This structure ensures that an **Item always belongs to exactly one Collection** (as per STAC's model), and that Collection carries the common metadata for those Items. Meanwhile, the station Catalog and root Catalog just serve to group and connect everything in a browseable tree.

## Catalog Overview

Our top-level **Catalog** (and intermediate station catalogs) follow the concept of **self-contained catalogs** and **relative links** for portability.

A "self-contained" STAC catalog is one where all links that tie the structure together (the links to parent, child, item, collection within the dataset) are **relative paths**, not absolute URLs. This means we can zip up the whole set of JSON files and move it somewhere else (or host it on a different domain) and all the references still work. This is great for scenarios like providing the catalog on physical media or as a downloadable package.

However, for a published online catalog, it's also useful to have a known entry point URL. We implement a **Relative Published Catalog** pattern, which is essentially a self-contained catalog with one addition: the root Catalog (and/or Collections) includes an absolute `self` **link** pointing to its canonical URL. For example, if the root catalog is intended to live at `https://hfeolus.example.com/catalog.json`, we put a self link with that URL in the root JSON. All other links (children, items, etc.) remain relative. This hybrid approach means:

- If a user is browsing the files locally (or we move the files), the internal navigation works via relative links.

- If a user knows the online endpoint, the self link advertises where the authoritative source is, and tools can use that to identify the catalog.

In practice, we keep our STAC implementation simple: we use JSON files on object storage (like AWS S3 or a web server) and ensure that for any given catalog or collection JSON, the `links` include entries for `child` (pointing to sub-catalogs or collections), `item` (pointing to item files), and `parent`/`root` as appropriate. Each item has a link to its parent (collection) and each collection has a link to root (the top catalog). This forms a complete graph for traversal.

## HF-EOLUS Specifications

Now we describe how the general STAC concepts are applied to the HF-EOLUS data, specifically focusing on the **HF-Radar Radial Metrics** portion of the project. (Meteorological model outputs could be cataloged in a similar fashion; for now, our emphasis is on the radar data as it presents the more complex use case with multiple assets per item and custom metadata.)

#### HF-Radar

For HF-Radar Radial Metrics, we define the following structure in STAC:

-   **Assets**: Each radar measurement time interval produces **two GeoParquet files** as data assets. One corresponds to the **positive Bragg peak** echoes and the other to the **negative Bragg peak** echoes. (In HF radar processing, sea echo Doppler spectra have two main peaks -- one from waves approaching and one from waves receding relative to the radar, called Bragg peaks. They are processed separately.) Both files share the same schema (columns like bearing, range, velocity, etc.), but split the data by signal type. We store these Parquet files in an `assets/` subdirectory for each station for organizational and performance reasons. By keeping all Parquet files for a station in one folder, it's easy to run time-partitioned queries (for example, a SQL query scanning all files in that folder by date range).

-   **Item**: Each STAC Item represents the data from **one time step** (e.g., one 30-minute period) for one station. The Item's `datetime` property is set to the nominal time of the observation (for example, the start or center time of that 60-min interval). The Item has two asset entries: perhaps named `"pos"` and `"neg"` (or similar keys) pointing to the Parquet files for that interval. Each asset entry includes metadata such as the file type (`"application/x-parquet"` as media type, if using the common STAC media type for Parquet), and perhaps a brief title (\"Radial metrics (positive Bragg peak)\" etc.). The Item's geometry could be a point (the location of the radar site) or a polygon approximating the coverage area. We have flexibility here, but a point for the station is a simple representative geometry. The properties of the Item include things like the station ID, the antenna pattern ID in use, and any processing parameters that vary over time (though typically, those are consistent and might be recorded at the collection level if constant).

-   **Collection**: As mentioned, we create one Collection per station **per antenna pattern (APM) configuration**. HF radar stations periodically undergo recalibration of their antenna pattern (which affects how radial components are interpreted). When that happens, data before and after are not strictly identical in terms of how they should be processed, so we separate them. Thus, for station **XYZ**, if it had two antenna patterns over our data period, we have two collections: e.g., \"Station XYZ (APM v1, 2019-2021)\" and \"Station XYZ (APM v2, 2021-2023)\". Each Collection will have:

-   The station's name or ID in its title and a description of that antenna pattern usage.

-   The spatial extent likely as a circle or radius around the station (since the radar has a max range).

-   The temporal extent covering the first to last item in that collection.

-   The antenna pattern ID stored as a property in the collection (and each item references which one by belonging to this collection).

-   We also utilize the **STAC Table Extension** for each Collection. This extension allows us to include a snippet in the Collection metadata describing the columns of the data tables. For example, we can list that each Parquet asset has columns: `time` (ISO timestamp), `bearing` (degrees), `range` (km), `doppler_velocity` (m/s), `snr` (dB), etc., along with their data types and units. By including this in the Collection, a user or client can know the schema of the data **without opening every Parquet file** -- especially useful when integrating with tools or for quick understanding of what each asset contains.

-   **Station Catalog**: We introduce an intermediate Catalog for each station (if a station has multiple collections). The station Catalog is basically a folder that groups all Collections of that station. For instance, a Catalog `station_XYZ/catalog.json` would have links to the two collections (APM v1 and APM v2). This Station Catalog has an `id` equal to the station id perhaps, and can include station-level info like location or operator in its description. But since Collections carry most info, the station catalog is mostly structural.

-   **Root Catalog**: At the top, we have a root `catalog.json` (or a directory with a catalog) that links to each station's catalog (or collection if we decided one collection per station -- but we have multiple, so a catalog per station is cleaner). The root provides a general description of the HF-EOLUS dataset as a whole and maybe a link to project documentation or DOI if available.

Concerning the **naming conventions** for files and IDs:

- We name each Parquet file and corresponding item file in a consistent way: `{station_id}_{timestamp}_{bragg}.parquet` for data files and `{station_id}_{timestamp}.stac.json` for the item. Here, `{bragg}` is either \"pos\" or \"neg\". The timestamp is formatted in UTC **ISO 8601** (compact form, e.g., `20230101T120000Z` or similar) -- using a standard format means that lexicographically sorting the filenames also sorts by time, which is convenient. For example: `ABCD_20230715T120000Z_pos.parquet`, `ABCD_20230715T120000Z_neg.parquet`, and `ABCD_20230715T120000Z.stac.json` would be a set for station ABCD at noon on 2023-07-15.

- Item `id` could be set to the same base `{station_id}_{timestamp}` so that it's unique across the catalog.

- Collection `id` might be the station id plus a suffix for the antenna pattern (e.g., `ABCD_apm_1234` where 1234 is a short UID of the pattern file). - Station catalog id just `ABCD` for example.

With this organization, **discovery and use** of the data becomes straightforward:

- A user can open the root catalog and see links to each station.
- They navigate into a station and see possibly multiple collections (if multiple data subsets exist).
- Each collection describes itself and provides a link to all items (often via a `child` link or an items list; in a static catalog we often provide a list of
item links or use the convention of an `item` field in the catalog JSON
linking to each item -- but listing thousands of items in one file is
impractical, so we often do pagination or simply rely on the fact that
item files can be enumerated in the storage). 
- Because our files are named by timestamp, a user can easily find a specific time or do a time range query by scanning filenames. Additionally, using STAC tools, one could load all items and filter by the `datetime` field.

Finally, we ensure that each Item's `properties` include any needed domain metadata. For HF-Radar, this might include: 

- The frequency of the radar (MHz), 
- The transmitting antenna type, - Radial resolution, etc., 
- Quality indicators or processing flags if any. These could also be added via a custom extension or just included as additional fields under properties (STAC is okay with extra fields not in core). By capturing such details, we make the Item self-explanatory to someone who finds it through a search interface.

In summary, the HF-EOLUS STAC implementation organizes the radial metrics into a **navigable catalog** where each measurement set is an Item with two data assets. Collections group these by station and configuration for clarity, and all metadata needed to interpret and use the data (from file schema to spatial/temporal context) is provided in the STAC JSON. This approach ensures that researchers can discover and access the HF-Radar data (and similarly the model data, if cataloged) using standard tools, and integrate it with other STAC-indexed datasets as needed. The combination of **GeoParquet** for storage and **STAC** for metadata yields a powerful, standards-based framework for handling our geospatial data deluge.

## References

<ol>
<li id="ref1">STAC Contributors. (2024). SpatioTemporal Asset Catalog (STAC) specification (Version 1.1.0). https://stacspec.org</li>
</ol>