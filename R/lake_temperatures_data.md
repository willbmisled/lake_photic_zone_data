lake\_temperatures\_data
================
bryan
August 1, 2019

Note
----

-   This is a modification of the a file with the same names in: <https://github.com/willbmisled/photic_zone_temp>

Introduction
------------

-   The goal of this project is to compile all of the data necessary to model lake surface temperatures and make predictions for all lakes in NHDplus
-   This file replaces two preliminary attempts:
    -   lake\_temp\_data.Rmd
    -   prism\_big\_data.Rmd
-   All data will be assembled and stored as a SQLite database (lake\_temperatures.sqlite)
    -   database too large to share on github. Must be assembled with the code below.
    -   \[SQLite tables and fields schema and data definitions\] (<https://github.com/willbmisled/photic_zone_temp/blob/master/data/lake_temperatures_sqlite.xlsx>)
-   This document also makes a subset of the data available as a .csv file. This file \[nla\_base.csv\] includes just the data for NLA lakes (2007 & 2012) that will be used to build a random forest model to predict lake temperatures. Data definitions are at the bottom of this document.

Data sources
------------

-   **lmorpho**: [Jeff Hollister's Lake Morphometry database](https://edg.epa.gov/data/PUBLIC/ORD/NHEERL/LakeMorphometry.zip)
-   **nla**
    -   [NLA2007 Lake locations shapefile](https://www.epa.gov/sites/production/files/2017-10/national_lakepoly_withmetrics_20090929.zip)
    -   [NLA2012 Lake locations shapefile](https://www.epa.gov/sites/production/files/2017-10/nla_2012_sites_sampled.zip)
    -   [NLA2007 design file](https://www.epa.gov/sites/production/files/2014-01/nla2007_sampledlakeinformation_20091113.csv%22))
    -   [NLA2012 design file](https://www.epa.gov/sites/production/files/2016-12/nla2012_wide_siteinfo_08232016.csv)
    -   [NLA2007 profile data with lake temperatures by depth](https://www.epa.gov/sites/production/files/2013-09/nla2007_profile_20091008.csv)
    -   [NLA2012 profile data with lake temperatures by depth](https://www.epa.gov/sites/production/files/2016-12/nla2012_wide_profile_08232016.csv)
-   **nlcd**: [2006 & 2011 NLCD percent impervious cover in a 3000m buffer around lake](https://www.mrlc.gov/index.php)
-   **prism**: [prism estimates of daily min & max temperatures for date range (sample\_date - 30 days):(sample\_date) by location](http://www.prism.oregonstate.edu/)

CRS info
--------

-   prism: '+proj=longlat +datum=NAD83 +no\_defs +ellps=GRS80 +towgs84=0,0,0'
-   centroids: '+proj=longlat +datum=NAD83 +no\_defs +ellps=GRS80 +towgs84=0,0,0'
-   lmorhpho: "+proj=merc +a=6378137 +b=6378137 +lat\_ts=0.0 +lon\_0=0.0 +x\_0=0.0 +y\_0=0 +k=1.0 +units=m +<nadgrids=@null> +wktext +no\_defs"
-   nlcd: "+proj=aea +lat\_1=29.5 +lat\_2=45.5 +lat\_0=23 +lon\_0=-96 +x\_0=0 +y\_0=0 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=m +no\_defs"

Data steps
----------

-   Download the tmean prism daily for May1-Sep 30 for years 1981:2018

-   Lake Morphometry
    -   download, convert to a single sf file
    -   save as "lmorpho.rda"
-   add centroids (longitude and latitude) to lmorpho
    -   extract centroids from lmorpho
    -   transform to prism crs (+proj=longlat +datum=NAD83 +no\_defs +ellps=GRS80 +towgs84=0,0,0)
    -   add as longitude and latitude
-   get the prism cellnums (the grid cells) for the centroids

-   write a function to extract the prism data
    -   each year will be separate file saved as data\_local/meanYYYY.rds with YYYY = year
-   Extract the values from the prism rasters and add to the table tmeans\_raw in lake\_temperatures.sqlite

-   calculate 3, 7, and 30 mean temps
    -   means calculated and added to the table tmeans in lake\_temperatures.sqlite
    -   this takes a long time to eval = FALSE; change if you want to rerun
-   get the elevations for the centroids

-   create the lakes table from lmorpho, centroids, and elevation
-   seven COMIDs removed because they were repeated and had slighly different data
-   select and rename fields

-   NLA
    -   download the nla2007 & nla2012 shapefiles
    -   transform to st\_crs(lmorpho)
        -   EPSG: 3857
        -   proj4string: "+proj=merc +a=6378137 +b=6378137 +lat\_ts=0.0 +lon\_0=0.0 +x\_0=0.0 +y\_0=0 +k=1.0 +units=m +<nadgrids=@null> +wktext +no\_defs"
    -   use st\_join to match the 2007 and 2012 lakes to lmorpho
    -   54 nla07 lakes removed due to lack of 1:1 correspondence with lmorpho
    -   7 nla12 lakes removed because they did not map to any lmorpho lakes
    -   82 nla12 lakes removed due to lack of 1:1 correspondence with lmorpho
    -   rbind nla2007 and nla2012
-   get the nla profile data
-   calc the mean temperature for top 2m by nla\_id, date, and visit\_no
-   join to nla\_id to create table nla
-   no profile data for 12 nla07 and 160 nla12 lakes
-   184 lakes sampled twice
-   4 lakes sampled on visit\_no == 2 only

### nlcd impervious data

-   calculate impervious surface in a 3000m buffer for lmorpho / NLA lakes
    -   use year = 2006 for the 2007 lakes
    -   use year = 2011 for the 2012 lakes
-   restrict to lakes in tmean\_2m since these have water temperature data.
-   3k buffer also used in the ecosphere paper: <https://esajournals.onlinelibrary.wiley.com/doi/epdf/10.1002/ecs2.1321>
-   The buffering failed for comid == 14489102 (nla\_id == NLA06608-0243 & nla\_id == NLA12\_SD-101); for now this lake will be removed.
-   goatscape did not return data for 14 of the 2007 lakes and 2 of the 2012 lakes. We could work on this further but it is easiest just to eliminate these.
-   The percent overlap field shows what percent of the buffer is covered by NLCD. For 2007 and 2012 1 and 2 lakes are not completely covered respectively. Eliminate these.
-   For QAQC percent impervious was calculated in ArcMap for a couple of lakes. Results comparable.

-   create table "nla"
    -   primary keys are nla\_id & visit\_no

### QAQC tmeans\_raw: checked and ready to go

-   Use arcMap to extract the prism values for 1 date and compare to table tmean\_raw
    -   open arcMap and add prism raster tmeans for 19920930 ('data\_local/prism\_qaqc.mxd')
    -   save table "lakes" as lakes.csv
    -   import into arcmap and save as lakes.shp
    -   use the extract multi point tools to add prism tmean to lakes.shp (.dbf)
    -   bring back into r and compare
    -   everything peachy:
    -   median difference = 0.0000006
    -   max difference = 2.7 \# checked this out and the lake is on the border of two grid cells
-   no data for 16 cellnums corresponding to 22 lakes
    -   looked at these in arcmap and all are just outside the prism raster; no data

### QAQC tmeans

-   A date chosen randomly
-   Get the raw prism data from table tmean\_raw for the date and the 30 days prior for all cellnums
-   Re-calculate the values in table tmeans from the raw data for all cellnums
-   Compare the results to table tmeans
    -   Done several times - all values match
-   Re-extract the values for the date from the prism raster and compare
    -   Also done several times - all values match
    -   This also verifies the cellnum field in table 'lakes'

### QAQc impervious

-   repeat the impervious buffer clip in arcmap
-   join the nla to lmorpho and transform to nlcd crs
-   write nla lakes to 'data\_local/nla.shp'
-   download 2011 % Dev. Imperv. CONUS: (<https://www.mrlc.gov/data?f%5B0%5D=category%3Aurban%20imperviousness>)
-   add nla.shp and NLCD imperv to arcmap ('data\_local/nlcd\_qaqc.mxd)
-   select nla 2012 lakes for visit\_no == 2
-   buffer with "buffer" tool
    -   Distance = linear unit = 3000m
    -   Side type = Outside Only
    -   End Type = Round
    -   Method = Planar
    -   Dissolve Type = None
    -   save as: nla2012\_visit2\_buf3000
-   use the clip (Data Management) tool to clip 2011 nlcd raster to the buffer
    -   input raster = nlcd imp 2011
    -   output extent = nla2012\_visit2\_buf3000
    -   Check Box: Use input Features for Clipping Geometry = Checked
    -   NoData = 255
    -   save as: nla12\_v1\_buf3000\_imp.tif
-   use Zonal Statistics as Table to sumarize raster clip
    -   input ... zone data = nla2012\_visit2\_buf3000
    -   zone field = 'nla\_id'
    -   nla12\_v1\_buf3000\_imp.tif
    -   output table = nla12\_v1\_buf3000\_imp.dbf
-   compare arcmap and nla.impervious values graphically - near perfect fit. Small differences expected.

### QAQC leftovers

-   The database elevation compared to the nla design file elevations; excellent match
-   tmean\_n recalculated in excel and compared to database; all.equal == TRUE

Document database
-----------------

-   create a table in the database of data definitions (data\_def) from 'data/lake\_temperatures.xlsx'

-   summarize the database

-   summarize the database

<!-- -->

    ## [1] "data_defs"  "lakes"      "nla"        "tmeans"     "tmeans_raw"

-   The database (L:/Public/Milstead\_Lakes/prism/lake\_temperatures.sqlite) is too large (~100 gb) to keep on github. It will have to be compiled by running the code chunks in this document. It could be a lengthy and difficult task.

-   The database includes the 5 tables: "data\_defs", "lakes", "nla", "tmeans", and "tmeans\_raw"

#### Data definitions for table "data\_defs"

-   This table has the data definitions for all tables in \[lake\_temperatures.sqlite\]
-   observations = 46

<table style="width:64%;">
<colgroup>
<col width="16%" />
<col width="47%" />
</colgroup>
<thead>
<tr class="header">
<th>field</th>
<th>description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><strong>table</strong></td>
<td>database table for data definitions</td>
</tr>
<tr class="even">
<td><strong>field</strong></td>
<td>field in database table for data definitions</td>
</tr>
<tr class="odd">
<td><strong>source</strong></td>
<td>source of the data for the field</td>
</tr>
<tr class="even">
<td><strong>units</strong></td>
<td>units of measurement</td>
</tr>
<tr class="odd">
<td><strong>indexed</strong></td>
<td>does a database index exist for the field</td>
</tr>
<tr class="even">
<td><strong>primary</strong></td>
<td>Is this field a primary field? NA = no; primary = simple primary key; composite = primary key composed of 2 or more fields</td>
</tr>
<tr class="odd">
<td><strong>description</strong></td>
<td>a general description of the field</td>
</tr>
</tbody>
</table>

#### Data definitions for table "lakes"

-   This table has the morphometry and elevation for the lmorpho lakes
-   observations = 363,300

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
<col width="10%" />
<col width="6%" />
<col width="6%" />
<col width="44%" />
</colgroup>
<thead>
<tr class="header">
<th>field</th>
<th>source</th>
<th>units</th>
<th>indexed</th>
<th>primary</th>
<th>description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><strong>cellnum</strong></td>
<td>prism</td>
<td>NA</td>
<td>YES</td>
<td></td>
<td>the grid cell number of the prism raster(s) for the lake</td>
</tr>
<tr class="even">
<td><strong>comid</strong></td>
<td>lmorpho</td>
<td>int</td>
<td>YES</td>
<td>primary</td>
<td>nhdplus comid from the lmorpho dataset</td>
</tr>
<tr class="odd">
<td><strong>longitude</strong></td>
<td>lmorpho</td>
<td>dd</td>
<td></td>
<td></td>
<td>longitude of lmorpho centroid. Transformed to prism crs ==&quot;+proj=longlat +datum=NAD83 +no_defs +ellps=GRS80 +towgs84=0,0,0&quot;</td>
</tr>
<tr class="even">
<td><strong>latitude</strong></td>
<td>lmorpho</td>
<td>dd</td>
<td></td>
<td></td>
<td>latitude of lmorpho centroid. Transformed to prism crs ==&quot;+proj=longlat +datum=NAD83 +no_defs +ellps=GRS80 +towgs84=0,0,0&quot;</td>
</tr>
<tr class="odd">
<td><strong>surface_area</strong></td>
<td>lmorpho</td>
<td>m2</td>
<td></td>
<td></td>
<td>lake surface area</td>
</tr>
<tr class="even">
<td><strong>shoreline_length</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>length of lake shoreline</td>
</tr>
<tr class="odd">
<td><strong>shoreline_dev</strong></td>
<td>lmorpho</td>
<td>NA</td>
<td></td>
<td></td>
<td>shoreline development index</td>
</tr>
<tr class="even">
<td><strong>max_length</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>maximum length of the lake polygon</td>
</tr>
<tr class="odd">
<td><strong>max_width</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>maximum width of the lake polygon</td>
</tr>
<tr class="even">
<td><strong>mean_width</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>mean width of the lake polygon</td>
</tr>
<tr class="odd">
<td><strong>max_depth</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>estimated maximum depth of lake</td>
</tr>
<tr class="even">
<td><strong>mean_depth</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>estimated mean depth of lake</td>
</tr>
<tr class="odd">
<td><strong>volume</strong></td>
<td>lmorpho</td>
<td>m3</td>
<td></td>
<td></td>
<td>estimated volume depth of lake</td>
</tr>
<tr class="even">
<td><strong>elevation</strong></td>
<td>aws</td>
<td>m</td>
<td></td>
<td></td>
<td>elevation of the lake</td>
</tr>
</tbody>
</table>

#### Data definitions for table "nla"

-   This table has the NLA water temperature and NLCD impervious data
-   observations = 2370

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
<col width="10%" />
<col width="6%" />
<col width="6%" />
<col width="44%" />
</colgroup>
<thead>
<tr class="header">
<th>field</th>
<th>source</th>
<th>units</th>
<th>indexed</th>
<th>primary</th>
<th>description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><strong>nla_id</strong></td>
<td>nla</td>
<td>char</td>
<td>YES</td>
<td>composite</td>
<td>unique identifiers for the NLA lakes</td>
</tr>
<tr class="even">
<td><strong>comid</strong></td>
<td>lmorpho</td>
<td>int</td>
<td>YES</td>
<td></td>
<td>nhdplus comid from the lmorpho dataset</td>
</tr>
<tr class="odd">
<td><strong>cellnum</strong></td>
<td>prism</td>
<td>NA</td>
<td></td>
<td></td>
<td>the grid cell number of the prism raster(s) for the lake</td>
</tr>
<tr class="even">
<td><strong>year</strong></td>
<td>nla</td>
<td>YYYY</td>
<td></td>
<td></td>
<td>sample year 2007 or 2012</td>
</tr>
<tr class="odd">
<td><strong>date</strong></td>
<td>nla</td>
<td>YYYYMMDD</td>
<td></td>
<td></td>
<td>date the profile temperatures were collected</td>
</tr>
<tr class="even">
<td><strong>visit_no</strong></td>
<td>nla</td>
<td>int</td>
<td></td>
<td>composite</td>
<td>All lakes were sampled once (visit_no = 1); some were sampled twice (visit_no = 2)</td>
</tr>
<tr class="odd">
<td><strong>tmean_2m</strong></td>
<td>nla</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>mean water temperature for depth &lt;= 2 m</td>
</tr>
<tr class="even">
<td><strong>tmean_2m_n</strong></td>
<td>nla</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>number of observations used to calculate tmean_2m</td>
</tr>
<tr class="odd">
<td><strong>percent_impervious</strong></td>
<td>nlcd</td>
<td>percent</td>
<td></td>
<td></td>
<td>percent impervious surface in a 3000m buffer around the lake (includes islands); only observations with 100% coverage kept (i.e., lakes with partial NLCD coverage such as border lakes are excluded)</td>
</tr>
<tr class="even">
<td><strong>area_impervious</strong></td>
<td>nlcd</td>
<td>m2</td>
<td></td>
<td></td>
<td>m2 of impervious surface in a 3000m buffer around the lake (includes islands); only observations with 100% coverage kept (i.e., lakes with partial NLCD coverage such as border lakes are excluded)</td>
</tr>
<tr class="odd">
<td><strong>area_total</strong></td>
<td>lmorpho</td>
<td>m2</td>
<td></td>
<td></td>
<td>area of 3000 m buffer around the lmorpho lake</td>
</tr>
</tbody>
</table>

#### Data definitions for table "tmeans"

-   This table has the aggregated prism air temperature estimates
-   observations = 6.518306310^{8}
-   fields = cellnum, date, tmean, tmean\_dm1, tmean\_avg3, tmean\_avg7, tmean\_avg30

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
<col width="10%" />
<col width="6%" />
<col width="6%" />
<col width="44%" />
</colgroup>
<thead>
<tr class="header">
<th>field</th>
<th>source</th>
<th>units</th>
<th>indexed</th>
<th>primary</th>
<th>description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><strong>cellnum</strong></td>
<td>prism</td>
<td>NA</td>
<td>YES</td>
<td>composite</td>
<td>the grid cell number of the prism raster(s) for the lake</td>
</tr>
<tr class="even">
<td><strong>date</strong></td>
<td>prism</td>
<td>YYYYMMDD</td>
<td>YES</td>
<td>composite</td>
<td>date of the prism estimate</td>
</tr>
<tr class="odd">
<td><strong>tmean</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>mean temperature on the date</td>
</tr>
<tr class="even">
<td><strong>tmean_dm1</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>mean temperature on the day prior to the date (date minus 1)</td>
</tr>
<tr class="odd">
<td><strong>tmean_avg3</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>average of prism means for the 3 days prior to the date</td>
</tr>
<tr class="even">
<td><strong>tmean_avg7</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>average of prism means for the 7 days prior to the date</td>
</tr>
<tr class="odd">
<td><strong>tmean_avg30</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>average of prism means for the 30 days prior to the date</td>
</tr>
</tbody>
</table>

#### Data definitions for table "tmeans\_raw"

-   This table has the raw (by date) prism air temperature estimates
-   observations = 5.34287410^{6}
-   fields = cellnum, d0501, d0502, d0503, d0504, d0505, d0506, d0507, d0508, d0509, d0510, d0511, d0512, d0513, d0514, d0515, d0516, d0517, d0518, d0519, d0520, d0521, d0522, d0523, d0524, d0525, d0526, d0527, d0528, d0529, d0530, d0531, d0601, d0602, d0603, d0604, d0605, d0606, d0607, d0608, d0609, d0610, d0611, d0612, d0613, d0614, d0615, d0616, d0617, d0618, d0619, d0620, d0621, d0622, d0623, d0624, d0625, d0626, d0627, d0628, d0629, d0630, d0701, d0702, d0703, d0704, d0705, d0706, d0707, d0708, d0709, d0710, d0711, d0712, d0713, d0714, d0715, d0716, d0717, d0718, d0719, d0720, d0721, d0722, d0723, d0724, d0725, d0726, d0727, d0728, d0729, d0730, d0731, d0801, d0802, d0803, d0804, d0805, d0806, d0807, d0808, d0809, d0810, d0811, d0812, d0813, d0814, d0815, d0816, d0817, d0818, d0819, d0820, d0821, d0822, d0823, d0824, d0825, d0826, d0827, d0828, d0829, d0830, d0831, d0901, d0902, d0903, d0904, d0905, d0906, d0907, d0908, d0909, d0910, d0911, d0912, d0913, d0914, d0915, d0916, d0917, d0918, d0919, d0920, d0921, d0922, d0923, d0924, d0925, d0926, d0927, d0928, d0929, d0930, year

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
<col width="10%" />
<col width="6%" />
<col width="6%" />
<col width="44%" />
</colgroup>
<thead>
<tr class="header">
<th>field</th>
<th>source</th>
<th>units</th>
<th>indexed</th>
<th>primary</th>
<th>description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><strong>cellnum</strong></td>
<td>prism</td>
<td>NA</td>
<td>YES</td>
<td>composite</td>
<td>the grid cell number of the prism raster(s) for the lake</td>
</tr>
<tr class="even">
<td><strong>d0501:d0531</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>estimated mean temperature for dates May 1:May 31 by tmeans_raw.year</td>
</tr>
<tr class="odd">
<td><strong>d0601:d0630</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>estimated mean temperature for dates June 1:June 30 by tmeans_raw.year</td>
</tr>
<tr class="even">
<td><strong>d0701:d0731</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>estimated mean temperature for dates July 1:July 31 by tmeans_raw.year</td>
</tr>
<tr class="odd">
<td><strong>d0801:d0831</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>estimated mean temperature for dates August 1:August 31 by tmeans_raw.year</td>
</tr>
<tr class="even">
<td><strong>d0901:d0930</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>estimated mean temperature for dates September 1:September 30 by tmeans_raw.year</td>
</tr>
<tr class="odd">
<td><strong>year</strong></td>
<td>prism</td>
<td>YYYY</td>
<td>YES</td>
<td>composite</td>
<td>year 1981 to 2017</td>
</tr>
</tbody>
</table>

Create a subset of the data for lake temperature model and save as nla\_base.Rds
--------------------------------------------------------------------------------

Set up Data
===========

-   load nla data from "data\_local/lake\_temperatures.sqlite"
-   create nla\_base with the 2007 and 2012 nla, prism, lmorpho & imperv data for random forest analysis

### data definitions for nla\_base.Rds & nla\_base.csv

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
<col width="10%" />
<col width="6%" />
<col width="6%" />
<col width="44%" />
</colgroup>
<thead>
<tr class="header">
<th>field</th>
<th>source</th>
<th>units</th>
<th>indexed</th>
<th>primary</th>
<th>description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><strong>nla_id</strong></td>
<td>nla</td>
<td>char</td>
<td>YES</td>
<td>composite</td>
<td>unique identifiers for the NLA lakes</td>
</tr>
<tr class="even">
<td><strong>comid</strong></td>
<td>lmorpho</td>
<td>int</td>
<td>YES</td>
<td></td>
<td>nhdplus comid from the lmorpho dataset</td>
</tr>
<tr class="odd">
<td><strong>cellnum</strong></td>
<td>prism</td>
<td>NA</td>
<td></td>
<td></td>
<td>the grid cell number of the prism raster(s) for the lake</td>
</tr>
<tr class="even">
<td><strong>year</strong></td>
<td>nla</td>
<td>YYYY</td>
<td></td>
<td></td>
<td>sample year 2007 or 2012</td>
</tr>
<tr class="odd">
<td><strong>date</strong></td>
<td>nla</td>
<td>YYYYMMDD</td>
<td></td>
<td></td>
<td>date the profile temperatures were collected</td>
</tr>
<tr class="even">
<td><strong>visit_no</strong></td>
<td>nla</td>
<td>int</td>
<td></td>
<td>composite</td>
<td>All lakes were sampled once (visit_no = 1); some were sampled twice (visit_no = 2)</td>
</tr>
<tr class="odd">
<td><strong>tmean_2m</strong></td>
<td>nla</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>mean water temperature for depth &lt;= 2 m</td>
</tr>
<tr class="even">
<td><strong>percent_impervious</strong></td>
<td>nlcd</td>
<td>percent</td>
<td></td>
<td></td>
<td>percent impervious surface in a 3000m buffer around the lake (includes islands); only observations with 100% coverage kept (i.e., lakes with partial NLCD coverage such as border lakes are excluded)</td>
</tr>
<tr class="odd">
<td><strong>longitude</strong></td>
<td>lmorpho</td>
<td>dd</td>
<td></td>
<td></td>
<td>longitude of lmorpho centroid. Transformed to prism crs ==&quot;+proj=longlat +datum=NAD83 +no_defs +ellps=GRS80 +towgs84=0,0,0&quot;</td>
</tr>
<tr class="even">
<td><strong>latitude</strong></td>
<td>lmorpho</td>
<td>dd</td>
<td></td>
<td></td>
<td>latitude of lmorpho centroid. Transformed to prism crs ==&quot;+proj=longlat +datum=NAD83 +no_defs +ellps=GRS80 +towgs84=0,0,0&quot;</td>
</tr>
<tr class="odd">
<td><strong>surface_area</strong></td>
<td>lmorpho</td>
<td>m2</td>
<td></td>
<td></td>
<td>lake surface area</td>
</tr>
<tr class="even">
<td><strong>shoreline_length</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>length of lake shoreline</td>
</tr>
<tr class="odd">
<td><strong>shoreline_dev</strong></td>
<td>lmorpho</td>
<td>NA</td>
<td></td>
<td></td>
<td>shoreline development index</td>
</tr>
<tr class="even">
<td><strong>max_length</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>maximum length of the lake polygon</td>
</tr>
<tr class="odd">
<td><strong>max_depth</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>estimated maximum depth of lake</td>
</tr>
<tr class="even">
<td><strong>max_width</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>maximum width of the lake polygon</td>
</tr>
<tr class="odd">
<td><strong>mean_width</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>mean width of the lake polygon</td>
</tr>
<tr class="even">
<td><strong>elevation</strong></td>
<td>aws</td>
<td>m</td>
<td></td>
<td></td>
<td>elevation of the lake</td>
</tr>
<tr class="odd">
<td><strong>mean_depth</strong></td>
<td>lmorpho</td>
<td>m</td>
<td></td>
<td></td>
<td>estimated mean depth of lake</td>
</tr>
<tr class="even">
<td><strong>volume</strong></td>
<td>lmorpho</td>
<td>m3</td>
<td></td>
<td></td>
<td>estimated volume depth of lake</td>
</tr>
<tr class="odd">
<td><strong>tmean</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>mean temperature on the date</td>
</tr>
<tr class="even">
<td><strong>tmean_dm1</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>mean temperature on the day prior to the date (date minus 1)</td>
</tr>
<tr class="odd">
<td><strong>tmean_avg3</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>average of prism means for the 3 days prior to the date</td>
</tr>
<tr class="even">
<td><strong>tmean_avg7</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>average of prism means for the 7 days prior to the date</td>
</tr>
<tr class="odd">
<td><strong>tmean_avg30</strong></td>
<td>prism</td>
<td>degrees C</td>
<td></td>
<td></td>
<td>average of prism means for the 30 days prior to the date</td>
</tr>
</tbody>
</table>
