## Why GeoParquet?

GeoParquet is an open format based on Apache Parquet that enables efficient and scalable storage of geospatial data. It leverages Parquet’s columnar architecture for compression and fast access to large volumes of information, while remaining fully compatible with the Apache Arrow and Parquet ecosystems. By embedding geometry-specific metadata (points, lines, polygons) directly within the Parquet schema, GeoParquet makes spatial data immediately interpretable by libraries such as GDAL, GeoPandas and others across multiple languages (Python, Java, C++), without sacrificing performance or advanced spatial operations.

**Key advantages of GeoParquet include:**

* **Efficiency and compression**: Columnar storage dramatically reduces file sizes and boosts read/write speeds.
* **Interoperability**: As an open, widely supported standard, GeoParquet integrates seamlessly into diverse platforms and tools.
* **Scalability**: Designed for very large datasets, GeoParquet is ideal for data pipelines, geospatial analysis, and Big Data applications.

Although other HF-Radar products—such as gridded surface-current fields—are typically stored in NetCDF (the de facto standard for multi-dimensional scientific data, using CF metadata conventions to describe time, latitude, longitude, depth and variable attributes), there is, to our knowledge, no established standard for Radial Metrics. Because NetCDF is less efficient than GeoParquet when processing massive volumes of individual echoes, we have chosen GeoParquet as the ingestion format for LLUV Radial Metrics.