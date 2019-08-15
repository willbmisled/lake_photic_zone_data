predicted\_summer\_lake\_temperatures
================
bryan
August 1, 2019

Notes
-----

-   This is a modification of the a file with the same names in: <https://github.com/willbmisled/photic_zone_temp>

-   This document was knit with all chunks set to "eval = FALSE"; all the chunks run but the last two won't knit. It works.

Introduction
------------

-   A random forest model to predict summer photic zone temperatures in conterminous USA lakes was developed (see "model.Rmd").
-   The model was based on measured photic zone temperatures from the 2007 and 2012 National Lake assessment and a series of predictor variables. Response and predictor variables were combined in a single database ("L:/Public/Milstead\_Lakes/prism/lake\_temperatures.sqlite"). The database is documented in "lake\_temperatures\_data.Rmd".
-   In this document we use the random forest model and the database to predict lake photic zone temperature for all lakes in NHDplus for dates June 1 to September 30 for years 1981 to 2017.
-   The original rmd file for the predictions can be found here: <https://github.com/willbmisled/photic_zone_temp/blob/master/R/data_to_predict_summer_lake_temperatures.Rmd>
-   The predicted summer temperatures were stored in "L:/Public/Milstead\_Lakes/prism/lake\_temperatures\_predicted.sqlite"
-   This document shows the steps used to create the predictions. It takes about a week to run on a dedicated computer.

Data
----

-   "RFAll20190503.Rds" ("L:/Public/Milstead\_Lakes/prism/RFAll20190503.Rds") is the random forest model object used for the predictions
-   All predictor data needed are in: "L:/Public/Milstead\_Lakes/prism/lake\_temperatures.sqlite"
-   Data will be joined from two tables:

### Table "lakes" in lake\_temperatures.sqlite; observations = 363,300

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

### Table "tmeans"in lake\_temperatures.sqlite; observations = 651,830,628

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

-   data predictions are here: "L:/Public/Milstead\_Lakes/prism/lake\_temperatures\_predicted.sqlite"
-   This database has two tables:
    -   **p\_day**:
    -   **p\_means**:

### Table "p\_day" in lake\_temperatures\_predicted.sqlite; observations = 1,639,936,200

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
<td><strong>comid</strong></td>
<td>lmorpho</td>
<td>int</td>
<td>YES</td>
<td>composite</td>
<td>nhdplus comid from the lmorpho dataset</td>
</tr>
<tr class="even">
<td><strong>date</strong></td>
<td>user defined</td>
<td>YYYYMMDD</td>
<td>YES</td>
<td>composite</td>
<td>date of the lake photic zone temperature estimate</td>
</tr>
<tr class="odd">
<td><strong>predicted</strong></td>
<td>predicted</td>
<td>degrees C</td>
<td>NO</td>
<td>NO</td>
<td>predicted lake photic zone temperature</td>
</tr>
</tbody>
</table>

### Table "p\_means" in lake\_temperatures\_predicted.sqlite; observations = 80,652,600

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
<td><strong>comid</strong></td>
<td>lmorpho</td>
<td>int</td>
<td>YES</td>
<td>composite</td>
<td>nhdplus comid from the lmorpho dataset</td>
</tr>
<tr class="even">
<td><strong>year</strong></td>
<td>user defined</td>
<td>YYYY</td>
<td>NO</td>
<td>composite</td>
<td>year for the lake photic zone temperature prediction; values = c(1981 to 2017)</td>
</tr>
<tr class="odd">
<td><strong>mo</strong></td>
<td>user defined</td>
<td>MM</td>
<td>NO</td>
<td>composite</td>
<td>month for the lake photic zone temperature prediction; values = c(NA, 6, 7, 8, 9)</td>
</tr>
<tr class="even">
<td><strong>season</strong></td>
<td>user defined</td>
<td>char</td>
<td>NO</td>
<td>composite</td>
<td>season for the lake photic zone temperature prediction; c(&quot;NA&quot;, &quot;june2aug&quot;, &quot;sostice2equinox&quot;)</td>
</tr>
<tr class="odd">
<td><strong>mean</strong></td>
<td>calculated</td>
<td>degrees C</td>
<td>NO</td>
<td>NA</td>
<td>mean temperature for month or season</td>
</tr>
<tr class="even">
<td><strong>sd</strong></td>
<td>calculated</td>
<td>degrees C</td>
<td>NO</td>
<td>NA</td>
<td>std deviation of temperature for month or season</td>
</tr>
<tr class="odd">
<td><strong>z</strong></td>
<td>calculated</td>
<td>NA</td>
<td>NO</td>
<td>NA</td>
<td>z score of temperature for month or season</td>
</tr>
</tbody>
</table>

