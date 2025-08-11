# STAC Specification

## STAC Overview

The SpatioTemporal Asset Catalog (STAC) family of specifications aim to standardize the way geospatial asset metadata is structured and queried. A "spatiotemporal asset" is any file that represents information about the Earth at a certain place and time.

STAC is intentionally designed with a minimal core and flexible extension mechanism to support a broad set of use cases. This specification has matured over the past several years, and is used in numerous production deployments.

This is advantageous to providers of geospatial data, as they can simply use a well-designed, standard format and API without needing to design their own proprietary one. This is advantageous to consumers of geospatial data, as they can use existing libraries and tools to access metadata, instead of needing to write new code to interact with each data provider's proprietary formats and APIs.

The Item, Catalog, and Collection specifications define a minimal core of the most frequently used JSON object types. Because of the hierarchical structure between these objects, a STAC catalog can be implemented in a completely 'static' manner as a group of hyperlinked Catalog, Collection, and Item URLs, enabling data publishers to expose their data as a browsable set of files.

## Item Overview

Fundamental to any SpatioTemporal Asset Catalog, an Item object represents a unit of data and metadata, typically representing a single scene of data at one place and time. A STAC Item is a GeoJSON Feature and can be easily read by any modern GIS or geospatial library, and it describes a SpatioTemporal Asset. The STAC Item JSON specification uses the GeoJSON geometry to describe the location of the asset, and then includes additional information:

the time the asset represents;
a thumbnail for quick browsing;
asset links, to enable direct download or streaming access of the asset;
relationship links, allowing users to traverse other related resources and STAC Items.
A STAC Item can contain additional fields and JSON structures to communicate more information about the asset, so it can be easily searched.

## Catalogs vs Collections

A Catalog is a very simple construct - it just provides links to Items or to other Catalogs. The closest analog is a folder in a file structure, it is the container for Items, but it can also hold other containers (folders / catalogs).

The Collection entity shares most fields with the Catalog entity but has a number of additional fields: license, extent (spatial and temporal), providers, keywords and summaries. Every Item in a Collection links back to their Collection, so clients can easily find fields like the license. Thus every Item implicitly shares the fields described in their parent Collection. Collection entities can be used just like Catalog entities to provide structure, as they provide all the same options for linking and organizing.



## Collection Overview

A STAC Collection includes the core fields of the Catalog entity and also provides additional metadata to describe the set of Items it contains. The required fields are fairly minimal - it includes the 4 required Catalog fields (id, description, stac_version and links), and adds license and extents. But there are a number of other common fields defined in the spec, and more common fields are also defined in STAC extensions. These serve as basic metadata, and ideally Collections also link to fuller metadata (ISO 19115, etc) when it is available.

As Collections contain all of Catalogs' core fields, they can be used just as flexibly. They can have both parent Catalogs and Collections as well as child Items, Catalogs and Collections. Items are strongly recommended to have a link to the Collection they are a part of. Items can only belong to one Collection, so if an Item is in a Collection that is the child of another Collection, then it must pick which one to refer to. Generally the 'closer' Collection, the more specific one, should be the one linked to.

The Collection specification is used standalone quite easily - it is used to describe an aggregation of data, and doesn't require links down to sub-catalogs and Items. This is most often used when the software does operations at the layer / coverage level, letting users manipulate a whole collection of assets at once.

## Catalog Overview

There are two required element types of a Catalog: Catalog and Item. A STAC Catalog points to STAC Items, or to other STAC catalogs. It provides a simple linking structure that can be used recursively so that many Items can be included in a single Catalog, organized however the implementor desires.

En estas especificaciones vamos a implementar un catalogo autocontenido que pueda ser utilizado offline, compartido online o añadido a un servicio de descarga online.

### Self-contained Catalogs
A 'self-contained catalog' is one that is designed for portability. Users may want to download a catalog from online and be able to use it on their local computer, so all links need to be relative. Or a tool that creates catalogs may need to work without knowing the final location that it will live at online, so it isn't possible to set absolute 'self' URL's. These use cases should utilize a catalog that follows the listed principles:

Only relative href's in structural links: The full catalog structure of links down to sub-catalogs and Items, and their links back to their parents and roots, should be done with relative URL's. The structural rel types include root, parent, child, item, and collection. Other links can be absolute, especially if they describe a resource that makes less sense in the catalog, like sci:doi, derived_from or even license (it can be nice to include the license in the catalog, but some licenses live at a canonical online location which makes more sense to refer to directly). This enables the full catalog to be downloaded or copied to another location and to still be valid. This also implies no self link, as that link must be absolute.

Use Asset href links consistently: The links to the actual assets are allowed to be either relative or absolute. There are two types of 'self-contained catalogs'.

### Self-contained with Assets
These use relative href links for the assets, and includes them in the folder structure. This enables offline use of a catalog, by including all the actual data, referenced locally.

Self-contained catalogs tend to be used more as static catalogs, where they can be easily passed around. But often they will be generated by a more dynamic STAC service, enabling a subset of a catalog or a portion of a search criteria to be downloaded and used in other contexts. That catalog could be used offline, or even published in another location.

Self-contained catalogs are not just for offline use, however - they are designed to be able to be published online and to live on the cloud in object storage. They just aim to ease the burden of publishing, by not requiring lots of updating of links. Adding a single self link at the root is recommended for online catalogs, turning it into a 'relative published catalog', as detailed below. This anchors it in an online location and enables provenance tracking.

### Relative Published Catalog
This is a self-contained catalog as described above, except it includes an absolute self link at the root to identify its online location. This is designed so that a self-contained catalog (of either type, with its assets or just metadata) can be 'published' online by just adding one field (the self link) to its root (Catalog or Collection). All the other links should remain the same. The resulting catalog is no longer compliant with the self-contained catalog recommendations, but instead transforms into a 'relative published catalog'. With this, a client may resolve Item and sub-catalog self links by traversing parent and root links, but requires reading multiple sources to achieve this.

So if you are writing a STAC client it is recommended to start with just supporting these two types of published catalogs. In turn, if your data is published online publicly or for use on an intranet then following these recommendations will ensure that a wider range of clients will work with it.

## Especificaciones para HF-EOLUS

### Radial Metrics

En nuestras especificaciones para Radial Metrics tenemos, para cada timestamp, un fichero geoparquet para los radial metrics provenientes del pico de bragg positivo y otro para el negativo. Estos son nuestros assets. Con el fin de facilitar su ingestion en una bese de datos (por ejemplo Athena en AWS) Todos los assets de una estacion se almacenan en un mismo directorio assets. Un item incluye los dos ficheros geoparquet de cada timestamp (el que contiene los radial metrics del pico positivo y el que contiene los radial metrics del pico negativo). Los items de una estacion se organizan en colecciones. Una coleccion son todos los radial metrics de una estacion procesados un mismo Antenna Pattern. Por tanto todos los items de una coleccion tienen en comun el UUID del pattern empleado para procesarlos. Las colecciones de periodos de antenna pattern se organizan en estaciones mediante un catalogo. Un catalogo de estaciones es la raiz del catalogo STAC.