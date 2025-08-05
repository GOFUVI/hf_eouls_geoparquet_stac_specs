# GeoParquet v1.1.0 Specification

This section describes the GeoParquet v1.1.0 standard and provides an example of the `geo` metadata property as embedded in our GeoParquet files, together with a discussion of how it fulfills required fields and which optional extensions are included.

### Introduction to GeoParquet v1.1.0

GeoParquet v1.1.0 is an extension of Apache Parquet for geospatial data. It defines the conventions for encoding geometry columns, coordinate reference systems (CRS), and file-level and column-level metadata to ensure interoperability across platforms and tools. The main requirements of the standard include:

- Geometry columns MUST be stored at the root schema and encoded as Well-Known Binary (WKB) or native GeoArrow types.
- CRS information MUST be provided in PROJJSON format or implied as OGC:CRS84 when absent.
- File-level metadata MUST include the GeoParquet version, the name of the primary geometry column, and metadata for each geometry column (encoding, geometry types, CRS, bounding box, etc.).
- Column-level metadata MUST describe each geometry column’s encoding, supported geometry types, CRS, orientation, edges, and may include an optional epoch for dynamic CRS.

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
- **Dataset-specific extension**: clients may include an `extra_radial_metadata` object to embed HF-Radar radial metrics per feature.
- **Additional optional spec elements**: a coordinate `epoch` could be supplied to support dynamic CRS scenarios.
