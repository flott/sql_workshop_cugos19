# Parks Assessment

## Table-based queries

*Filter facility points to just show King County Parks assets*
```sql
SELECT KC_Fac_FID, f_name, f_type, sitename
FROM facilities
WHERE sitetype = 'Park Site'
AND ownertype='King County Parks'
```

*What facility types are in the park?*
```sql
SELECT f_type AS facility, count(f_type) AS facility_count
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

*Which peaks are inside King County Parks?*
```sql
--TODO
```

*What peaks are close to trails?*
```sql
select distinct peaks.name, peaks.elev, trails.trail_name
from peaks
join trails
on st_distance(peaks.geometry, trails.geometry) < 500
```
