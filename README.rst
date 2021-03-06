==============
timezonefinder
==============

.. image:: https://img.shields.io/travis/MrMinimal64/timezonefinder.svg?branch=master
    :target: https://travis-ci.org/MrMinimal64/timezonefinder

.. image:: https://img.shields.io/pypi/wheel/timezonefinder.svg
    :target: https://pypi.python.org/pypi/timezonefinder

.. image:: https://img.shields.io/pypi/v/timezonefinder.svg
    :target: https://pypi.python.org/pypi/timezonefinder
    
.. image:: https://anaconda.org/conda-forge/timezonefinder/badges/version.svg   
    :target: https://anaconda.org/conda-forge/timezonefinder


This is a fast and lightweight python project for looking up the corresponding
timezone for a given lat/lng on earth entirely offline.

The underlying timezone data is now based on work done by `Evan Siroky <https://github.com/evansiroky/timezone-boundary-builder>`__. Current version: 2017a (March 2017, since 2.1.0)
Previously: `tz_world <http://efele.net/maps/tz/world/>`__ 2016d (May 2016, until 2.0.2)


Also see:
`GitHub <https://github.com/MrMinimal64/timezonefinder>`__,
`PyPI <https://pypi.python.org/pypi/timezonefinder/>`__,
`conda-forge feedstock <https://github.com/conda-forge/timezonefinder-feedstock>`__,
`timezone_finder (ruby port) <https://github.com/gunyarakun/timezone_finder>`__


This project is derived from and has been successfully tested against
`pytzwhere <https://pypi.python.org/pypi/tzwhere>`__
(`github <https://github.com/pegler/pytzwhere>`__), but aims at providing
improved performance and usability.

``pytzwhere`` is parsing a 76MB .csv file (floats stored as strings!) completely into memory and computing shortcuts from this data on every startup.
This is time, memory and CPU consuming. Additionally calculating with floats is slow, keeping those 4M+ floats in the RAM all the time is unnecessary and the precision of floats is not even needed in this case (s. detailed comparison and speed tests below).


NOTE: the huge underlying timezone boundary data set (s. below) in use now blew up the size of this package. It had to be changed, because the smaller "tz_world" data set is not being maintained any more. I wanted to keep this as lightweight as possible, but at least the data it is up to date. 
In case size and speed matter more you than actuality, consider checking out older versions of timezonefinder or even timezoenfinderL. The timezone polygons also do NOT follow the shorelines any more (as they did with tz_world). This makes the results of closest_timezone_at() somewhat meaningless (as with timezonefinderL).


Check out the even faster and lighter (but outdated) version `timezonefinderL <https://github.com/MrMinimal64/timezonefinderL>`__
and its demo and API at: `timezonefinderL GUI <http://timezonefinder.michelfe.it/gui>`__


Dependencies
============

(``python``)
``numpy``

**Optional:**

