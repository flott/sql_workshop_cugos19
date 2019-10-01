# North Bend Floodplain analysis


## Virtual layers and geometry operators
For this example we only care about North Bend. Everyone else can go kick rocks!
Let's use virtual layers and a query to clip the floodplain to the city limits.

```sql
SELECT f.FID, f.FLD_ZONE
, ST_Intersection(f.geometry, c.geometry) AS geometry
FROM city c, floodplain f
```

The `ST_Union` function will dissolve the geometries.
```sql
SELECT 1 AS fid
, ST_Union(ST_Intersection(f.geometry, c.geometry)) AS geometry
FROM city c, floodplain f
```

## Summarizing data based on spatial relationships

Let's say you're writing an article about flood risk in North Bend. You want to describe the affected properties.

First, let's look at the parcel table. There's a ton of data! What looks useful to us?
Always be sure to look at [the metadata](http://www5.kingcounty.gov/sdc/Metadata.aspx?Layer=parcel_address) to know what you're dealing with.

- `PIN`: Unique parcel ID number
- `PROP_NAME`: Names of 
- `CTYNAME`: City name of property
- `KCTP_CIY`: City where the taxpayer lives
- `APPRLNDVAL`: Appraised value of the land
- `APPR_IMPR`: Appraised value of improvements (buildings, paving, etc)
- `PROPTYPE`: Property type, residential/commercial/etc
- `PREUSE_DESC`: Description of present use

For starters, how many parcels are in North Bend?
```sql
SELECT COUNT(fid)
FROM parcels
WHERE CTYNAME = 'NORTH BEND'
```
2653 parcels

```sql
SELECT PROPTYPE, COUNT(fid)
FROM parcels
WHERE CTYNAME = 'NORTH BEND'
GROUP BY PROPTYPE
```

Be sure to use distinct when doing counts based on spatial joins or intersections, becaues you can have multiple pieces of floodplain geometry intersect a single parcel.

```sql
SELECT COUNT(DISTINCT p.PIN)
FROM parcels p
JOIN floodplain f
	ON ST_Intersects(p.geometry, f.geometry) = 1
WHERE CTYNAME = 'NORTH BEND'
```
1139 parcels

Maybe that's alarmist. What about parcels completely contained in the floodplain instead?
Dissolve the floodplain into a single feature with `ST_Union`.

```sql
WITH floodplain_dissolved AS (
    SELECT ST_Union(geometry) as geometry
	FROM floodplain
    )
SELECT COUNT(DISTINCT p.PIN)
FROM parcels p
JOIN floodplain_dissolved f
	ON ST_Contains(f.geometry, p.geometry) = 1
WHERE CTYNAME = 'NORTH BEND'
```
964 parcels -- still a lot!

### Property types in the floodplain

We don't want to write all these numbers down! We can make the intersection query return 0 for false and 1 for true, then roll these up.

Here's the whole list.
```sql
SELECT DISTINCT
	p.PIN
	, p.PROPTYPE
	, ST_Intersects(p.geometry, f.geometry) as in_floodplain
FROM parcels p
	, floodplain f
WHERE CTYNAME = 'NORTH BEND'
```

Now summarized. We can't do math on a new column, so it has to be wrapped in a subquery.

```sql
SELECT
    CASE
        WHEN proptype = 'C' THEN 'Commercial'
        WHEN proptype = 'R' THEN 'Residential'
		WHEN proptype = 'K' THEN 'Condo'
        END AS "Property Type"
    , COUNT(PIN) AS "Total parcels"
    , SUM(in_floodplain) AS "Number in floodplain"
    , ROUND(CAST(SUM(in_floodplain) AS FLOAT) / CAST(COUNT(PIN) AS FLOAT) * 100,1) AS "Percent in floodplain"
FROM
    (
    SELECT DISTINCT
        p.PIN
        , p.PROPTYPE
        , ST_Intersects(p.geometry, f.geometry) as in_floodplain
    FROM 
        parcels p
        , floodplain f
    WHERE p.CTYNAME = 'NORTH BEND'
        AND p.PROPTYPE IN ('C','R','K')
    )
GROUP BY proptype
```
### Value of potentially affected properties

What's the appraised value of improvements for parcels that are in the floodplain?

```sql
SELECT
    SUM(APPR_IMPR)
FROM parcels p
JOIN
	(SELECT ST_Union(geometry) as geometry
	FROM floodplain) f
    ON ST_Intersects(p.geometry, f.geometry) = 1
WHERE p.CTYNAME = 'NORTH BEND'
```

*What are the top properties affected?*

Make the dissolved floodplain a CTE for clarity

```sql
WITH f AS (
    SELECT ST_Union(geometry) as geometry
	FROM floodplain
    )
SELECT
	p.PIN
	,p.PROP_NAME
    ,p.APPR_IMPR
    ,p.PREUSE_DESC
FROM parcels p
JOIN f
    ON ST_Intersects(p.geometry, f.geometry) = 1
WHERE APPR_IMPR > 0
    AND p.CTYNAME = 'NORTH BEND'
ORDER BY APPR_IMPR DESC
LIMIT 5
```
Uh oh, be careful, Nintendo!

### Present use breakdown of affected parcels

What about a breakdown of all the different property uses in the floodplain and their values?

```sql
WITH f AS (
    SELECT ST_Union(geometry) as geometry
	FROM floodplain
    )
SELECT
	PREUSE_DESC AS "Present Use"
	, SUM(APPR_IMPR) AS "Total Value"
FROM parcels p
JOIN f
    ON ST_Intersects(p.geometry, f.geometry) = 1
WHERE APPR_IMPR > 0
    AND p.CTYNAME = 'NORTH BEND'
GROUP BY PREUSE_DESC
ORDER BY "Total Value" DESC
```

### Finding homes in a specific flood zone

Let's find single family homes in shallow flooding areas, they might be good candidates for home elevation projects.

```sql
SELECT
    p.PIN
    , p.ADDR_FULL
FROM parcels p
JOIN floodplain f
    ON ST_Intersects(p.geometry, f.geometry) = 1
WHERE 
    p.CTYNAME = 'NORTH BEND'
    AND p.APPR_IMPR > 0
    AND PREUSE_DESC LIKE 'Single Family%'
    AND (f.FLD_ZONE = 'AH' OR f.FLD_ZONE = 'AO')
```

We can add this as a virtual layer, too if we return the geometry.
```sql
SELECT
    p.*
FROM parcels p
JOIN floodplain f
    ON ST_Intersects(p.geometry, f.geometry) = 1
WHERE 
    p.CTYNAME = 'NORTH BEND'
    AND p.APPR_IMPR > 0
    AND PREUSE_DESC LIKE 'Single Family%'
    AND (f.FLD_ZONE = 'AH' OR f.FLD_ZONE = 'AO')
```