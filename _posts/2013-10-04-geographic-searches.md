---
layout: post
title: Geographic searches within a certain distance
image: geographic-searches.png
description: A 2-dimensional location on our earth can be represented via a coordinate system similar to an X &amp; Y-axis. These axes are called latitude (lat) &amp; longitude (lng).
tags: [ geography, php, mysql ]
---

A 2-dimensional location on our earth can be represented via a coordinate system similar to an X & Y-axis. These axes are called latitude (lat) & longitude (lng).

Latitude is the north-south axis with a minimum of -90 (south pole) and maximum of 90 degrees (north pole). The equator is zero degrees latitude.

Longitude is the X-axis equivalent, running around the globe from east to west: from -180 to +180 degrees. The [Greenwich meridian](http://en.wikipedia.org/wiki/Prime_meridian) is 0 degrees longitude. Everything west and east from it is respectively negative and positive on the longitude scale, up until the middle of the Pacific Ocean, near the [International Date Line](http://en.wikipedia.org/wiki/International_Date_Line), where -180° longitude crosses over to 180°.

<!-- more -->

**Update**: I've created a small repository with all of the below code in a couple of neat little classes. If you're looking to calculate distance between multiple coordinates, or calculate a bounding box to find nearby coordinates in your database, it may make sense to [check it out](https://github.com/matthiasmullie/geo).

# Coordinates and kilometres/miles

You've always been told that Greenland is not as large as it is depicted on your average 2D map. It's about the size of Congo, but in a [2D projection](http://en.wikipedia.org/wiki/Mercator_projection), the earth's edges appear larger than they actually are.

Because the earth is (almost) spherical instead of 2-dimensional, the distance between 2 coordinates is not linear. The distance between 0 and 10 degrees longitude in reality is much shorter at the north pole, than on the equator.

To calculate bird's-eye distance between 2 coordinates, geometry finally comes in handy!

The earth's radius is approximately 6371 kilometres or 3959 miles. Said radius multiplied by the [great-circle distance](http://en.wikipedia.org/wiki/Great-circle_distance) calculated between the 2 coordinates mapped on a sphere, should yield the distance between both points.

```php
function distance($lat1, $lng1, $lat2, $lng2) {
    // convert latitude/longitude degrees for both coordinates
    // to radians: radian = degree * π / 180
    $lat1 = deg2rad($lat1);
    $lng1 = deg2rad($lng1);
    $lat2 = deg2rad($lat2);
    $lng2 = deg2rad($lng2);

    // calculate great-circle distance
    $distance = acos(sin($lat1) * sin($lat2) + cos($lat1) * cos($lat2) * cos($lng1 - $lng2));

    // distance in human-readable format:
    // earth's radius in km = ~6371
    return 6371 * $distance;
}
```

Please note that the earth is not exactly spherical: the earth's radius is slightly larger at the equator (~6378 km) than at the poles (~6356 km), so the exact distance we just calculated may be slightly off.

If instead of bird's-eye distance, you're looking to calculate road travel distance between 2 points, you're probably best off using [Google's distance matrix API](https://developers.google.com/maps/documentation/distancematrix/).

# Find nearby locations in database

While there are far superior solutions (like [ElasticSearch](http://www.elasticsearch.org)) to perform geographic searches, you might find your data stuck in a relational database, like [MySQL](http://www.mysql.com). MySQL also has a [SPATIAL extension](http://dev.mysql.com/doc/refman/5.7/en/spatial-extensions.html) to facilitate geography-based operations (though I haven't actually used it much, I actually find it easier dealing with the raw data myself.)

A common location- & distance-based search is a "find everything within a radius of X kilometres". There are multiple ways to do this. You could, for example, create a WHERE condition that mimics the aforementioned great-circle distance based formula to calculate the difference between every coordinate in your database, and the given point, to leave out all entries where the distance is greater than what you'd like. Once your database grows really large, you don't actually want to calculate the distance for every location in your database, though: it'll take some time to calculate all these differences & there is no way an index can be used.

Instead, we'll want to find a rough subset of results within certain fixed boundaries. These boundaries are the maximum and minimum latitude & longitude values of your coordinate plus/minus the distance. We can calculate them like:

```php
// we'll want everything within, say, 10km distance
$distance = 10;

// earth's radius in km = ~6371
$radius = 6371;

// latitude boundaries
$maxlat = $lat + rad2deg($distance / $radius);
$minlat = $lat - rad2deg($distance / $radius);

// longitude boundaries (longitude gets smaller when latitude increases)
$maxlng = $lng + rad2deg($distance / $radius / cos(deg2rad($lat)));
$minlng = $lng - rad2deg($distance / $radius / cos(deg2rad($lat)));
```

Now that we have these outer bounds, we can fetch results in our database like this (notice how an index can now be used to retrieve matching values for latitude/longitude):

```sql
SELECT *
FROM coordinates
WHERE
    lat BETWEEN :minlat AND :maxlat
    lng BETWEEN :minlng AND :maxlng
```

Or using the SPATIAL extension (no point in keeping `lat` and `lng` floats here; `coordinate` is a `Point`: `GeomFromText(CONCAT("Point(", :lng, " ", :lat, ")"))`):

```sql
WHERE MBRWithin(coordinate, GeomFromText(CONCAT("Polygon((", :maxlng, " ", :maxlat, ",", :minlng, " ", :maxlat, ",", :minlng, " ", :minlat, ",", :maxlng, " ", :minlat, ",", :maxlng, " ", :maxlat, "))")))
```

We have maxed out the speedy retrieval of coordinates, but not all matching coordinates actually fall within the distance we wanted to match. Using these boundaries, we've queried for a 2D square-like area, but we actually want to find results in a circle-like area. Here's an image to simplify why we're not yet done:

![Image showing how a point can fit the bounding box, but still be outside the radius]({{ site.baseurl }}public/posts/geographic-searches-circle.png)

The black box signifies the area we've just queried the database for. The orange circle represents what would actually be a real 10-kilometer boundary. Notice how both white coordinates are within the rough boundaries, but only the bottom one is actually within the requested distance.

To weed out these results that did fall into our rough boundaries, but are not actually within the desired area distance, let's just loop all of these entries and calculate the distance there. Because our resultset will now be pretty small, this shouldn't really hurt us:

```php
// our own location &amp; distance we want to search
$lat = 50.52;
$lng = 4.22;
$distance = 10;

// weed out all results that turn out to be too far
foreach ($results as $i => $result) {
    $resultDistance = distance($lat, $lng, $result['lat'], $result['lng']);
    if ($resultDistance > $distance) {
        unset($results[$i]);
    }
}
```

Tadaa! All coordinates within a given distance!
