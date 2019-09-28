# North Bend Floodplain analysis

For this example we only care about North Bend. Everyone else can go kick rocks!
Let's use virtual layers and a query to clip the floodplain to the city limits.

```sql
SELECT f.FID, f.FLD_ZONE
, ST_Intersection(f.geometry, c.geometry) AS geometry
FROM city c, floodplain f
```

The `ST_Union` function will dissolve the geometries 
```sql
SELECT 1 AS fid
, ST_Union(ST_Intersection(f.geometry, c.geometry)) AS geometry
FROM city c, floodplain f
```