Data Steps
----------

-   Write a function to extract the tmeans data by date and / or cellnum

``` r
#' get data from table "tmeans" in "lake_temperatures.sqlite"
#' for most computers you will not have the RAM to return all data so you must specify
#' a date and / or a cellnum.  You can specify just the date or just the cellnum or both.
#' @param dates (optional) date or a vector of dates in format YYYYMMDD.  Dates can be character or numeric.
#'    If no dates are specified all dates will be returned.
#' @param cellnums (optional) cellnum or a vector of cellnums, If no cellnums are specified 
#`    all cellnums will be returned.
#' @examples
#' get_tmeans()
#' get_tmeans(19820911)
#' get_tmeans(c(19820911, 20120911))
#' get_tmeans(cellnum = 26008)
#' get_tmeans(cellnum = c(20389, 26008))
#' get_tmeans(20120911, 20389)
#' get_tmeans(c(19820911, 20120911), c(20389, 26008))

get_tmeans <- function(dates, cellnums){
  db <- dbConnect(RSQLite::SQLite(), db_tmeans)
    if((missing(cellnums)) & (missing(dates))){ 
      print('no date or cellnum provided; cannot return this much data; start over')
    } else {
      if(missing(dates)){
        print('no dates provided; data for all dates will be returned; press ESC to abort')
        return(dbGetQuery(db, paste0('SELECT * FROM tmeans WHERE cellnum IN (',
                          paste(as.character(cellnums), collapse = ", "), ')')))
      } else {
        if(missing(cellnums)){
          print('no cellnums provided; data for all cellnums will be returned; press ESC to abort')
          return(dbGetQuery(db, paste0('SELECT * FROM tmeans WHERE date IN (',
                          paste(as.character(dates), collapse = ", "), ')')))
        } else {
            return(dbGetQuery(db, paste0('SELECT * FROM tmeans WHERE 
                           date IN (', paste(as.character(dates), collapse=", "), ') AND 
                           cellnum in (', paste(as.character(cellnums), collapse=", "), ')')))
        }
      }}
  dbDisconnect(db)
}
```

-   Write a function to extract the lakes data by comid and / or cellnum

``` r
#' get data from table "lakes" in "lake_temperatures.sqlite"
#' The entire table can be downloaded or you can choose a set of lakes based on comid and/or cellnum.
#' @param comid (optional) comid or a vector of lmorpho(NHDplus) comids.
#' @param cellnums (optional) cellnum or a vector of cellnums.
#' @examples
#' get_lakes()
#' get_lakes(120050269)
#' get_lakes(c(120050269, 120050594))
#' get_lakes(cellnum = 26008)
#' get_lakes(cellnum = c(20389, 26008))
#' get_lakes(120050269, 20389)
#' get_lakes(c(120050269, 120050594), c(20389, 26008))

get_lakes <- function(comids, cellnums){
  db <- dbConnect(RSQLite::SQLite(), db_tmeans)
    if((missing(cellnums)) & (missing(comids))){ 
      print('no comid or cellnum provided; data for all lakes will be returned; press ESC to abort')
      return(dbGetQuery(db, "SELECT * FROM lakes"))
    } else {
      if(missing(comids)){
        print('no comids provided; data for selected cellnums will be returned; press ESC to abort')
        return(dbGetQuery(db, paste0('SELECT * FROM lakes WHERE cellnum IN (',
                          paste(as.character(cellnums), collapse = ", "), ')')))
      } else {
        if(missing(cellnums)){
          print('no cellnums provided; data for all selected comids will be returned; press ESC to abort')
          return(dbGetQuery(db, paste0('SELECT * FROM lakes WHERE comid IN (',
                          paste(as.character(comids), collapse = ", "), ')')))
        } else {
            return(dbGetQuery(db, paste0('SELECT * FROM lakes WHERE 
                           comid IN (', paste(as.character(comids), collapse=", "), ') OR 
                           cellnum in (', paste(as.character(cellnums), collapse=", "), ')')))
        }
      }}
  dbDisconnect(db)
}
```

-   write function to predict daily lake photic zone temperatures and monthly & seasonal means & z scopes for lakes from table "lakes" in "lake\_temperatures.sqlite"
    -   join to tmeans data (table "tmeans" in "lake\_temperatures.sqlite") for dates
    -   get predictions from rf model from: "RFAll20190503.Rds"
    -   calculate means and zscores:
    -   by comid and mo
    -   by calendar season (June 1 - Aug. 31)
    -   by Astronomical season (June 21 to Sep. 22)
    -   output list with data.frames for "predictions" and "means"

``` r
#' get predicted lake photo zone temperatures for a subset of data from 
#' table "lakes" in "lake_temperatures.sqlite"
#' User must supply the lakes data.frame and the dates for the predictions
#' @param lakes1 A dataframe with the lakes data from table "lakes" in "lake_temperatures.sqlite". 
#' @param dates A vector of dates in format YYYYMMDD that you wish to have
#' predictions for.
#' @param rf The random forest model to use for the predictions 
#' @examples
#' lakes1 <- get_lakes(120050269)
#' rf <- readRDS(rf_location) 
#' dates <- c(19810704, 20150821)
#' get_predictions(lakes1, dates, rf)

get_predictions <- function(lakes1, dates, rf){
  
# helper function to calculate z scores
  scale_this <- function(x) as.vector(scale(x))

# get a list of cellnums to extract the tmeans data for
  cnum <- unique(lakes1$cellnum)

# join lakes1 to tmeans
  lakes1 <- left_join(lakes1, get_tmeans(dates, cnum)) # 451400      20

# redo the dates; for random forest need to convert date to day of year
    # date is the day of year
    # cdate is the calendar date yyyymmdd
  lakes1$cdate <- lakes1$date
  lakes1$year = year(ymd(lakes1$date))
  lakes1$mo = month(ymd(lakes1$date))
  lakes1$date <- yday(lubridate::ymd(lakes1$date)) # convert date to day of year
 
# predict photic zone temp
  lakes1$predicted <- predict(rf, newdata = lakes1) # 451400     24

# get means, sd, & z-score for each comid by year & mo
  sum<- group_by(lakes1, comid, year, mo) %>% 
    summarise(mean = mean(predicted, na.rm = TRUE), sd = sd(predicted, na.rm = TRUE))
  
# scale predicted mean; get z scores by comid & mo
  z <- group_by(sum, comid, mo) %>% 
  mutate(z = scale_this(mean))   # 14800     6

# get means & sd for each comid by year for summer = June1 to Aug. 31
  summer1 <- filter(lakes1, mo < 9)
  summer1$season = "june2aug" 
  
  sum1<- group_by(summer1, comid, year, season) %>% 
    summarise(mean = mean(predicted, na.rm = TRUE), sd = sd(predicted, na.rm = TRUE)) 
# scale predicted mean; get z scores by comid & season
  z1 <- group_by(sum1, comid, season) %>% 
      mutate(z = scale_this(mean))  # 3700    6
  
# get means & sd for each comid by year for summer = solistice to equinox
  # for summer solstice ~ June 21 
  # for fall equinox ~ sep 23 
  summer2 <- filter(lakes1, 
                    as.numeric(substr(cdate, 6, 8)) > 620 &
                    as.numeric(substr(cdate, 6, 8)) < 923)
  summer2$season = "sostice2equinox"
  
  sum2<- group_by(summer2, comid, year, season) %>% 
    summarise(mean = mean(predicted, na.rm = TRUE), sd = sd(predicted, na.rm = TRUE))
  
# scale predicted mean; get z scores by comid & season
  z2 <- group_by(sum2, comid, season) %>% 
    mutate(z = scale_this(mean))  # 3700    6
  
# put all the means together
  means <- bind_rows(z, z1, z2) %>%
    select(comid, year, mo, season, mean, sd, z) %>%
    arrange(comid, mo, season, year) # 22200     7

# reformat lakes1$date to calendar date
  predictions <- select(lakes1, comid, date = cdate, predicted)  # 451400      3

# output data
  return(list(predictions, means))
}
```

-   load rf model model "RFAll20190503.Rds"
-   get data for all lakes in table "lakes" in "lake\_temperatures.sqlite"
-   create vector of dates for June 1 to Sep. 30 for years 1981 to 2017

``` r
# load rf
  rf <- readRDS(rf_location) 

# get the lakes data
lakes <- get_lakes() %>%
  arrange(cellnum) # 363300     14
```

    ## [1] "no comid or cellnum provided; data for all lakes will be returned; press ESC to abort"

``` r
# all dates for June 1 to Sep. 30 for years 1981 to 2017
dates <- c()
for(y in 1981:2017){
  dates <- c(dates,as.numeric(format(seq(ymd(paste0(y, '06', '01')),
                                         ymd(paste0(y, '06', '30')), by = '1 day'), "%Y%m%d")))
  dates <- c(dates,as.numeric(format(seq(ymd(paste0(y, '07', '01')),
                                         ymd(paste0(y, '07', '31')), by = '1 day'), "%Y%m%d")))
  dates <- c(dates,as.numeric(format(seq(ymd(paste0(y, '08', '01')),
                                         ymd(paste0(y, '08', '31')), by = '1 day'), "%Y%m%d")))  
  dates <- c(dates,as.numeric(format(seq(ymd(paste0(y, '09', '01')),
                                         ymd(paste0(y, '09', '30')), by = '1 day'), "%Y%m%d")))
}
```

-   use loop to predict daily lake photic zone temperatures and monthly & seasonal means & z scopes for lakes

``` r
# loop to predict daily lake photic zone temperatures and monthly & seasonal means & z scopes for lakes

check <- c() # blank object to hold i values with problems

for(i in 1:3633) {
  # for(i in 1:1) {
  begin <- Sys.time()
  print(paste0('starting group ', i, ' of 3633', "; ", begin))

  # subset the lakes data
    start <- (i * 100) - 99
    end <- (i * 100)
    lakes1 <- slice(lakes, start:end)

  # get the predictions
    a <- get_predictions(lakes1, dates, rf)
    
  # save the predicted temps for day and monthly means to lake_temperatures_predicted.sqlite
    db <- dbConnect(RSQLite::SQLite(), db_preds)
    x <- tryCatch(dbWriteTable(db, "p_day", a[[1]], 
                               append = TRUE), error = function(e) FALSE)
      if(x == TRUE) print(
        paste0('predictions for group ', i, ' written to p_day'))
      if(x == FALSE) {
        print(paste0('OJO predictions for group ', i, ' NOT written to p_day OJO'))
        check <- c(check, paste0('p_day_', i))
      }
    
    y <- tryCatch(dbWriteTable(db, "p_means", a[[2]], 
                               append = TRUE),error = function(e) FALSE)
      if(y == TRUE) print(
        paste0('predictions for group ', i, ' written to p_means'))
      if(y == FALSE) {
        print(paste0('OJO predictions for group ', i, ' NOT written to p_means OJO'))
        check <- c(check, paste0('p_means_', i))
      }
    
    print(paste0('group ', i, ' completed; ', Sys.time()))
    print(Sys.time() - begin)
    print('***********')
    dbDisconnect(db)
    saveRDS(i, here::here('data_local/i.Rds'))
}  
```

-   check database and add indices

``` r
db <- dbConnect(RSQLite::SQLite(), db_preds)
dbListTables(db) # "p_day"   "p_means"
dbListFields(db, "p_day") # "comid"     "date"      "predicted"
# dbGetQuery(db, 'SELECT Count(*) FROM p_day') # 1639936200
# dbExecute(db,"CREATE INDEX index_p_day_comid ON p_day (comid)")
dbExecute(db,"CREATE INDEX index_p_day_date ON p_day (date)")
## not run


dbListFields(db, "p_means") # "comid"  "year"   "mo"     "season" "mean"   "sd"     "z" 
dbGetQuery(db, 'SELECT Count(*) FROM p_means') #
dbExecute(db,"CREATE INDEX index_p_means_comid ON p_means (comid)")

dbDisconnect(db)
```

How to use the predicted data
-----------------------------

-   Both sqlite database are fully functional relational databases. This means that you can join tables with common keys (e.g. comid), create new variables,perform summary calculations (e.g. mean, sd, etc.), and other database actions. However the files are large and operations can take a long time to return the data. Some of the tables have millions of observations so downloading an entire table is probably not possible unless you have a bunch of ram. What seems to work best is to use sql language to download parts of large tables and then work do the joins and other operations directly in R. Below are some examples.

-   examine the structure of a database
    -   connect to a database
    -   list the tables
    -   list the fields in a table
    -   count the records in table

``` r
# connect to the database (db_tmeans and db_preds are the locations of the databases specified at the top of the document)
  dbm <- dbConnect(RSQLite::SQLite(), db_tmeans)
  dbListTables(dbm) # "data_defs"  "lakes"  "nla"   "tmeans" "tmeans_raw"
  dbListFields(dbm, "lakes") # "cellnum"  "comid"  "longitude"  "latitude"  etc.
  dbGetQuery(dbm, 'SELECT Count(*) FROM lakes') # 363300
  dbDisconnect(dbm) # disconnect from database when done
```

-   download all records and all fields from a table as a data.frame

``` r
# connect to the database (db_tmeans and db_preds are the locations of the databases specified at the top of the document)
  dbm <- dbConnect(RSQLite::SQLite(), db_tmeans)
  lakes <- dbGetQuery(dbm, "SELECT * FROM lakes") # 363300     14
  dbDisconnect(dbm) # disconnect from database when done
```

-   download all records and *selected* fields from a table as a data.frame

``` r
# connect to the database (db_tmeans and db_preds are the locations of the databases specified at the top of the document)
  dbm <- dbConnect(RSQLite::SQLite(), db_tmeans)
  dbListFields(dbm, "lakes") # "cellnum"  "comid"  "longitude"  "latitude"  etc.
  lakes <- dbGetQuery(dbm, "SELECT lakes.comid, lakes.latitude, lakes.longitude 
                      FROM lakes") # 363300     3
  dbDisconnect(dbm) # disconnect from database when done
```

-   download *selected* records and *selected* fields from a table as a data.frame
    -   for example you can download a lake with a specific comid
    -   or a group of lakes from a list of comids
-   if you want to get fancy you can figure out more complicated queries using:
    -   "and" and "or"
    -   or even \[paste\] to generate the values for the SELECT and WHERE commands in the query. Smart people like you will figure this out if you need it.

``` r
# connect to the database (db_tmeans and db_preds are the locations of the databases specified at the top of the document)
  dbm <- dbConnect(RSQLite::SQLite(), db_tmeans)

# a specific comid
  lakes <- dbGetQuery(dbm, " 
              SELECT lakes.comid, lakes.latitude, lakes.longitude FROM lakes
              WHERE comid = 3095201") # 1     3
  
# a list of comids
  lakes <- dbGetQuery(dbm, " 
              SELECT lakes.comid, lakes.latitude, lakes.longitude FROM lakes
              WHERE comid IN (3095201, 16009504, 12899164, 1614198)") # 4     3
 
  
  dbDisconnect(dbm) # disconnect from database when done
```

-   As an example let's download data and map it.
    -   get the season = 'sostice2equinox' means for all lakes and years
    -   WOW, 13M records
    -   get the lake locations from table "lakes"
    -   join the two datasets
    -   This is just too big so we need to select fewer records
    -   We can do this as an SQL query (see above) or just do it in R (more flexible)
-   Use tmap to map one year of a subset of the lakes

``` r
# https://geocompr.robinlovelace.net/adv-map.html
# https://cran.r-project.org/web/packages/tmap/vignettes/tmap-getstarted.html

# get the 'sostice2equinox' means for all years
  dbp <- dbConnect(RSQLite::SQLite(), db_preds)
  dbListTables(dbp) # "p_day"   "p_means"
  dbListFields(dbp, "p_means") # "comid"  "year"   "mo"     "season" "mean"   "sd"     "z"  
  s2e <- dbGetQuery(dbp, "SELECT * FROM p_means 
                          WHERE season = 'sostice2equinox'") # 13442100        7
  dbDisconnect(dbp)

# lakes  
  dbm <- dbConnect(RSQLite::SQLite(), db_tmeans)
  lakes <- dbGetQuery(dbm, "SELECT lakes.comid, lakes.latitude, lakes.longitude 
                      FROM lakes") # 363300     3
  dbDisconnect(dbm) # disconnect from database when done

# join the lakes and predicted means as an sf object
  s1 <- inner_join(lakes, s2e) %>%
      st_as_sf(coords = c('longitude', 'latitude'), crs ="+proj=longlat +datum=NAD83 +no_defs +ellps=GRS80 +towgs84=0,0,0") # 13442100        8
  
# map the data - but it is way too big.  So map a subset
  # how many lakes?
    nn <- 3000
    
  # Which year?
    yr <- 2013
    
  # subset the data
    nnn <- sample(lakes$comid, nn) # select comids to map
        s1n <- filter(s1, comid %in% nnn & year == yr)
  
  # map it
    tmap_mode("view")  
    tm_shape(s1n) +
      tm_symbols(col = "z", midpoint = 0, palette = "-Spectral", n = 8, border.alpha = 0, size = 1.0) +
    tm_basemap("Stamen.Watercolor")
```

-   Just as a suggestion of of a way to look at the data here is a ridge plot
-   Plotted are all of the z values by year for season = 'sostice2equinox'
-   pretty but nothing seems to jump out. If you squint maybe a slight increase in temps.
-   you may want to figure out a way to look at this regionally or perhaps cluster the lakes (straight up or fancy, i.e., NMDS) and then look at how groups of lakes change over time.

``` r
# https://cran.r-project.org/web/packages/ggridges/vignettes/introduction.html

ggplot(s1, aes(x = z, y = as.factor(year))) + geom_density_ridges()
```
