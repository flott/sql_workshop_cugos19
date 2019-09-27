# Solving Spatial Problems with Queries

CUGOS Fall Fling Workshop, 2019-10-06

## Session Description
*Solving Spatial Problems with Queries* is an introduction to using SQL with spatial extensions to create reports, summarize data, and explore spatial relationships. We will use [QGIS](https://www.qgis.org/) 3.8 to conduct SQL queries on tabular and spatial data. This workshop will focus on [Spatialite](https://www.gaia-gis.it/fossil/libspatialite/index), but the concepts are applicable to [PostGIS](https://postgis.net/) and to some extent, MySQL and SQL Server. Please bring your laptop with [QGIS 3.8](https://www.qgis.org/en/site/forusers/download.html) installed. Some familiarity with SQL, QGIS, and spatial concepts will come in handy, but beginners are welcome.

Data provided by permission of [King County](https://gis-kingcounty.opendata.arcgis.com/). 

## Agenda
1. What is SQL, and why use it for geospatial analysis?
3. Overview of geospatial databases
4. Overview of tools and [functions](http://www.gaia-gis.it/gaia-sins/spatialite-sql-4.3.0.html)
5. Examples
    - Parks assessment
    - Voting information
    - Floodplain analysis
6. Next steps

## QGIS / GeoPackage / Spatialite quirks

To use Spatialite functions on GeoPackages in QGIS, you need to run these queries first.

```sql
SELECT GetGpkgMode();  -- just a check. should return 0
SELECT EnableGpkgMode();  -- should return NULL
SELECT GetGpkgMode();  -- now it should return 1 (TRUE)
```

If spatial functions stop working, the "turn it off and on again" approach works. This tends to happen when you haven't run a query in a while -- I think QGIS resets the database connection, and GeoPackage mode is only active for the duration of a connection.

```sql
SELECT DisableGpkgMode(); -- should return NULL
SELECT GetGpkgMode();  -- should return 0
SELECT EnableGpkgMode();  -- should return NULL
SELECT GetGpkgMode();  -- now it should return 1 (TRUE)
```