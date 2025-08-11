# GeoParquet v1.1.0 Specification

This section describes the GeoParquet v1.1.0 standard and provides an example of the `geo` metadata property as embedded in our GeoParquet files, together with a discussion of how it fulfills required fields and which optional extensions are included.

### Introduction to GeoParquet v1.1.0

GeoParquet v1.1.0 is an extension of Apache Parquet for geospatial data. It defines the conventions for encoding geometry columns, coordinate reference systems (CRS), and file-level and column-level metadata to ensure interoperability across platforms and tools. The main requirements of the standard include:

- Geometry columns MUST be stored at the root schema and encoded as Well-Known Binary (WKB) or native GeoArrow types.
- CRS information MUST be provided in PROJJSON format or implied as OGC:CRS84 when absent.
- File-level metadata MUST include the GeoParquet version, the name of the primary geometry column, and metadata for each geometry column (encoding, geometry types, CRS, bounding box, etc.).
- Column-level metadata MUST describe each geometry column’s encoding, supported geometry types, CRS, orientation, edges, and may include an optional epoch for dynamic CRS.

### Why GeoParquet?

GeoParquet enables efficient and scalable storage of geospatial data. It leverages Parquet’s columnar architecture for compression and fast access to large volumes of information, while remaining fully compatible with the Apache Arrow and Parquet ecosystems. By embedding geometry-specific metadata (points, lines, polygons) directly within the Parquet schema, GeoParquet makes spatial data immediately interpretable by libraries such as GDAL, GeoPandas and others across multiple languages (Python, Java, C++), without sacrificing performance or advanced spatial operations.

**Key advantages of GeoParquet include:**

* **Efficiency and compression**: Columnar storage dramatically reduces file sizes and boosts read/write speeds.
* **Interoperability**: As an open, widely supported standard, GeoParquet integrates seamlessly into diverse platforms and tools.
* **Scalability**: Designed for very large datasets, GeoParquet is ideal for data pipelines, geospatial analysis, and Big Data applications.

Although gridded surface-current fields—are typically stored in NetCDF (the de facto standard for multi-dimensional scientific data, using CF metadata conventions to describe time, latitude, longitude, depth and variable attributes), NetCDF is less efficient than GeoParquet when processing massive volumes of data. 

The HF-EOLUS project routinely handles very large datasets—sometimes numbering in the millions of records—and requires highly efficient analysis pipelines, which motivated our adoption of GeoParquet.

### Example Geo Metadata and Compliance

Below is an excerpt of the `geo` metadata property as it appears in our GeoParquet files. It includes all **required** top-level fields from GeoParquet v1.1.0, as well as several **optional** spec extensions that we leverage for radial metrics.

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

- **Required top-level fields**: `version`, `primary_column`, and `columns` satisfy GeoParquet v1.1.0 file metadata requirements.
- **Required geometry column metadata** under `columns.geometry`: `encoding`, `geometry_types`, `crs` (explicit PROJJSON), and `bbox`.
- **Included optional GeoParquet fields**: `orientation` (polygon winding order) and `edges` (set to "spherical" for geodetic edges).