``Numba`` (https://github.com/numba/numba) and its Requirement `llvmlite <http://llvmlite.pydata.org/en/latest/install/index.html>`_


This is only for precompiling the time critical algorithms. When you only look up a
few points once in a while, the compilation time is probably outweighing
the benefits. When using ``certain_timezone_at()`` and especially
``closest_timezone_at()`` however, I highly recommend using ``numba``!

Installation
============


Installation with conda: see instructions at `conda-forge feedstock <https://github.com/conda-forge/timezonefinder-feedstock>`__


Installation with pip:
in the command line:

::

    pip install timezonefinder





Usage
=====

Basics:
-------

in Python:

::

    from timezonefinder import TimezoneFinder

    tf = TimezoneFinder()


for testing if numba is being used:
(if the import of the optimized algorithms worked)

::

    TimezoneFinder.using_numba()   # this is a static method returning True or False


**timezone_at():**

This is the default function to check which timezone a point lies within (similar to tzwheres ``tzNameAt()``).
If no timezone has been found, ``None`` is being returned.

**PLEASE NOTE:** This approach is optimized for speed and the common case to only query points within a timezone.
The last possible timezone in proximity is always returned (without checking if the point is really included).
So results might be misleading for points outside of any timezone.


::

    longitude = 13.358
    latitude = 52.5061
    tf.timezone_at(lng=longitude, lat=latitude) # returns 'Europe/Berlin'


**certain_timezone_at():**

This function is for making sure a point is really inside a timezone. It is slower, because all polygons (with shortcuts in that area)
are checked until one polygon is matched.

::

    tf.certain_timezone_at(lng=longitude, lat=latitude) # returns 'Europe/Berlin'


**closest_timezone_at():**

Only use this when the point is not inside a polygon, because the approach otherwise makes no sense.
This returns the closest timezone of all polygons within +-1 degree lng and +-1 degree lat (or None).

::

    longitude = 12.773955
    latitude = 55.578595
    tf.closest_timezone_at(lng=longitude, lat=latitude) # returns 'Europe/Copenhagen'

Other options:
To increase search radius even more, use the ``delta_degree``-option:

::

    tf.closest_timezone_at(lng=longitude, lat=latitude, delta_degree=3)


This checks all the polygons within +-3 degree lng and +-3 degree lat.
I recommend only slowly increasing the search radius, since computation time increases quite quickly
(with the amount of polygons which need to be evaluated). When you want to use this feature a lot,
consider using ``Numba`` to save computing time.


Also keep in mind that x degrees lat are not the same distance apart than x degree lng (earth is a sphere)!
As a consequence getting a result does NOT mean that there is no closer timezone! It might just not be within the area being queried.

With ``exact_computation=True`` the distance to every polygon edge is computed (way more complicated), instead of just evaluating the distances to all the vertices.
This only makes a real difference when polygons are very close.


With ``return_distances=True`` the output looks like this:

( 'tz_name_of_the_closest_polygon',[ distances to every polygon in km], [tz_names of every polygon])

Note that some polygons might not be tested (for example when a zone is found to be the closest already).
To prevent this use ``force_evaluation=True``.

::

    longitude = 42.1052479
    latitude = -16.622686
    tf.closest_timezone_at(lng=longitude, lat=latitude, delta_degree=2,
                                        exact_computation=True, return_distances=True, force_evaluation=True)
    '''
    returns ('uninhabited',
    [80.66907784731714, 217.10924866254518, 293.5467252349301, 304.5274937839159, 238.18462606485667, 267.918674688949, 207.43831938964408, 209.6790144988553, 228.42135641542546],
    ['uninhabited', 'Indian/Antananarivo', 'Indian/Antananarivo', 'Indian/Antananarivo', 'Africa/Maputo', 'Africa/Maputo', 'Africa/Maputo', 'Africa/Maputo', 'Africa/Maputo'])
    '''



**get_geometry:**


for querying timezones for their geometric shape use ``get_geometry()``.
output format: ``[ [polygon1, hole1,...), [polygon2, ...], ...]``
and each polygon and hole is itself formated like: ``([longitudes], [latitudes])``
or ``[(lng1,lat1), (lng2,lat2),...]`` if ``coords_as_pairs=True``.

::

    tf.get_geometry(tz_name='Africa/Addis_Ababa', coords_as_pairs=True)

    tf.get_geometry(tz_id=400, use_id=True)




Further application:
--------------------

**To maximize the chances of getting a result in a** ``Django`` **view it might look like:**

::

    def find_timezone(request, lat, lng):
        lat = float(lat)
        lng = float(lng)

        try:
            timezone_name = tf.timezone_at(lng=lng, lat=lat)
            if timezone_name is None:
                timezone_name = tf.closest_timezone_at(lng=lng, lat=lat)
                # maybe even increase the search radius when it is still None

        except ValueError:
            # the coordinates were out of bounds
            # {handle error}

        # ... do something with timezone_name ...

**To get an aware datetime object from the timezone name:**

::

    # first pip install pytz
    from pytz import timezone, utc
    from pytz.exceptions import UnknownTimeZoneError

    # tzinfo has to be None (means naive)
    naive_datetime = YOUR_NAIVE_DATETIME

    try:
        tz = timezone(timezone_name)
        aware_datetime = naive_datetime.replace(tzinfo=tz)
        aware_datetime_in_utc = aware_datetime.astimezone(utc)

        naive_datetime_as_utc_converted_to_tz = tz.localize(naive_datetime)

    except UnknownTimeZoneError:
        # ... handle the error ...

also see the `pytz Doc <http://pytz.sourceforge.net/>`__.

**parsing the data:**


Download the latest .json from `GitHub <https://github.com/evansiroky/timezone-boundary-builder>`__.
Place it inside the timezonefinder folder and run the ``file_converter.py`` first only until the ``timezone_names.py`` have been updated.
Then abort and rerun ``file_converter.py``, this time until the compilation of the binary files is completed. (compilation need to import the new timezone_names.py file)


**Calling timezonefinder from the command line:**

With -v you get verbose output, without it only the timezone name is printed.
Internally this is calling the function timezone_at(). Please note that this is slow.

::

    python timezonefinder.py lng lat [-v]





Known Issues
============

I ran tests for approx. 5M points and the only differences (in comparison to tzwhere) are due to the outdated
data being used by tzwhere.


Contact
=======

Most certainly there is stuff I missed, things I could have optimized even further etc. I would be really glad to get some feedback on my code.


If you notice that the tz data is outdated, encounter any bugs, have
suggestions, criticism, etc. feel free to **open an Issue**, **add a Pull Requests** on Git or ...

contact me: *[python] {at} [michelfe] {dot} [it]*


Credits
=======

Thanks to:

`Adam <https://github.com/adamchainz>`__ for adding organisational features to the project and for helping me with publishing and testing routines.

`cstich <https://github.com/cstich>`__ for the little conversion script (.shp to .json).

`snowman2 <https://github.com/snowman2>`__ for creating the conda-forge recipe.

`synapticarbors <https://github.com/synapticarbors>`__ for fixing Numba import with py27.

License
=======

``timezonefinder`` is distributed under the terms of the MIT license
(see LICENSE.txt).


Comparison to pytzwhere
=======================

In comparison most notably initialisation time and memory usage are
significantly reduced, while the algorithms yield the same results and are as fast or even faster
(depending on the dependencies used, s. test results below). pytzwhere is using up to 450MB!!! of RAM while in use (with shapely and numpy active) and this package uses at most 40MB (= encountered memory consumption of the python process).
In some cases ``pytzwhere`` even does not find anything and ``timezonefinder`` does, for example
when only one timezone is close to the point.

**Similarities:**

-  results

-  data being used


**Differences:**

-  highly decreased memory usage

-  highly reduced start up time

-  usage of 32bit int (instead of 64+bit float) reduces computing time and memory consumption

-  the precision of 32bit int is still high enough (according to my calculations worst resolution is 1cm at the equator -> far more precise than the discrete polygons)

-  the data is stored in memory friendly binary files (approx. 41MB in total, original data 120MB .json)

-  data is only being read on demand (not completely read into memory if not needed)

-  precomputed shortcuts are included to quickly look up which polygons have to be checked

-  available proximity algorithm ``closest_timezone_at()``

-  function ``get_geometry()`` enables querying timezones for their geometric shape (= multipolygon with holes)

-  further speedup possible by the use of ``numba`` (code precompilation)



test results\*:
===============

::


    test correctness:

    results timezone_at()
    LOCATION             | EXPECTED             | COMPUTED             | ==
    ====================================================================
    Arlington, TN        | America/Chicago      | America/Chicago      | OK
    Memphis, TN          | America/Chicago      | America/Chicago      | OK
    Anchorage, AK        | America/Anchorage    | America/Anchorage    | OK
    Eugene, OR           | America/Los_Angeles  | America/Los_Angeles  | OK
    Albany, NY           | America/New_York     | America/New_York     | OK
    Moscow               | Europe/Moscow        | Europe/Moscow        | OK
    Los Angeles          | America/Los_Angeles  | America/Los_Angeles  | OK
    Moscow               | Europe/Moscow        | Europe/Moscow        | OK
    Aspen, Colorado      | America/Denver       | America/Denver       | OK
    Kiev                 | Europe/Kiev          | Europe/Kiev          | OK
    Jogupalya            | Asia/Kolkata         | Asia/Kolkata         | OK
    Washington DC        | America/New_York     | America/New_York     | OK
    St Petersburg        | Europe/Moscow        | Europe/Moscow        | OK
    Blagoveshchensk      | Asia/Yakutsk         | Asia/Yakutsk         | OK
    Boston               | America/New_York     | America/New_York     | OK
    Chicago              | America/Chicago      | America/Chicago      | OK
    Orlando              | America/New_York     | America/New_York     | OK
    Seattle              | America/Los_Angeles  | America/Los_Angeles  | OK
    London               | Europe/London        | Europe/London        | OK
    Church Crookham      | Europe/London        | Europe/London        | OK
    Fleet                | Europe/London        | Europe/London        | OK
    Paris                | Europe/Paris         | Europe/Paris         | OK
    Macau                | Asia/Macau           | Asia/Macau           | OK
    Russia               | Asia/Yekaterinburg   | Asia/Yekaterinburg   | OK
    Salo                 | Europe/Helsinki      | Europe/Helsinki      | OK
    Staffordshire        | Europe/London        | Europe/London        | OK
    Muara                | Asia/Brunei          | Asia/Brunei          | OK
    Puerto Montt seaport | America/Santiago     | America/Santiago     | OK
    Akrotiri seaport     | Asia/Nicosia         | Asia/Nicosia         | OK
    Inchon seaport       | Asia/Seoul           | Asia/Seoul           | OK
    Nakhodka seaport     | Asia/Vladivostok     | Asia/Vladivostok     | OK
    Truro                | Europe/London        | Europe/London        | OK
    Aserbaid. Enklave    | Asia/Baku            | Asia/Baku            | OK
    Tajikistani Enklave  | Asia/Dushanbe        | Asia/Dushanbe        | OK
    Busingen Ger         | Europe/Busingen      | Europe/Busingen      | OK
    Genf                 | Europe/Zurich        | Europe/Zurich        | OK
    Lesotho              | Africa/Maseru        | Africa/Maseru        | OK
    usbekish enclave     | Asia/Tashkent        | Asia/Tashkent        | OK
    usbekish enclave     | Asia/Tashkent        | Asia/Tashkent        | OK
    Arizona Desert 1     | America/Denver       | America/Denver       | OK
    Arizona Desert 2     | America/Phoenix      | America/Phoenix      | OK
    Arizona Desert 3     | America/Phoenix      | America/Phoenix      | OK
    Far off Cornwall     | None                 | None                 | OK

    certain_timezone_at():
    LOCATION             | EXPECTED             | COMPUTED             | Status
    ====================================================================
    Arlington, TN        | America/Chicago      | America/Chicago      | OK
    Memphis, TN          | America/Chicago      | America/Chicago      | OK
    Anchorage, AK        | America/Anchorage    | America/Anchorage    | OK
    Eugene, OR           | America/Los_Angeles  | America/Los_Angeles  | OK
    Albany, NY           | America/New_York     | America/New_York     | OK
    Moscow               | Europe/Moscow        | Europe/Moscow        | OK
    Los Angeles          | America/Los_Angeles  | America/Los_Angeles  | OK
    Moscow               | Europe/Moscow        | Europe/Moscow        | OK
    Aspen, Colorado      | America/Denver       | America/Denver       | OK
    Kiev                 | Europe/Kiev          | Europe/Kiev          | OK
    Jogupalya            | Asia/Kolkata         | Asia/Kolkata         | OK
    Washington DC        | America/New_York     | America/New_York     | OK
    St Petersburg        | Europe/Moscow        | Europe/Moscow        | OK
    Blagoveshchensk      | Asia/Yakutsk         | Asia/Yakutsk         | OK
    Boston               | America/New_York     | America/New_York     | OK
    Chicago              | America/Chicago      | America/Chicago      | OK
    Orlando              | America/New_York     | America/New_York     | OK
    Seattle              | America/Los_Angeles  | America/Los_Angeles  | OK
    London               | Europe/London        | Europe/London        | OK
    Church Crookham      | Europe/London        | Europe/London        | OK
    Fleet                | Europe/London        | Europe/London        | OK
    Paris                | Europe/Paris         | Europe/Paris         | OK
    Macau                | Asia/Macau           | Asia/Macau           | OK
    Russia               | Asia/Yekaterinburg   | Asia/Yekaterinburg   | OK
    Salo                 | Europe/Helsinki      | Europe/Helsinki      | OK
    Staffordshire        | Europe/London        | Europe/London        | OK
    Muara                | Asia/Brunei          | Asia/Brunei          | OK
    Puerto Montt seaport | America/Santiago     | America/Santiago     | OK
    Akrotiri seaport     | Asia/Nicosia         | Asia/Nicosia         | OK
    Inchon seaport       | Asia/Seoul           | Asia/Seoul           | OK
    Nakhodka seaport     | Asia/Vladivostok     | Asia/Vladivostok     | OK
    Truro                | Europe/London        | Europe/London        | OK
    Aserbaid. Enklave    | Asia/Baku            | Asia/Baku            | OK
    Tajikistani Enklave  | Asia/Dushanbe        | Asia/Dushanbe        | OK
    Busingen Ger         | Europe/Busingen      | Europe/Busingen      | OK
    Genf                 | Europe/Zurich        | Europe/Zurich        | OK
    Lesotho              | Africa/Maseru        | Africa/Maseru        | OK
    usbekish enclave     | Asia/Tashkent        | Asia/Tashkent        | OK
    usbekish enclave     | Asia/Tashkent        | Asia/Tashkent        | OK
    Arizona Desert 1     | America/Denver       | America/Denver       | OK
    Arizona Desert 2     | America/Phoenix      | America/Phoenix      | OK
    Arizona Desert 3     | America/Phoenix      | America/Phoenix      | OK
    Far off Cornwall     | None                 | None                 | OK

    closest_timezone_at():
    LOCATION             | EXPECTED             | COMPUTED             | Status
    ====================================================================
    Arlington, TN        | America/Chicago      | America/Chicago      | OK
    Memphis, TN          | America/Chicago      | America/Chicago      | OK
    Anchorage, AK        | America/Anchorage    | America/Anchorage    | OK
    Shore Lake Michigan  | America/New_York     | America/New_York     | OK
    English Channel1     | Europe/London        | Europe/London        | OK
    English Channel2     | Europe/Paris         | Europe/Paris         | OK
    Oresund Bridge1      | Europe/Stockholm     | Europe/Stockholm     | OK
    Oresund Bridge2      | Europe/Copenhagen    | Europe/Copenhagen    | OK


    Speed Tests:
    _________________________
    shapely: OFF (tzwhere)
    Numba: OFF (timezonefinder)


    Startup times:
    tzwhere: 0:00:07.875212
    timezonefinder: 0:00:00.000688
    11445.53 times faster

    _________________________
    shapely: ON (tzwhere)
    Numba: ON (timezonefinder)


    Startup times:
    tzwhere: 0:00:29.365294
    timezonefinder: 0:00:00.000888
    33068.02 times faster


    NOTE: all the other test are not expressive atm, because tz_where is using very outdated data


    \* System: MacBookPro 2,4GHz i5 (2014) 4GB RAM pytzwhere with numpy active

    \*\*mismatch: pytzwhere finds something and then timezonefinder finds
    something else

    \*\*\*realistic queries: just points within a timezone (= pytzwhere
    yields result)

    \*\*\*\*random queries: random points on earth
