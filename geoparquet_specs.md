# GeoParquet v1.1.0 Specification

This section summarizes the GeoParquet v1.1.0 standard and provides an example of the `geo` metadata property as embedded in our GeoParquet files, together with a discussion of how it fulfills required fields and which optional extensions are included. Please refer to  [\[1\]](#ref1) for the full set of specifications.

## Introduction to GeoParquet v1.1.0

**GeoParquet v1.1.0** [\[1\]](#ref1) is an Open Geospatial Consortium (OGC) specification that extends **Apache Parquet** (a columnar storage format) to handle geospatial vector data. It defines how to embed geometric data and coordinate reference information within a Parquet file's schema and metadata so that any compliant software can interpret the spatial content. Key aspects of the GeoParquet standard include:

-   **Geometry Column Encoding**: Geometry data (points, lines polygons, etc.) are stored in a binary form (often Well-Known Binary - WKB) or in optimized encodings defined by **GeoArrow**. All geometry columns must reside at the top level of the table schema (not nested in structs/arrays), and each geometry column is either required (one geometry per row) or optional (zero or one geometry per row) but never repeated.

-   **CRS Specification**: A GeoParquet file records the Coordinate Reference System for geometries. The CRS is stored in the metadata as a **PROJJSON** object (the JSON representation of coordinate reference definitions) If no CRS is provided, it defaults to **OGC:CRS84** (which is effectively WGS84 longitude-latitude). This ensures that consumers of the data know how to interpret the coordinates (e.g., latitude/longitude vs. projected coordinates).

-   **File-Level Metadata**: Each file's Parquet metadata includes a top-level `geo` key containing a JSON structure that describes the GeoParquet version and a *columns* object detailing each geometry column. Required fields at this level include `version` (GeoParquet spec version, e.g., \"1.1.0\"), `primary_column` (the name of the primary geometry field), and `columns` (the dictionary of geometry column metadata).

-   **Geometry Column Metadata**: Under the `columns` entry, each geometry column is described by required fields and optional fields. **Required** per-column metadata includes the geometry `encoding` (e.g., \"WKB\"), the list of geometry types present (`geometry_types` such as Point, Polygon, etc.), the `crs` (coordinate reference system, or null if undefined), and a `bbox` (bounding box for all geometries). **Optional** fields can include `orientation` (to specify polygon winding order, if all polygons follow a given convention) and `edges` (to indicate if geometry edges are straight in a planar projection or follow great-circles on a sphere), as well as an `epoch` for dynamic CRS reference times, among others.

By adhering to GeoParquet's schema, the HF-EOLUS data files become self-describing for spatial content. Any library that understands GeoParquet (for example, GDAL, GeoPandas, Apache Sedona, etc.) can read these Parquet files and immediately know how to reconstruct geometries in the correct coordinate
system. This interoperability is critical: it means the data can be used across different languages and platforms (Python, R, Java, SQL engines) without custom import code. The **primary geometry column** in our case is typically a point geometry representing either the location of a radar echo (latitude/longitude of the echo's coordinates) or a point on a model grid.

## Why GeoParquet?

GeoParquet was chosen to enable **efficient, scalable storage and analysis** of the project's geospatial data. It leverages Parquet's columnar layout, which offers several key advantages for large datasets:

-   **High Compression & Speed**: Parquet's columnar storage compresses data very effectively and allows reading only the needed columns. For example, if an analysis only needs timestamp and wind speed, Parquet can skip over other columns like geometry or bearing, leading to faster scan times. Columnar layout also aligns well with CPU caching and vectorized processing, yielding faster I/O for analytical queries.

-   **Interoperability**: As an open standard, Parquet (and by extension GeoParquet) is supported by many tools in the big-data ecosystem (Spark, Hive, Dask, DuckDB, etc.). This means our radar metrics can be queried with SQL engines (e.g., AWS Athena or DuckDB) directly from the Parquet files, or loaded into dataframes for processing in Python/R without conversion. The data format is self-contained; all necessary schema and CRS info travels with the file.

-   **Scalability**: Parquet is designed for *big data*. Splitting data into many Parquet files (for example, one file per time interval, per station) allows parallel processing across a cluster. The combination of partitioned files and Parquet's internal indexing (row groups, statistics) means queries can be executed on subsets of data efficiently. This is ideal for HF-EOLUS, where one might run queries over *years* of measurements or join radar data with other large datasets.

Beyond these advantages, using GeoParquet aligns with emerging best practices for **cloud-native geospatial data**. Traditional formats like CSV or shapefiles are inefficient for large-scale use. By contrast, columnar formats like Parquet are *specifically designed for the cloud paradigm*, often yielding faster reads and smaller storage footprints than row-oriented formats.

### GeoParquet vs. NetCDF

It is instructive to compare GeoParquet with **NetCDF**, as NetCDF has long been a cornerstone for scientific data storage (especially for meteorological and oceanographic grids). NetCDF (which typically uses an HDF5 under the hood) excels at packaging **multi-dimensional arrays** (e.g., variables on a 2D grid over time) along with rich metadata, following conventions like CF (Climate and Forecast) metadata for geospatial referencing. It has been the *de facto* standard for many environmental datasets and works well in HPC environments for numeric
simulations.

However, **NetCDF is not optimized for query-oriented analytics on massive collections of files**, especially in cloud or distributed environments. Key differences and trade-offs include:

-   **Data Model**: NetCDF's strength is in representing n-dimensional arrays (e.g., a variable with dimensions \[time, depth, lat, lon\]). This makes it very natural for gridded model output. Parquet, on the other hand, is inherently tabular (two-dimensional tables of rows and columns). This means that to store the same gridded data in Parquet, one might flatten it (each row = one grid cell with coordinates and value) or use nested structures. While Parquet *can* store these, it doesn't natively understand multi-dimensional chunking the way NetCDF does. For *point* data like HF radar echoes, the tabular model is perfectly suited -- each echo is a row, with columns for its attributes.

-   **Compression and Size**: NetCDF data can be compressed (NetCDF4 allows DEFLATE compression on variables), but Parquet's columnar compression often yields equal or better reduction for many data types, and it can compress across many rows of a single column very efficiently. In one example comparison, a time-series dataset stored in NetCDF was over 140 MB, while the same data in Parquet was around 8 MB -- an order of magnitude smaller [\[1\]](#ref2). Write and read speeds were also favorable for Parquet in that test. This is not universally true for all datasets, but it underscores that Parquet's binary encoding plus compression is very effective for large tables.

-   **Random Access and Cloud Access**: NetCDF is designed for partial access *in theory* -- one can read a subset of a NetCDF file (a range of times or a region of the grid) if the software supports it. But in practice, especially on cloud storage, accessing a small portion of a NetCDF often requires downloading the entire file or large chunks of it. The format was created decades ago, before cloud object storage; it doesn't inherently support the HTTP range queries or the "windowed reads" needed for efficient cloud use. GeoParquet/Parquet, conversely, was built with modern data lake usage in mind -- it splits data into row groups and uses column indexes, so a query engine can skip through a Parquet file via byte offsets to retrieve only relevant pieces. This trait, along with the ability to store many small Parquet files in a directory, makes Parquet very **cloud-friendly**. Users can query thousands of Parquet files on S3 with tools like Spark or DuckDB without downloading them entirely, whereas NetCDF would force downloading each file for local access in many cases.

-   **Metadata and Standards**: NetCDF relies on community conventions (like CF) to standardize meaning of metadata (e.g., units, coordinate variables), and it is highly mature in the climate/oceanography community. GeoParquet is newer and focuses on geospatial vector metadata (CRS, geometry). For HF-EOLUS, the *spatial* aspect (locations of observations) is critical, and GeoParquet explicitly handles that with CRS and geometry columns. NetCDF lacks a standardized way to embed a geometry with a coordinate reference system beyond listing coordinate axes; it treats latitude/longitude as data variables rather than as an intrinsic geometry object. By using GeoParquet, we directly attach the geometry and CRS in a standard way, which is more **interoperable** with GIS tools out-of-the-box.

-   **Ecosystem and Usability**: Many modern data science tools (like Pandas, dask, Arrow, Spark) have first-class support for Parquet. They can scan and filter data in parallel, join tables, etc., treating the data like a database. NetCDF is usually accessed through specialized libraries (netCDF4, xarray) that are excellent for numerical analysis but not as seamless for large-scale query operations or SQL-style analysis. In HF-EOLUS, using Parquet means we can leverage SQL engines (DuckDB or Athena) to query the radar data *directly in place*, e.g., "find all echoes in a given spatial bounding box and time range" or "compute monthly statistics" without converting files or loading everything in memory. This capability was crucial for us to build efficient analysis pipelines.

In summary, **NetCDF remains a powerful format for certain use cases** (especially multi-dimensional gridded data in HPC contexts), but for the *volume and variety of data* in HF-EOLUS, **GeoParquet provides better performance and integration** with modern analytic tools. We found NetCDF increasingly inefficient when dealing with millions of individual point observations and long time series, whereas Parquet allowed us to scale out horizontally and use distributed computing. The adoption of GeoParquet was motivated by these efficiency gains and the desire to maintain a unified approach for both irregular point data and gridded data. As cloud-native formats like Parquet (and also Zarr for arrays) gain traction, they complement or even replace older formats by offering more accessible data **without sacrificing detail**.


## Example Geo Metadata and Compliance

To illustrate how the GeoParquet standard is applied, below is an excerpt of the `geo` metadata JSON from one of our Parquet files (simplified for clarity). This metadata snippet shows all required fields mandated by GeoParquet v1.1.0, as well as optional fields we use for HF-Radar metrics:

```json
{
  "version": "1.1.0",
  "primary_column": "geometry",
  "columns": {
    "geometry": {
      "encoding": "WKB",
      "geometry_types": [
        "Point"
      ],
      "crs": {
        "$schema": "https://proj.org/schemas/v0.5/projjson.schema.json",
        "type": "GeographicCRS",
        "name": "WGS 84 longitude-latitude",
        "datum": {
          "type": "GeodeticReferenceFrame",
          "name": "World Geodetic System 1984",
          "ellipsoid": {
            "name": "WGS 84",
            "semi_major_axis": 6378137,
            "inverse_flattening": 298.257223563
          }
        },
        "coordinate_system": {
          "subtype": "ellipsoidal",
          "axis": [
            {
              "name": "Geodetic longitude",
              "abbreviation": "Lon",
              "direction": "east",
              "unit": "degree"
            },
            {
              "name": "Geodetic latitude",
              "abbreviation": "Lat",
              "direction": "north",
              "unit": "degree"
            }
          ]
        },
        "id": {
          "authority": "OGC",
          "code": "CRS84"
        }
      },
      "bbox": [
       <minX>, <minY>, <maxX>, <maxY>
      ],
      "orientation": "counterclockwise",
      "edges": "spherical"
    }
  }
}
```

In this example:

-   **Required top-level fields**: The `version` is **1.1.0** indicating compliance with that GeoParquet spec version. The `primary_column` is `"geometry"`, meaning this is the main geometry field for spatial operations. The `columns` object then contains an entry for `"geometry"`.

-   **Required geometry column metadata** (for the `"geometry"` column): We specify the `encoding` as **WKB** (well-known binary for geometry storage), the `geometry_types` as `["Point"]` since each record is a point location, and a detailed `crs` object in PROJJSON format describing WGS84 (longitude-latitude axes). The `bbox` provides the file's spatial extent in \[minX, minY, maxX, maxY\] (here, effectively min lon, min lat, max lon, max lat).

-   **Optional GeoParquet fields used**: We include `orientation: "counterclockwise"`, indicating that if there were any polygon geometries, their outer rings follow a counterclockwise vertex order (an OGC convention). For point data this may not apply, but it's included for completeness and future compatibility. We also set `edges: "spherical"`, declaring that edges between coordinates should be interpreted along the surface of the ellipsoid (great-circle arcs) rather than straight lines on a projected plane. These optional fields come from the GeoParquet spec extensions and ensure that if we ever include non-point geometries (e.g., coverage polygons), their orientation and edge interpolation are standardized.

Every Parquet file in HF-EOLUS containing geospatial data has a similar metadata block, ensuring it **fully conforms to GeoParquet v1.1.0**. This means any tool can verify the file by checking that all required fields are present and correctly formatted per the official schema. It also future-proofs the data: as GeoParquet evolves, versioning in the metadata (`"version": "1.1.0"`) lets readers know exactly which ruleset was used, and they can adapt accordingly.

By following the GeoParquet specification, HF-EOLUS achieves a blend of *efficiency* and *self-descriptiveness*: the data is stored compactly and can be queried at scale, and yet each file carries the information needed to interpret its spatial content without external documentation.

## References

<ol>
<li id="ref1">GeoParquet v1.1.0. https://geoparquet.org/releases/v1.1.0</li>
<li id="ref2">Efficient Storage and Querying of Geospatial Data with Parquet and DuckDB https://waterprogramming.wordpress.com/2022/03/07/efficient-storage-and-querying-of-geospatial-data-with-parquet-and-duckdb</li>
</ol>