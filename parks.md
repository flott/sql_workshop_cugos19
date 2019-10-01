# Parks Assessment

## Table-based queries

### Basic selections

Let's look at some Parks data! We'll start with simple tabular queries.

```sql
SELECT * 
FROM peaks
```

What if you don't want to return everything? You can choose the columns. Treat these things like verb/noun syntax.

```sql
SELECT name, elev
FROM peaks
```

### Aliases
You can use `AS` to give an alias to a table or column name. If you want to use a space, you have to surround it in quotes. When we start using multiple tables, this saves a lot of typing because you have to specify the table name when column names are the same (like "geom").

*Return the elevations in meters instead of feet*
(3.28084 feet in a meter)

```sql
SELECT p.name AS "Peak Name", p.ELEV/3.28084 AS "Elevation (m)"
FROM peaks AS p  
```

### Ordering
```sql
SELECT name, elev
FROM peaks
ORDER by elev
```
Ascending (`ASC`) is the default.

Change to descending order with `DESC`
```sql
SELECT name, elev
FROM peaks
ORDER by elev DESC
```

### Limiting

*Just show the five highest peaks*
```sql
SELECT name, elev
FROM peaks
ORDER by elev DESC
LIMIT 5
```

### Filtering

*Find peaks above 5000 feet*

```sql
SELECT name, elev
FROM peaks
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
SELECT facilities.F_NAME
    , parks.KCPARKFID
    , parks.SITENAME
FROM facilities
JOIN parks
	ON facilities.SiteName = parks.SITENAME
WHERE facilities.f_type = 'Picnic Area'
```
you can filter results based on columns that aren't in your output table!

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
	, ST_Area(geometry) / 43560 AS acres
FROM parks
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
SELECT SUM(ST_Length(trails.geometry)) / 5280 AS trail_miles
FROM trails
```

*How many miles of trail are there by the surface type?*
```sql
SELECT Surf_Type AS "Surface Type"
	, ROUND(SUM(ST_Length(trails.geometry)) / 5280,1) AS "Length (miles)"
FROM trails
GROUP BY Surf_Type
```

### Finding mountain peaks

*Which peaks are inside the Alpine Lakes Wilderness?*

```sql
SELECT peaks.name, peaks.elev
FROM peaks
	, parks
WHERE ST_Within(peaks.geometry, parks.geometry) = 1 
AND parks.sitename = 'Alpine Lakes Wilderness'
```

You can also use spatial relationships in `JOIN` statements.

```sql
SELECT peaks.name, peaks.elev
FROM peaks
JOIN parks
  ON ST_Within(peaks.geometry, parks.geometry) = 1 
WHERE parks.sitename = 'Alpine Lakes Wilderness'
```

*What peaks are close to trails?*

`DISTINCT` gets rid of duplicates and shows only unique rows
```sql
SELECT DISTINCT peaks.name, peaks.elev, trails.trail_name
FROM peaks
JOIN trails
  ON ST_Distance(peaks.geometry, trails.geometry) < 100
```

*What parks can I get to from the Snoqualmie Valley Trail?*
Be aware of your data! If it were just a line, we could use `ST_Crosses`, but the Snoqualmie Valley Trail right-of-way is listed as its own park site, so the results wouldn't be very exciting. Instead, let's take the Snoqualmie Valley Trail park site and see what's adjacent to it.

```sql
SELECT DISTINCT
	SiteName
FROM
	parks AS p
JOIN (
	SELECT fid, geometry
	FROM parks
	WHERE SiteName = 'Snoqualmie Valley Trail Site'
	) AS svt
  ON ST_Touches(p.geometry, svt.geometry)
    AND SiteType = 'Park Site'
```