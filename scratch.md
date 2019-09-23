# Solving Spatial Problems with Queries

Which facilities are unique to a single park?

```sql
SELECT sitename, f_type
FROM park_facility_point
WHERE sitetype = 'Park Site'
GROUP BY f_type
HAVING COUNT(DISTINCT sitename) = 1 
ORDER BY sitename;
```

mountain peaks you can reach
bathrooms, bike racks, kids stuff

closest drop box

```sql
select precinct, dropbox, dbdist
from
  (select
  v.name as precinct
  , d.name as dropbox
  , st_distance(st_pointonsurface(v.geometry), d.geometry) as dbdist
  , ROW_NUMBER() OVER (
    PARTITION BY v.name
    ORDER BY st_distance(st_pointonsurface(v.geometry), d.geometry)
    ) as rank
  from votdst_area as v
  cross join ballot_dropbox_point as d)
  as precinct_dropbox
where rank=1
```

which voter precincts are the farthest from a ballot drop box?
color code map by nearest dropbox
draw lines from precinct to dropbox

dropbox areas using voronoj

```sql
-- example 
SELECT ST_VoronojDiagram(ST_Collect(geometry))
FROM italy_populated_places;
```

```sql
-- this returns a table of all possible combinations
SELECT 
   A.id , 
   B.myValue, 
   MIN(Distance(A.Geometry, B.Geometry)) AS distance
FROM tableOne AS A, tableTwo AS B
GROUP BY A.id, B.myValue
```


```sql
-- using a subquery
SELECT categoryid,
       productid,
       productName,
       unitprice
FROM products a
WHERE unitprice = (
                SELECT MIN(unitprice)
                FROM products b
                WHERE b.categoryid = a.categoryid)
```


```sql
-- this one will be very efficient
SELECT
       t.precinct, t.dropbox, t.distance
FROM
	(
	SELECT
		p.NAME AS precinct
		, b.NAME AS dropbox
		, ST_Distance(ST_Centroid(p.geometry), b.geometry) AS distance
		, MIN(ST_Distance(ST_Centroid(p.geometry), b.geometry)) OVER (PARTITION BY p.NAME)
			AS min_distance 
	FROM
		precincts AS p
		, ballot_dropboxes AS b
	) t
WHERE distance = min_distance
```


```sql
SELECT feature_name, feature_class, ST_Distance(Geometry,
MakePoint(-70.250, 43.802)) AS Distance
FROM XYGNIS
WHERE distance < 0.1
AND ROWID IN (
SELECT ROWID FROM SpatialIndex
WHERE f_table_name = 'XYGNIS'
AND search_frame =
BuildCircleMbr(-70.250, 43.802, 0.1))
ORDER BY distance LIMIT 10
```

join district data

# Misc notes to self

QGIS 3.8 uses Spatialite 4.3.0a.

[Spatialite Function Reference](http://www.gaia-gis.it/gaia-sins/spatialite-sql-4.3.0.html)

Show of hands on experience level
Tell story and present problem first
then develop queries that will get at that
"parks assessment"

connection between arcgis style tools and sql

- Why use this
- Tabular
  + Filter
  + Group (min, max)
  + Order
- Spatial
  + Buffer
  + Clip
  + Intersect
  + Merge
  + Union/Dissolve
  + Erase/Symmetrical difference
  + Centroids
  + Linear referencing
  + Make points from XY Table



