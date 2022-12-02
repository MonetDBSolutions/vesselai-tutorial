# MonetDB VesselAI Tutorial

- [Plugins and Clients](#plugins-and-clients-or-how-to-communicate-with-monetdb)
- [Inserting data](#inserting-data)
    - [Importing ShapeFiles](#importing-shapefiles)
    - [Importing WKT and WKB geometries](#importing-wkt-and-wkb-geometries)
- [Pre-processing spatio-temporal data](#pre-processing-spatio-temporal-data-)
    - [Geometry vs Geography](#geometry-vs-geography)
- [Reading data](#reading-data)
    - [Temporal filters](#temporal-filters)
    - [Spatial filters](#spatial-filters)
        - [Distance within](#distance-within)
        - [Intersects](#intersects)
        - [Simple latitude/longitude comparisons](#simple-latitudelongitude-comparisons)
- [Processing data](#processing-data)
    - [Geometric functions index](#geometric-functions-index)
    - [Use Case: Aggregating trajectories](#use-case-aggregating-trajectories)
    - [Use Case: Calculate distance of ships to ports](#use-case-calculate-distance-of-ships-to-ports)
- [Exporting data](#exporting-data)
- [Geometric Functions Support](#geometric-functions-support)
    - [OGC SF Types](#ogc-sf-types)
    - [GOC SF Functions](#ogc-sf-functions)
    - [Other supported functions](#other-supported-functions)

This guide should give you an overview on how to use MonetDB and it's
new geometric features, developed in the context of VesselAI. The
documentation for MonetDB can be found
[here](https://www.monetdb.org/documentation/user-guide/). 

The implementation of the geometric database module follows the _OGC [Simple Feature
Access Standard](https://www.ogc.org/standards/sfs)_. For a detailed
list of the features that we support please look at the end of this
document.

If you have any questions or problems, do not hesitate to contact us
at *vesselai@monetdbsolutions.com*.

## Plugins and Clients (or: How to communicate with MonetDB)

To use and interact with MonetDB, you must use a client or plugin. The
environment you are using determines which you should use: if you want
to connect with your Python program, you can use
[pymonetdb](https://pypi.org/project/pymonetdb/); if you
want to inspect the contents of the database through your terminal,
[mclient](https://www.monetdb.org/documentation/user-guide/client-interfaces/mclient/)
is the appropriate tool.

Here are some of the most relevant plugins/clients:
- *Python*: [pymonetdb](https://pypi.org/project/pymonetdb/) (DBAPI
  interface, simple client)
- *Python*: [SQLAlchemy](https://github.com/MonetDB/sqlalchemy-monetdb)
  (Python ORM)
- *Java*: [JDBC driver](https://www.monetdb.org/documentation/user-guide/client-interfaces/libraries-drivers/jdbc-driver/)
- *Terminal CLI*: [mclient](https://www.monetdb.org/documentation/user-guide/client-interfaces/mclient/)

After choosing the plugin, you just have to connect to the server using: 
	- the server address and port
	- the username and password you were assgined
	- the name of the database you want to connect to

```python
conn = pymonetdb.connect(
        username="vessel_user1",
        password="vessel_pass1",
        database="vessel_db",
        hostname="www.server.db"
        port=50000)
```

## Inserting data

**Note**: If you are only going to read data from the production
database, you can skip the next few sections until the ["Reading
Data"](#reading_data) chapter.

To start storing data in MonetDB, you must create a table with the
`CREATE TABLE` command, using a schema that fits your data
([documentation](https://www.monetdb.org/documentation/user-guide/sql-manual/data-definition/table-definition/)).

After defining the schema, you can use the `COPY INTO` command to load
data from a file (e.g. CSV) or from standard input
([documentation](https://www.monetdb.org/documentation/user-guide/sql-manual/data-loading/copy-from/)).

```sql
-- AIS navigation example (does not contain all the message fields)
CREATE TABLE ais_navigation (
    mmsi       INTEGER,
	speed      DOUBLE PRECISION, 
	lon        DOUBLE PRECISION, 
	lat        DOUBLE PRECISION,
	time_epoch BIGINT, 
	time_stamp TIMESTAMP, 
	geom       GEOMETRY,
	geom_3035  GEOMETRY   -- (optional) same geometry on a different SRID
);

COPY INTO ais_navigation(mmsi, speed, lon, lat, time_epoch, time_stamp, geom) 
FROM '/path/to/data/ais_navigation.csv' (mmsi, speed, lon, lat, time_epoch) 
DELIMITERS ',', '\n', '"' NULL AS '';
```

**Note**: the previous example is not creating timestamp or geometric
objects, as they need to be derived from the raw data. To learn how to
transform the raw timestamp and geometric data into TIMESTAMP and
GEOMETRY objects, read the ["Pre-processing spatio-temporal
data"](#handling_st_data) section below.

### Importing ShapeFiles

If you have data in the ESRI Shapefile format, you can easily import it
with the `ShpLoad()` procedure. This procedure will create a new table
and fill it with all the information in the shapefile, including
`GEOMETRY` objects. You just need to provide the path to the *.shp* file
plus the schema and table name of the new table.

```sql
-- ShpLoad(path STRING, schema STRING, table STRING);
call shpload('/path/to/data/shpfile.shp','sys','shp_table');
``` 

### Importing WKT and WKB geometries

If you want to load geometries represented in the Well-Known Text
([WKT](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry))
format, use the `ST_GeomFromText()` function. First, you need to import
your WKT representation as a string object. Then, you can use
`ST_GeomFromText()` to create the `GEOMETRY` objects. Make sure to also
set the SRID of your geometries (`ST_SetSRID()`).

```sql
CREATE TABLE wkt_import (
	wkt_string	STRING,
	wkt_geom	GEOMETRY
);

COPY INTO wkt_import(wkt_string,wkt_geom)
FROM '/path/to/data/wkt_import.csv' (wkt_string)
DELIMITERS ',', '\n', '"' NULL AS '';

UPDATE wkt_import SET wkt_geom = ST_GeomFromText(wkt_string);
UPDATE wkt_import SET wkt_geom = ST_SetSRID(wkt_geom, 4326);
```

If you want to load geometries in their binary format, Well-Known Binary
(WKB), you can use the same method or use the `ST_WKBToSQL()` function.

## Pre-processing spatio-temporal data <a name="handling_st_data"></a>

We can use simple types to store spatio-temporal data (like `INT`), but
by using the respective data types for spatial (`GEOMETRY`) and temporal
(`TIMESTAMP`) data, we can leverage the storage and processing
capabilities of MonetDB to their fullest. If you intend to use temporal
and spatial filters when retrieving data, or wish to process the data
before you use it (e.g. aggregating ship positions into trajectories),
you must transform the raw spatio-temporal data.

To add timestamp data to your table, derived from UNIX epoch data, use:
```sql
UPDATE TABLE ais_navigation SET time_stamp = epoch(cast(time_epoch as int));
```

To add geometry data based on latitude and longitude and set the default
SRID, use:
```sql
UPDATE ais_navigation SET geom = st_setsrid(st_point(lon, lat), 4326);
```

If you want to transform your geometry from its original SRID into
another representation, use:
```sql
UPDATE ais_navigation SET geom_3035 = st_transform(geom, 3035);
```

### Geometry vs Geography

The difference between handling cartesian and geodetic data is an
important but subtle distinction. If your use case uses a cartesian
reference system that represents data in a planar space, you are working
with *geometric* data. If you instead work on the spherical space, you
are working with *geography* (or geodetic) data. This distinction has
important consequences on the type of processing that must be done to
calculate predicates like the distance between points. 

For some operations (e.g. `ST_Distance`), you have two versions
available: the *geometric* operation, `ST_Distance`, that will calculate
cartesian distances, and the *geographic* operation,
`ST_DistanceGeographic`, that will return geodetic distances. You can
find some of the supported geography functions below. All operations
that do not depend on the reference system can be used on both types of
data.

Geodetic operations produce more accurate results but require more
complex processing, leading to a longer processing time. If the accuracy
needs of your use case allow for the use of a cartesian (geometric)
projection, your performance will benefit from their use.

While this distinction is crucial for the correct results of operations
which depend on the reference system, both representations work on top
of the `GEOMETRY` database object. This means that you can store both
representations under the same database type.

**Note**: To ensure correct results, make sure you are using the correct
function name when dealing with geodetic data.

## Reading data<a name="reading_data"></a>

After your data is stored, you want to retrieve it for use in your
program. The simplest way to do this is to retrieve all the information
in the table (e.g. `SELECT * FROM table`), but this is usually not
necessary. Using standard SQL you can filter the amount of rows and
columns you retrieve. For filtering on spatio-temporal data, you can use
the features below.

### Temporal filters

Retrieve all ship positions from January 2022:
```sql
SELECT geom 
FROM ais_navigation
WHERE time_stamp 
    BETWEEN timestamp '2022-01-01 00:00:00.000'
    AND	    timestamp '2022-01-31 23:59:59.999';
```

More info on [time types](https://www.monetdb.org/documentation/user-guide/sql-manual/data-types/temporal-types/)
and [time processing functions](https://www.monetdb.org/documentation/user-guide/sql-functions/date-time-functions/).

### Spatial filters

Retrieving information using spatial filters is a more complicated
affair, and you may need to use geospatial functions to fit your needs.

#### Distance within

Using the `ST_DWithin()` and `ST_DWithinGeographic()` functions, you can
retrieve data based on how close ship positions are to a relevant
geometric object. This can be useful for getting data for ships that are
close to ports, for example.

```sql
-- Example: getting ship positions for all ships within 100 meters from
-- a port location (-4.853 48.038)
SELECT geom
FROM ais_navigation
WHERE ST_DWithinGeographic(
    geom,                                     --ship geometry column
    st_setsrid(st_point(-4.853 48.038),4326), --geometry of port with 4326 SRID
    100                                       --100 meters
);
```

#### Intersects

By using the `ST_Intersects()` and `ST_IntersectsGeographic()`
functions, we can get data that intersects other relevant geometries
(meaning, is inside or on the border). This is the same as using
`ST_DWithin()` with distance 0.

```sql
-- Example: getting ship positions for all ships intersecting fishing areas (polygons)
SELECT ais_navigation.geom 
FROM ais_navigation
JOIN fishing_areas
WHERE ST_IntersectsGeographic(
    ais_navigation.geom, --ship geometry column
    fishing_areas.geom   --fishing areas geometry column
);				
```

#### Simple latitude/longitude comparisons

If you need to get data from a specific latitude and longitude range,
you don't need to use distance-calculating functions like the two
examples above. A simple value comparison will give you even faster
results.

```sql
SELECT ais_navigation.geom
FROM ais_navigation
WHERE
	lat < -9.7 AND lat > -9.5 AND
	lon > 45.0 AND lon < 47.0;
```

## Processing data

Besides allowing for the filtering of data on spatio-temporal
conditions, MonetDB allows you to derive new data that may be useful for
your use case. You can calculate distances from relevant objects or
calculate the length of a ship trajectory using geometric/geographic
functions, and either get the derived data immediately or store it in a
table for later use. This section will only focus on the geometric data
processing, but you can visit the [SQL functions
documentation](https://www.monetdb.org/documentation/user-guide/sql-functions/)
to find out how you can pre-process your non-geometric data inside the
database.

### Geometric functions index

- `ST_Distance(geometry_1 GEOMETRY, geometry_2 GEOMETRY) RETURNS
  DOUBLE`: Calculate (Cartesian) distance between two geometric objects.
  `ST_DistanceGeographic()` is also available, and should be used to
  calculate geodetic distances.
- `ST_DistanceWithin(geometry_1 GEOMETRY, geometry_2 GEOMETRY,
  distance_meters DOUBLE) RETURNS BOOLEAN`: Returns TRUE if the two
  geometries are within a given distance in meters.
  `ST_DWithinGeographic()` is also available. 
- `ST_Intersects(geometry_1 GEOMETRY, geometry_2 GEOMETRY) RETURNS
  BOOLEAN`: Returns TRUE if one of the geometries is within the other
  (inside or on the border). `ST_IntersectsGeographic()` is also
  available.
- `ST_Collect(geometry_group GEOMETRY AGGREGATE) RETURNS GEOMETRY`:
  *Aggregate Function* Joins a group of geometries together into a
  single multi geometry object.
- `ST_Transform(geometry GEOMETRY, new_srid INT) RETURNS GEOMETRY`:
  Projects the original geometry into a new reference system, given in
  the `new_srid` parameter. Can be used to convert geodetic data into
  geometric data.

### Use Case: Aggregating trajectories

You may want to transform ship position data (represented as points)
into a line geometry containing all of the positions over time. This
will allow you to do operations on the position of a ship over a period
of time, such as calculating the distance travelled by a ship during a
voyage. It is also a more compact way of storing position data.

Here is how you can transform single point positions into a line object
with the positions of a ship ordered in time.

First, create segments by connecting consecutive positions of each ship.
```sql
CREATE TABLE trajectory_segments AS
  SELECT mmsi, t1, t2, st_makeline(p1, p2) as segment
  FROM (
    SELECT mmsi, LEAD(mmsi) OVER (ORDER BY mmsi, t) as mmsi2,
           t AS t1, LEAD(t) OVER (ORDER BY mmsi, t) as t2,
           geom AS p1, LEAD(geom) OVER (ORDER BY mmsi, t) as p2
    FROM ais_dynamic) as q1
  WHERE mmsi = mmsi2;
```

Then, aggregate the segments into trajectories.
```sql
CREATE TABLE trajectory AS
  SELECT mmsi, st_collect(segment) as geom
  FROM trajectory_segments
  GROUP BY mmsi;
```

### Use Case: Calculate distance of ships to ports

Calculating distances is as easy as using the `ST_Distance` call in your
query. But you need to know if your use case needs the accuracy of a
geodetic distance, or if a projected cartesian distance is enough. The
use of geodetic operations may give you the most accurate results, but
it is significantly slower due to the trigonometric functions it
requires.

To calculate the geodetic distance between the positions of a certain
ship over an one hour window, we can use this query:

```sql
SELECT ST_DistanceGeographic(ais_navigation.geom, port.geom)
FROM ais_navigation
JOIN port
WHERE ais_navigation.mmsi = 999
    AND ais_navigation.time_stamp
        BETWEEN timestamp '2022-01-01 12:00:00.000'
            AND timestamp '2022-01-01 13:00:00.000';
```

If it turns out that the accuracy given by projected operations is
acceptable for your use case, you can first transform your geodetic
(lat/lon) data into geometric (x,y) data using an appropriate
projection. After you project your data, you can use the following query
to calculate the same distances:

```sql
SELECT ST_Distance(ais_navigation.geom3035, port.geom)
FROM ais_navigation
JOIN port
WHERE ais_navigation.mmsi = 999
    AND ais_navigation.time_stamp
        BETWEEN timestamp '2022-01-01 12:00:00.000'
            AND timestamp '2022-01-01 13:00:00.000';
```

## Exporting data

When you query geometric data, you will get the textual representation
(WKT) by default. This can be useful if your program is also working on
top of Well-Known Text, but usually is not the best option. If you want
to receive a binary representation of the geometry, meaning a Well-Known
Binary, you can use the `ST_AsBinary()` function in your SELECT query.

```sql
SELECT ST_AsBinary(geom) as binary_representation,
       geom as textual_representation
FROM ais_navigation;
```

If you want to convert a binary representation to a text representation,
use the `ST_AsText()` function.

## Geometric Functions Support

### OGC SF Types
- Point
- Linestring
- Polygon
- MultiPoint
- MultiLinestring
- MultiPolygon
- GeometryCollection
 
### OGC SF Functions
#### Input/Output
- ST_WKTToSQL/ST_GeomFromText
- ST_WKBToSQL
- ST_AsText
- ST_AsBinary
- ST_Point/ST_MakePoint
- ST_MakeLine
- ST_AsEWKT

#### Functions on Geometries
- ST_Dimension
- ST_GeometryType
- ST_SRID
- ST_SetSRID
- ST_IsEmpty
- ST_IsSimple
- ST_Boundary
- ST_Envelope
- ST_Intersection
- ST_Difference
- ST_Union
- ST_SymDifference
- ST_Buffer
- ST_ConvexHull
- ST_Equals
- ST_Disjoint
- ST_Touches
- ST_Crosses
- ST_Within
- ST_Contains
- ST_Overlaps
- ST_Relate
- ST_Distance

#### Functions on Point
- ST_X
- ST_Y
- ST_Z

#### Functions on Linestring
- ST_StartPoint
- ST_EndPoint
- ST_IsRing
- ST_Length
- ST_IsClosed
- ST_NumPoints
- ST_PointN

- ST_Centroid
- ST_PointOnSurface
- ST_Area

#### Functions on Polygon
- ST_ExteriorRing
- ST_SetExteriorRing
- ST_NumInteriorRing
- ST_InteriorRingN
- ST_InteriorRings

#### Functions on GeometryCollection
- ST_NumGeometries
- ST_GeometryN

### Other supported functions
- ST_Distance
- ST_DistanceGeographic
- ST_DWithin
- ST_DWithinGeographic
- ST_Intersects
- ST_IntersectsGeographic
- ST_Transform
- ST_Collect

- ST_CoordDim
- ST_IsValid
- ST_NPoints
- ST_NRings
- ST_NumInteriorRings
- ST_XMax
- ST_XMin
- ST_YMax
- ST_YMin
- ST_Segmentize
- ST_Translate
- ST_Covers
- ST_CoveredBy
- ST_DelaunayTriangles
- ST_Dump
- ST_DumpPoints

