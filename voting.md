# Voter precinct analysis


## Matching precincts to districts
Let's look at a precinct to see what legislative district it's in.

```sql
SELECT p.name
    , w.legdst
FROM precincts p
    , wa_leg_districts w
WHERE p.votdst = '2084'
    AND ST_Intersects(p.geometry, w.geometry) = 1
```

We got two different legislative districts because the geometry isn't drawn perfectly. 
Since no single precinct is split, we can just use any internal point to do the intersections. Not a centroid, though, because that might fall outside the polygon. We can use something called Point on Surface, which just puts a point somewhere inside the polygon.

```sql
SELECT p.name
    , w.legdst
FROM precincts p
    , wa_leg_districts w
WHERE p.votdst = '2084'
    AND ST_Within((ST_PointOnSurface(p.geometry)), w.geometry) = 1
```

## Nearest neighbor analysis for ballot dropboxes

Nearest neighbor analysis is a very common geospatial analysis. It answers the question "What's the closest thing?". Sometimes it's within a single dataset, like "What cities are closest to this city?" or, in this case, it's "For every location in *A*, what's the closest location from *B*?".

We can speed up some queries by setting a limit on how far away to look.
We'll look at our distance assumption with a virtual layer.

Also check how many unique precincts are in Seattle to make sure the count is correct.

```sql
SELECT COUNT(fid)
FROM precincts
WHERE name LIKE 'SEA %'
```

We'll get fancier here and use something called a **window function** -- that's the part there with `OVER (PARTITION BY p.NAME)`. It's a way of analyzing sub-groups of data without actually doing a `GROUP BY`. In this instance, we're adding two columns to our subquery table. The first is `distance`, which is the distance between the centroid of the precinct and the dropbox. The second column uses a window function to say "For *this particular precinct*, what is the minimum distance to a dropbox that I see?" and put the result in that column. After that, we just want to list each precinct and show the dropbox which matches that minimum distance.

```sql
SELECT
    precinct, dropbox, distance
FROM
  (
  SELECT
    p.NAME AS precinct
    , b.NAME AS dropbox
    , ST_Distance(ST_Centroid(p.geometry), b.geometry) AS distance
    , MIN(ST_Distance(ST_Centroid(p.geometry), b.geometry)) 
        OVER (PARTITION BY p.NAME) AS min_distance 
  FROM
    precincts AS p
    , ballot_dropboxes AS b
  WHERE
    p.NAME LIKE 'SEA %'
    AND distance < 5280 * 3
  )
WHERE distance = min_distance
ORDER BY distance DESC
```