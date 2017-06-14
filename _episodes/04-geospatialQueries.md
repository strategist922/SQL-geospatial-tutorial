---
title: "Geospatial Queries"
teaching: 10
exercises: 20
questions:
- "What are the various ways that we can represent geospatial information?"
- "How does a database store spatial information?"
objectives:
- "Learn the difference between geographic and projected coordinate systems"
- "Explore why understanding the coordinate reference frame matters when carrying out basic geospatial analysis"
- "Become familiar with some of the geospatial toolkits within PostGIS"
keypoints:
- "All geospatial analysis requires a knowledge of reference frames and coordinate systems"
- "many databases have extensions that encode geospatial information (e.g. PostGIS)"
- "PostGIS functions provide a wide range of tools to incorporate spatial analysis into your workflow"
---

## Calculating distances

* our last research question was: "What is the most common crime within 5 km of my house?"
* To answer this we'll need to understand something about mapping, and how databases encode spatial information.
* Note that the database currently has latitude and longitude information, for example:

~~~
SELECT latitude, longitude
FROM seattlecrimeincidents 
LIMIT 5;
~~~
{: .sql}

<br>
<hr>

| Latitude | Longitude |
| ---- | ---- |
| 47.6158384 | -122.3181689 |
| 47.60087709 | -122.3312162 |
| 47.59582098 | -122.3175691 | 
| 47.6140991 | -122.3174884 |
| 47.63148825 | -122.3125079 |

<hr>
<br>

* latitude and longitude are components of the _Geographic_ reference frame and are the most common ways to encode geospatial information:

<img src="../assets/img/databaseIntro/earthLatLong2.png" width = "600">

> ## What is the straight line distance between points 1 and 2?
> Point 1: 47.6158384, -112.3181689 
> 
> Point 2:  47.60087709, -112.3312162 
{: .challenge}

<!--
~~~
import numpy as np
latDist = 111.2 # one degree of latitude is 111.2 km
dLat = (47.6158384 - 47.60087709)
dLong = (-112.3181689 - -112.3312162)
distance = np.sqrt(dLat**2 + dLong**2) * latDist
print("Estimated distance is " + str(round(distance,3)) + " km")
~~~
{: .python}

~~~
Estimated distance is 2.207 km
~~~
{: .output}
-->

> ## Calculating distances in geographic coordinates
> * is it possible for us to calculate straight line distances directly from latitude and longitude data?
> * Why or why not?
> * Are distances between lines of latitude always the same? Between lines of longitude?
{: .discussion}

## Map Projections

* often we need to convert a 3D earth to a 2D plane
* this is necesary for accurate calculation of distances, bearing and area calculations

<img src="../assets/img/databaseIntro/projections2.png" width = "600">

<br><br><br>
* in a general sense, projection is similar to shining a light through a transparent sphere and tracing the lines of lat/long

<img src="../assets/img/databaseIntro/projections.png" width = "500">

<br><br><br>
* there are numerous different map projections, each optimized for different types of applications and calculations

<img src="../assets/img/databaseIntro/projectionTypes.png" width = "650">

<br>

## Encoding of geospatial information

* How does the database currently encode the latitude and longitude information?
* We can determine this by querying the database for information on the type of column variables:

~~~
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'seattlecrimeincidents' AND (column_name = 'Latitude' OR column_name = 'Longitude');
~~~
{: .sql}

* It would be better if the database understands latitude and longitude as locations rather than as double precision numbers.

> ## PostGIS: a geospatial database extension 
> * many databases, have extensions that allow for encoding of geospatial information
> * in the case of PostgreSQL this extension is called [PostGIS](http://postgis.net/)
> * our sample database already has the PostGIS extension installed and enabled
{: .callout}

## Database Geometries

* encoding of geospatial information occurs within a new _geometry_ column:

~~~
ALTER TABLE seattlecrimeincidents ADD COLUMN geom geometry(Point, 4326);
~~~
{:. sql}

* this adds an empty geometry column:

| Point | Latitude | Longitude | geom | 
| ---- | ---- | ---- | ---- |
| 1 | 47.6158384| -112.3181689 | |
| 2 | 47.60087709 | -112.3312162 | | 


* note that '4326' is the code for the geographic (latitude/longitude) coordinate system 
* Next, populate the geometry column:

```SQL
UPDATE seattlecrimeincidents SET geom = ST_setSRID(ST_MakePoint(longitude,latitude),4326);
```

| Point | Latitude | Longitude | geom | 
| ---- | ---- | ---- | ---- |
| 1 | 47.6158384| -112.3181689 | 0101000020E6100000AD0617E15C945EC07CCFEDCAD3CE4740 |
| 2 | 47.60087709 | -112.3312162 | 0101000020E6100000F2B96EA532955EC09A3B5D8AE9CC4740 | 

* the actual content of the _geom_ column is a binary string read by PostGIS
* Now that the database has geometric encoding, we can use a wide range of geospatial functions
<br>

## Geospatial functions

* often we receive geospatial information in various different projections and/or coordinate systems
* PostGIS provides mechanisms for us to _transform_ between different systems
* here we will transform our geometries into a _projected_ coordinate system so that we can calculate distances  

<br>
* first we add a new column that has a projected geometry
* in this case we will use 'Universal Transverse Mercator Zone 10 North', which has an ID of 3717:

```SQL
ALTER TABLE seattlecrimeincidents 
ADD COLUMN geom_utm geometry(Point, 3717);
```

* now we carry out the transformation

```SQL
UPDATE seattlecrimeincidents 
SET geom_utm = ST_Transform(geom,3717);
```
* note that all PostGIS functions begin with "ST" which means "Spatial Tool"
* now we can recalculate our distance using a built-in PostGIS distance function:

```SQL
SELECT ST_Distance(a.geom_utm,b.geom_utm)
FROM seattlecrimeincidents AS a, seattlecrimeincidents AS b
WHERE a.gid=1 AND b.gid=2;
```

* the result is:

| st_distance |
| ---- |
| 1930.45436426609 |

> ## Compare results
> How does this result compare to the distance we calculated from the latitude/longitude data alone?
> What explains the difference?
{: .discussion}


> ## What is the most common crime within 1 km of my house?
> 1. Find the coordinates of your house (or some other feature in the Seattle region).
> HINT: you can use google maps to find the latitude and longitude of any map location
> 2. Build the SQL query. You will need to nest 4 different functions to do this:
> * ST_MakePoint: to make a vector point geometry from your lat/long coordinate pair
> * ST_SetSRID: to tell the database which SRID your lat/long pair conforms to
> * ST_Transform: to transform your point geomtery to a projected coordinate system (use UTM Zone 10, '3717')
> * ST_Distance: to calculate the distance between all the geometries in the database and your single point
{: .challenge}