# Parks Assessment

## Table-based queries

### Basic selections

Let's look at some Parks data! We'll start with simple tabular queries.

```sql
SELECT * 
FROM mtpeaks_point
```

What if you don't want to return everything? You can choose the columns. Treat these things like verb/noun syntax.

```sql
-- sometimes column names are case sensitive. Best to match what you see.
SELECT name, elev
FROM mtpeaks_point
```

### Aliases
You can use `AS` to give an alias to a table or column name. If you want to use a space, you have to surround it in quotes. When we start using multiple tables, this saves a lot of typing because you have to specify the table name when column names are the same (like "geom").

*Return the elevations in meters instead of feet*
(3.28084 feet in a meter)

```sql
SELECT p.name AS "Peak Name", p.ELEV/3.28084 AS elev_m
FROM mtpeaks_point p  
```

### Ordering
```sql
SELECT name, elev
FROM mtpeaks_point
ORDER by elev  -- ascending (ASC) is the default.
```

```sql
-- change to descending order
SELECT name, elev
FROM mtpeaks_point
ORDER by elev DESC
```

### Limiting

*Just show the five highest peaks*
```sql
SELECT name, elev
FROM mtpeaks_point
ORDER by elev DESC
LIMIT 5
```

### Filtering

*Find peaks above 5000 feet*

```sql
SELECT name, elev
FROM mtpeaks_point
WHERE elev > 5000
```

*Filter park facility points to just show King County Parks assets*

```sql
SELECT KC_Fac_FID, f_name, f_type, sitename
FROM facilities
WHERE sitetype = 'Park Site'
AND ownertype='King County Parks'
```

### Joining

Join tables on some condition that evaluates to `TRUE`.

```sql
SELECT f.F_NAME
    , p.KCPARKFID
    , p.SITENAME
FROM park_facility_point f
JOIN park_area p
ON f.SiteName = p.SITENAME
```

### Grouping

When you group data, you can use **aggregate functions** like `MIN()`, `MAX()`, `AVG()`, or `COUNT()`.

*What facility types are in the park, and how many of each?*

```sql
SELECT f_type AS facility, COUNT(f_type) AS facility_count
FROM facilities
WHERE sitetype = 'Park Site'
GROUP BY f_type
ORDER BY facility_count DESC
```

*Which facilities are unique to a single park?*

```sql
SELECT sitename, f_type
FROM facilities
WHERE sitetype = 'Park Site'
GROUP BY f_type
HAVING COUNT(DISTINCT sitename) = 1 
ORDER BY sitename;
```

## Simple spatial queries

For details on the spatial relationships, the [PostGIS documentation](https://postgis.net/docs/reference.html#Spatial_Relationships_Measurements) has good examples, or the Wikipedia articles on [Spatial Relation](https://en.wikipedia.org/wiki/Spatial_relation) or [DE-9IM](https://en.wikipedia.org/wiki/DE-9IM).

### Fun park stats

*What's the biggest park owned and managed by King County?*

```sql
SELECT
	SITENAME
	, ST_Area(geom) / 43560 AS acres
FROM park_area
WHERE
	SITETYPE = 'Park Site'
	AND OWNERTYPE = 'King County Parks'
	AND MANAGETYPE = 'King County Parks'
ORDER BY acres DESC
```
Go down to the bottom of the list. *What's that tiny park?!*

### Trail info

*How many miles of trails are there in King County?*

```sql
SELECT SUM(ST_Length(t.geom)) / 5280 AS trail_miles
FROM trail_line t
```

*How many miles of trail are there by the surface type?*
```sql
SELECT Surf_Type AS "Surface Type"
, ROUND(SUM(ST_Length(t.geom)) / 5280,1) AS "Length (miles)"
FROM trail_line t
GROUP BY Surf_Type
```

### Finding mountain peaks

*Which peaks are inside the Alpine Lakes Wilderness?*

```sql
SELECT m.name, p.sitename
FROM mtpeaks_point m
, park_area p
WHERE ST_Within(m.geom, p.geom) = 1 
AND p.sitename = 'Alpine Lakes Wilderness'
```

You can also use spatial relationships in `JOIN` statements.

```sql
SELECT m.name, p.sitename
FROM mtpeaks_point m
JOIN park_area p
ON ST_Within(m.geom, p.geom) = 1 
WHERE p.sitename = 'Alpine Lakes Wilderness'
```

*What peaks are close to trails?*

```sql
-- DISTINCT gets rid of duplicates and shows only unique rows
SELECT DISTINCT peaks.name, peaks.elev, trails.trail_name
FROM mtpeaks_point peaks
JOIN trail_line trails
ON ST_Distance(peaks.geom, trails.geom) < 100
```
