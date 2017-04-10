---
title: Functions

menu:
  influxdb_1_2:
    weight: 60
    parent: query_language
---

Aggregate, select, transform, and predict data with InfluxQL functions.

#### Content

* [Aggregations](#aggregations)
    * [COUNT()](#count)
    * [DISTINCT()](#distinct)
    * [INTEGRAL()](#integral)
    * [MEAN()](#mean)
    * [MEDIAN()](#median)
    * [MODE()](#mode)
    * [SPREAD()](#spread)
    * [STDDEV()](#stddev)
    * [SUM()](#sum)
    * [Common Issues with Aggregations](#common-issues-with-aggregations)
* [Selectors](#selectors)
    * [BOTTOM()](#bottom)        
    * [FIRST()](#first)          
    * [LAST()](#last)            
    * [MAX()](#max)              
    * [MIN()](#min)              
    * [PERCENTILE()](#percentile)
    * [SAMPLE()](#sample)        
    * [TOP()](#top)
    * [Common Issues with Selectors](#common-issues-with-selectors)          
* [Transformations](#Transformations)
    * [CEILING()](#ceiling)  
    * [CUMULATIVE_SUM()](#cumulative-sum)                    
    * [DERIVATIVE()](#derivative)                          
    * [DIFFERENCE()](#difference)                          
    * [ELAPSED()](#elapsed)                                
    * [FLOOR()](#floor)                                    
    * [HISTOGRAM()](#histogram)                            
    * [MOVING_AVERAGE()](#moving-average)                  
    * [NON_NEGATIVE_DERIVATIVE()](#non-negative-derivative)
    * [Common Issues with Transformations](#common-issues-with-transformations)    
* [Predictors](#predictors)
    * [HOLT_WINTERS()](#holt-winters)
* [Other](#other)
    * [Sample Data](#sample-data)
    * [General Syntax for Functions](#general-syntax-for-functions)
        * [Multiple function specification](#multiple-function-specification)
        * [`fill()`](#fill)
        * [`AS`](#as)

# Aggregations

## COUNT()
Returns the number of non-null [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax

```
SELECT COUNT( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

#### Nested Syntax
```
SELECT COUNT(DISTINCT( [ * | <field_key> | /<regular_expression>/ ] )) [...]
```

### Description of Syntax

`COUNT(field_key)`  
&emsp;&emsp;&emsp;
Returns the number of field values associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`COUNT(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the number of field values associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`COUNT(*)`  
&emsp;&emsp;&emsp;
Returns the number of field values associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`COUNT()` supports all field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).
InfluxQL supports nesting [`DISTINCT()`](#distinct) with `COUNT()`.

### Examples

#### Example 1: Count the field values associated with a field key
```
> SELECT COUNT("water_level") FROM "h2o_feet"

name: h2o_feet
time                   count
----                   -----
1970-01-01T00:00:00Z   15258
```
The query returns the number of non-null field values in the `water_level` field key in the `h2o_feet` measurement.

#### Example 2: Count the field values associated with each field key in a measurement
```
> SELECT COUNT(*) FROM "h2o_feet"

name: h2o_feet
time                   count_level description   count_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   15258                     15258
```
The query returns the number of non-null field values for each field key associated with the `h2o_feet` measurement.
The `h2o_feet` measurement has two field keys: `level description` and `water_level`.

#### Example 3: Count the field values associated with each field key that matches a regular expression
```
> SELECT COUNT(/water/) FROM "h2o_feet"

name: h2o_feet
time                   count_water_level
----                   -----------------
1970-01-01T00:00:00Z   15258
```
The query returns the number of non-null field values for every field key that contains the word `water` in the `h2o_feet` measurement.

#### Example 4: Count the field values associated with a field key and include several clauses
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(200) LIMIT 7 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   count
----                   -----
2015-08-17T23:48:00Z   200
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2
2015-08-18T00:36:00Z   2
2015-08-18T00:48:00Z   2
```
The query returns the number of non-null field values in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `200` and [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to seven and one.

#### Example 5: Count the distinct field values associated with a field key
```
> SELECT COUNT(DISTINCT("level description")) FROM "h2o_feet"

name: h2o_feet
time                   count
----                   -----
1970-01-01T00:00:00Z   4
```

The query returns the number of unique field values for the `level description` field key and the `h2o_feet` measurement.

### Common Issues with COUNT()

#### Issue 1: COUNT() and fill()
Most InfluxQL functions report `null` values for time intervals with no data, and
[`fill(<fill_option>)`](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill)
replaces that `null` value with the `fill_option`.
`COUNT()` reports `0` for time intervals with no data, and `fill(<fill_option>)` replaces any `0` values with the `fill_option`.

##### Example
<br>
The first query in the codeblock below does not include `fill()`.
The last time interval has no data so the reported value for that time interval is zero.
The second query includes `fill(800000)`; it replaces the zero in the last interval with `800000`.

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-09-18T21:24:00Z' AND time <= '2015-09-18T21:54:00Z' GROUP BY time(12m)

name: h2o_feet
time                   count
----                   -----
2015-09-18T21:24:00Z   2
2015-09-18T21:36:00Z   2
2015-09-18T21:48:00Z   0

> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-09-18T21:24:00Z' AND time <= '2015-09-18T21:54:00Z' GROUP BY time(12m) fill(800000)

name: h2o_feet
time                   count
----                   -----
2015-09-18T21:24:00Z   2
2015-09-18T21:36:00Z   2
2015-09-18T21:48:00Z   800000
```

## DISTINCT()
Returns the list of unique [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT DISTINCT( [ * | <field_key> | /<regular_expression>/ ] ) FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

#### Nested Syntax
```
SELECT COUNT(DISTINCT( [ * | <field_key> | /<regular_expression>/ ] )) [...]
```

### Description of Syntax

`DISTINCT(field_key)`  
&emsp;&emsp;&emsp;
Returns the unique field values associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`DISTINCT(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the unique field values associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`DISTINCT(*)`  
&emsp;&emsp;&emsp;
Returns the unique field values associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`DISTINCT()` supports all field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).
InfluxQL supports nesting `DISTINCT()` with [`COUNT()`](#count).

### Examples

#### Example 1: List the distinct field values associated with a field key
```
> SELECT DISTINCT("level description") FROM "h2o_feet"

name: h2o_feet
time                   distinct
----                   --------
1970-01-01T00:00:00Z   between 6 and 9 feet
1970-01-01T00:00:00Z   below 3 feet
1970-01-01T00:00:00Z   between 3 and 6 feet
1970-01-01T00:00:00Z   at or greater than 9 feet
```
The query returns a tabular list of the unique field values in the `level description` field key in the `h2o_feet` measurement.

#### Example 2: List the distinct field values associated with each field key in a measurement
```
> SELECT DISTINCT(*) FROM "h2o_feet"

name: h2o_feet
time                   distinct_level description   distinct_water_level
----                   --------------------------   --------------------
1970-01-01T00:00:00Z   between 6 and 9 feet         8.12
1970-01-01T00:00:00Z   between 3 and 6 feet         8.005
1970-01-01T00:00:00Z   at or greater than 9 feet    7.887
1970-01-01T00:00:00Z   below 3 feet                 7.762
[...]
```
The query returns a tabular list of the unique field values for each field key in the `h2o_feet` measurement.
The `h2o_feet` measurement has two field keys: `level description` and `water_level`.

#### Example 3: List the distinct field values associated with each field key that matches a regular expression
```
> SELECT DISTINCT(/description/) FROM "h2o_feet"

name: h2o_feet
time                   distinct_level description
----                   --------------------------
1970-01-01T00:00:00Z   below 3 feet
1970-01-01T00:00:00Z   between 6 and 9 feet
1970-01-01T00:00:00Z   between 3 and 6 feet
1970-01-01T00:00:00Z   at or greater than 9 feet
```
The query returns a tabular list of the unique field values for each field key in the `h2o_feet` measurement that contains the word `description`.

#### Example 4: List the distinct field values associated with a field key and include several clauses
```
>  SELECT DISTINCT("level description") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   distinct
----                   --------
2015-08-18T00:00:00Z   between 6 and 9 feet
2015-08-18T00:12:00Z   between 6 and 9 feet
2015-08-18T00:24:00Z   between 6 and 9 feet
2015-08-18T00:36:00Z   between 6 and 9 feet
2015-08-18T00:48:00Z   between 6 and 9 feet
```
The query returns a tabular list of the unique field values in the `level description` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query also [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of series returned to one.

#### Example 5: Count the distinct field values associated with a field key
```
> SELECT COUNT(DISTINCT("level description")) FROM "h2o_feet"

name: h2o_feet
time                   count
----                   -----
1970-01-01T00:00:00Z   4
```

The query returns the number of unique field values in the `level description` field key and the `h2o_feet` measurement.

### Common Issues with DISTINCT()

#### Issue 1: DISTINCT() and the INTO clause

Using `DISTINCT()` with the [`INTO` clause](/influxdb/v1.2/query_language/data_exploration/#the-into-clause) can cause InfluxDB to overwrite points in the destination measurement.
`DISTINCT()` often returns several results with the same timestamp; InfluxDB assumes [points](/influxdb/v1.2/concepts/glossary/#point) with the same [series](/influxdb/v1.2/concepts/glossary/#series) and timestamp are duplicate points and simply overwrites any duplicate point with the most recent point in the destination measurement.

##### Example
<br>
The first query in the codeblock below uses the `DISTINCT()` function and returns four results.
Notice that each result has the same timestamp.
The second query adds an `INTO` clause to the initial query and writes the query results to the `distincts` measurement.
The last query in the codeblock selects all the data in the `distincts` measurement.

The last query returns one point because the four initial results are duplicate points; they belong to the same series and have the same timestamp.
When the system encounters duplicate points, it simply overwrites the previous point with the most recent point.

```
>  SELECT DISTINCT("level description") FROM "h2o_feet"

name: h2o_feet
time                   distinct
----                   --------
1970-01-01T00:00:00Z   below 3 feet
1970-01-01T00:00:00Z   between 6 and 9 feet
1970-01-01T00:00:00Z   between 3 and 6 feet
1970-01-01T00:00:00Z   at or greater than 9 feet

>  SELECT DISTINCT("level description") INTO "distincts" FROM "h2o_feet"

name: result
time                   written
----                   -------
1970-01-01T00:00:00Z   4

> SELECT * FROM "distincts"

name: distincts
time                   distinct
----                   --------
1970-01-01T00:00:00Z   at or greater than 9 feet
```

## INTEGRAL()
`INTEGRAL()` is not yet functional.

<dt> See GitHub Issue [#5930](https://github.com/influxdata/influxdb/issues/5930) for more information.
</dt>

## MEAN()
Returns the arithmetic mean (average) of [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT MEAN( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`MEAN(field_key)`  
&emsp;&emsp;&emsp;
Returns the average field value associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`MEAN(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the average field value associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`MEAN(*)`  
&emsp;&emsp;&emsp;
Returns the average field value associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`MEAN()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples

#### Example 1: Calculate the mean field value associated with a field key
```
> SELECT MEAN("water_level") FROM "h2o_feet"

name: h2o_feet
time                   mean
----                   ----
1970-01-01T00:00:00Z   4.442107025822522
```
The query returns the average field value in the `water_level` field key in the `h2o_feet` measurement.

#### Example 2: Calculate the mean field value associated with each field key in a measurement
```
> SELECT MEAN(*) FROM "h2o_feet"

name: h2o_feet
time                   mean_water_level
----                   ----------------
1970-01-01T00:00:00Z   4.442107025822522
```
The query returns the average field value for every field key that stores numerical values in the `h2o_feet` measurement.
The `h2o_feet` measurement has one numerical field: `water_level`.

#### Example 3: Calculate the mean field value associated with each field key that matches a regular expression
```
> SELECT MEAN(/water/) FROM "h2o_feet"

name: h2o_feet
time                   mean_water_level
----                   ----------------
1970-01-01T00:00:00Z   4.442107025822523
```
The query returns the average field value for each field key that stores numerical values and includes the word `water` in the `h2o_feet` measurement.

#### Example 4: Calculate the mean field value associated with a field key and include several clauses
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 7 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   mean
----                   ----
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.0625
2015-08-18T00:12:00Z   7.8245
2015-08-18T00:24:00Z   7.5675
2015-08-18T00:36:00Z   7.303
2015-08-18T00:48:00Z   7.11
```
The query returns the average of the values in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `9.01` and [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to seven and one.

## MEDIAN()
Returns the middle value from a sorted list of [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT MEDIAN( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`MEDIAN(field_key)`  
&emsp;&emsp;&emsp;
Returns the middle field value associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`MEDIAN(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the middle field value associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`MEDIAN(*)`  
&emsp;&emsp;&emsp;
Returns the middle field value associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`MEDIAN()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

> **Note:** `MEDIAN()` is nearly equivalent to [`PERCENTILE(field_key, 50)`](#percentile), except `MEDIAN()` returns the average of the two middle field values if the field contains an even number of values.

### Examples

#### Example 1: Calculate the median field value associated with a field key
```
> SELECT MEDIAN("water_level") FROM "h2o_feet"

name: h2o_feet
time                   median
----                   ------
1970-01-01T00:00:00Z   4.124
```
The query returns the middle field value in the `water_level` field key and in the `h2o_feet` measurement.

#### Example 2: Calculate the median field value associated with each field key in a measurement
```
> SELECT MEDIAN(*) FROM "h2o_feet"

name: h2o_feet
time                   median_water_level
----                   ------------------
1970-01-01T00:00:00Z   4.124
```
The query returns the middle field value for every field key that stores numerical values in the `h2o_feet` measurement.
The `h2o_feet` measurement has one numerical field: `water_level`.

#### Example 3: Calculate the median field value associated with each field key that matches a regular expression
```
> SELECT MEDIAN(/water/) FROM "h2o_feet"

name: h2o_feet
time                   median_water_level
----                   ------------------
1970-01-01T00:00:00Z   4.124
```
The query returns the middle field value for every field key that stores numerical values and includes the word `water` in the `h2o_feet` measurement.

#### Example 4: Calculate the median field value associated with a field key and include several clauses
```
> SELECT MEDIAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(700) LIMIT 7 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   median
----                   ------
2015-08-17T23:48:00Z   700
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   2.077
2015-08-18T00:24:00Z   2.0460000000000003
2015-08-18T00:36:00Z   2.0620000000000003
2015-08-18T00:48:00Z   700
```
The query returns the middle field value in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `700 `, [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to seven and one, and [offsets](/influxdb/v1.2/query_language/data_exploration/#the-offset-and-soffset-clauses) the series returned by one.

## MODE()
Returns the most frequent value in a list of [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT MODE( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`MODE(field_key)`  
&emsp;&emsp;&emsp;
Returns the most frequent field value associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`MODE(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the most frequent field value associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`MODE(*)`  
&emsp;&emsp;&emsp;
Returns the most frequent field value associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`MODE()` supports all field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

> **Note:** `MODE()` returns the field value with the earliest [timestamp](/influxdb/v1.2/concepts/glossary/#timestamp) if  there's a tie between two or more values for the maximum number of occurrences.

### Examples

#### Example 1: Calculate the mode field value associated with a field key
```
> SELECT MODE("level description") FROM "h2o_feet"

name: h2o_feet
time                   mode
----                   ----
1970-01-01T00:00:00Z   between 3 and 6 feet
```
The query returns the most frequent field value in the `level description` field key and in the `h2o_feet` measurement.

#### Example 2: Calculate the mode field value associated with each field key in a measurement
```
> SELECT MODE(*) FROM "h2o_feet"

name: h2o_feet
time                   mode_level description   mode_water_level
----                   ----------------------   ----------------
1970-01-01T00:00:00Z   between 3 and 6 feet     2.69
```
The query returns the most frequent field value for every field key in the `h2o_feet` measurement.
The `h2o_feet` measurement has two fields: `level description` and `water_level`.

#### Example 3: Calculate the mode field value associated with each field key that matches a regular expression
```
> SELECT MODE(/water/) FROM "h2o_feet"

name: h2o_feet
time                   mode_water_level
----                   ----------------
1970-01-01T00:00:00Z   2.69
```
The query returns the most frequent field value for every field key that includes the word `/water/` in the `h2o_feet` measurement.

#### Example 4: Calculate the mode field value associated with a field key and include several clauses
```
> SELECT MODE("level description") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* LIMIT 3 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   mode
----                   ----
2015-08-17T23:48:00Z
2015-08-18T00:00:00Z   below 3 feet
2015-08-18T00:12:00Z   below 3 feet
```
The query returns the mode of the values associated with the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to three and one, and it [offsets](/influxdb/v1.2/query_language/data_exploration/#the-offset-and-soffset-clauses) the series returned by one.

## SPREAD()
Returns the difference between the minimum and maximum [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT SPREAD( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`SPREAD(field_key)`  
&emsp;&emsp;&emsp;
Returns the difference between the minimum and maximum field values associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`SPREAD(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the difference between the minimum and maximum field values associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`SPREAD(*)`  
&emsp;&emsp;&emsp;
Returns the difference between the minimum and maximum field values associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`SPREAD()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples

#### Example 1: Calculate the spread for the field values associated with a field key
```
> SELECT SPREAD("water_level") FROM "h2o_feet"

name: h2o_feet
time                   spread
----                   ------
1970-01-01T00:00:00Z   10.574
```

The query returns the difference between the minimum and maximum field values in the `water_level` field key and in the `h2o_feet` measurement.

#### Example 2: Calculate the spread for the field values associated with each field key in a measurement
```
> SELECT SPREAD(*) FROM "h2o_feet"

name: h2o_feet
time                   spread_water_level
----                   ------------------
1970-01-01T00:00:00Z   10.574
```

The query returns the difference between the minimum and maximum field values for every field key that stores numerical values in the `h2o_feet` measurement.
The `h2o_feet` measurement has one numerical field: `water_level`.

#### Example 3: Calculate the spread for the field values associated with each field key that matches a regular expression
```
> SELECT SPREAD(/water/) FROM "h2o_feet"

name: h2o_feet
time                   spread_water_level
----                   ------------------
1970-01-01T00:00:00Z   10.574
```

The query returns the difference between the minimum and maximum field values for every field key that stores numerical values and includes the word `water` in the `h2o_feet` measurement.

#### Example 4: Calculate the spread for the field values associated with a field key and include several clauses
```
> SELECT SPREAD("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(18) LIMIT 3 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   spread
----                   ------
2015-08-17T23:48:00Z   18
2015-08-18T00:00:00Z   0.052000000000000046
2015-08-18T00:12:00Z   0.09799999999999986
```

The query returns the difference between the minimum and maximum field values in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z `and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `18`, [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to three and one, and [offsets](/influxdb/v1.2/query_language/data_exploration/#the-offset-and-soffset-clauses) the series returned by one.

## STDDEV()
Returns the standard deviation of [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT STDDEV( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`STDDEV(field_key)`  
&emsp;&emsp;&emsp;
Returns the standard deviation of field values associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`STDDEV(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the standard deviation of field values associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`STDDEV(*)`  
&emsp;&emsp;&emsp;
Returns the standard deviation of field values associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`STDDEV()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples

#### Example 1: Calculate the standard deviation for the field values associated with a field key
```
> SELECT STDDEV("water_level") FROM "h2o_feet"

name: h2o_feet
time                   stddev
----                   ------
1970-01-01T00:00:00Z   2.279144584196141
```

The query returns the standard deviation of the field values in the `water_level` field key and in the `h2o_feet` measurement.

#### Example 2: Calculate the standard deviation for the field values associated with each field key in a measurement
```
> SELECT STDDEV(*) FROM "h2o_feet"

name: h2o_feet
time                   stddev_water_level
----                   ------------------
1970-01-01T00:00:00Z   2.279144584196141
```

The query returns the standard deviation of the field values for each field key that stores numerical values in the `h2o_feet` measurement.
The `h2o_feet` measurement has one numerical field: `water_level`.

#### Example 3: Calculate the standard deviation for the field values associated with each field key that matches a regular expression
```
> SELECT STDDEV(/water/) FROM "h2o_feet"

name: h2o_feet
time                   stddev_water_level
----                   ------------------
1970-01-01T00:00:00Z   2.279144584196141
```

The query returns the standard deviation of the field values for each field key that stores numerical values and includes the word `water` in the `h2o_feet` measurement. 

#### Example 4: Calculate the standard deviation for the field values associated with a field key and include several clauses
```
> SELECT STDDEV("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(18000) LIMIT 2 SLIMIT 1 SOFFSET 1

name: h2o_feet
tags: location=santa_monica
time                   stddev
----                   ------
2015-08-17T23:48:00Z   18000
2015-08-18T00:00:00Z   0.03676955262170051
```

The query returns the standard deviation of the field values in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `18000`, [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to two and one, and [offsets](/influxdb/v1.2/query_language/data_exploration/#the-offset-and-soffset-clauses) the series returned by one.

## SUM()
Returns the sum of [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT SUM( [ * | <field_key> | /<regular_expression>/ ] ) [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`SUM(field_key)`  
&emsp;&emsp;&emsp;
Returns the sum of field values associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`SUM(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the sum of field values associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`SUM(*)`  
&emsp;&emsp;&emsp;
Returns the sums of field values associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`SUM()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples:

#### Example 1: Calculate the sum of the field values associated with a field key
```
> SELECT SUM("water_level") FROM "h2o_feet"

name: h2o_feet
time                   sum
----                   ---
1970-01-01T00:00:00Z   67777.66900000004
```

The query returns the summed total of the field values in the `water_level` field key and in the `h2o_feet` measurement.

#### Example 2: Calculate the sum of the field values associated with each field key in a measurement
```
> SELECT SUM(*) FROM "h2o_feet"

name: h2o_feet
time                   sum_water_level
----                   ---------------
1970-01-01T00:00:00Z   67777.66900000004
```

The query returns the summed total of the field values for each field key that stores numerical values in the `h2o_feet` measurement.
The `h2o_feet` measurement has one numerical field: `water_level`.

#### Example 3: Calculate the sum of the field values associated with each field key that matches a regular expression
```
> SELECT SUM(/water/) FROM "h2o_feet"

name: h2o_feet
time                   sum_water_level
----                   ---------------
1970-01-01T00:00:00Z   67777.66900000004
```

The query returns the summed total of the field values for each field key that stores numerical values and includes the word `water` in the `h2o_feet` measurement.

#### Example 4: Calculate the sum of the field values associated with a field key and include several clauses
```
> SELECT SUM("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(18000) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   sum
----                   ---
2015-08-17T23:48:00Z   18000
2015-08-18T00:00:00Z   16.125
2015-08-18T00:12:00Z   15.649
2015-08-18T00:24:00Z   15.135
```

The query returns the summed total of the field values in the `water_level` field.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag. The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with 18000, and it [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to four and one.

# Selectors

## BOTTOM()
Returns the smallest `N` [field values](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT BOTTOM(<field_key>[,<tag_key(s)>],<N> )[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`BOTTOM(field_key,N)`  
&emsp;&emsp;&emsp;
Returns the smallest N field values associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`BOTTOM(field_key,tag_key(s),N)`  
&emsp;&emsp;&emsp;
Returns the smallest field value for N tag values of the [tag key](/influxdb/v1.2/concepts/glossary/#tag-key).

`BOTTOM(field_key,N),tag_key(s),field_key(s)`  
&emsp;&emsp;&emsp;
Returns the smallest N field values associated with the field key in the parentheses and the relevant [tag](/influxdb/v1.2/concepts/glossary/#tag) and/or [field](/influxdb/v1.2/concepts/glossary/#field).

`BOTTOM()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

> **Note:** `BOTTOM()` returns the field value with the earliest timestamp if there's a tie between two or more values for the smallest value.

### Examples

#### Example 1: Select the bottom three field values associated with a field key
```
> SELECT BOTTOM("water_level",3) FROM "h2o_feet"

name: h2o_feet
time                   bottom
----                   ------
2015-08-29T14:30:00Z   -0.61
2015-08-29T14:36:00Z   -0.591
2015-08-30T15:18:00Z   -0.594
```
The query returns the smallest three field values in the `water_level` field key and in the `h2o_feet` [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

#### Example 2: Select the bottom field value associated with a field key for two tags
```
> SELECT BOTTOM("water_level","location",2) FROM "h2o_feet"

name: h2o_feet
time                   bottom   location
----                   ------   --------
2015-08-29T10:36:00Z   -0.243   santa_monica
2015-08-29T14:30:00Z   -0.61    coyote_creek
```
The query returns the smallest field values in the `water_level` field key for two tag values associated with the `location` tag key.

#### Example 3: Select the bottom four field values associated with a field key and the relevant tags and fields
```
> SELECT BOTTOM("water_level",4),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  bottom  location      level description
----                  ------  --------      -----------------
2015-08-29T14:24:00Z  -0.587  coyote_creek  below 3 feet
2015-08-29T14:30:00Z  -0.61   coyote_creek  below 3 feet
2015-08-29T14:36:00Z  -0.591  coyote_creek  below 3 feet
2015-08-30T15:18:00Z  -0.594  coyote_creek  below 3 feet
```
The query returns the smallest four field values in the `water_level` field key and the relevant values of the `location` tag key and the `level description` field key.

#### Example 4: Select the bottom three field values associated with a field key and include several clauses
```
> SELECT BOTTOM("water_level",3),"location" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(24m) ORDER BY time DESC

name: h2o_feet
time                   bottom   location
----                   ------   --------
2015-08-18T00:48:00Z   1.991    santa_monica
2015-08-18T00:48:00Z   2.054    santa_monica
2015-08-18T00:48:00Z   6.982    coyote_creek
2015-08-18T00:24:00Z   2.041    santa_monica
2015-08-18T00:24:00Z   2.051    santa_monica
2015-08-18T00:24:00Z   2.057    santa_monica
2015-08-18T00:00:00Z   2.028    santa_monica
2015-08-18T00:00:00Z   2.064    santa_monica
2015-08-18T00:00:00Z   2.116    santa_monica
```
The query returns the smallest three values in the `water_level` field key for each 24-minute [interval](/influxdb/v1.2/query_language/data_exploration/#basic-group-by-time-syntax) between `2015-08-18T00:00:00Z` and `2015-08-18T00:54:00Z`.
It also returns results in [descending timestamp](/influxdb/v1.2/query_language/data_exploration/#order-by-time-desc) order.

Notice that the [`GROUP BY time()` clause](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals) overrides the points' original timestamps.
The timestamps in the results indicate the the start of each 24-minute time interval;
the last three points in the results are for the time interval between `2015-08-18T00:00:00Z` and just before `2015-08-18T00:24:00Z`.

### Common Issues with `BOTTOM()`

#### Issue 1: BOTTOM(), the INTO clause, and the GROUP BY time() clause

Using the `BOTTOM()` function with the [`INTO` clause](/influxdb/v1.2/query_language/data_exploration/#the-into-clause)
and the [`GROUP BY time()` clause](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals) can cause InfluxDB to overwrite points in the destination measurement.
Using `BOTTOM()` with the `GROUP BY time()` clause often returns several results with the same timestamp; InfluxDB assumes [points](/influxdb/v1.2/concepts/glossary/#point) with the same series and timestamp are duplicate points and simply overwrites any duplicate point with the most recent point in the destination measurement.

##### Example
<br>
The first query in the codeblock below uses the `BOTTOM()` function with a `GROUP BY time()` clause, and it returns four results.
Notice that the first two results have the same timestamp and the last two results have the same timestamp.
The second query adds an `INTO` clause to the initial query and writes the query results to the `bottom_dweller` measurement.
The last query in the codeblock selects all the data in the `bottom_dweller` measurement.

The last query returns two points instead of four points, because two of the initial results are duplicate points; they belong to the same series and have the same timestamp.
When the system encounters duplicate points, it simply overwrites the previous point with the most recent point.

```
> SELECT BOTTOM("water_level",2),"location" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:24:00Z' GROUP BY time(24m)

name: h2o_feet
time                   bottom   location
----                   ------   --------
2015-08-18T00:00:00Z   2.028    santa_monica
2015-08-18T00:00:00Z   2.064    santa_monica
2015-08-18T00:24:00Z   2.041    santa_monica
2015-08-18T00:24:00Z   7.635    coyote_creek

> SELECT BOTTOM("water_level",2),"location" INTO "bottom_dweller" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:24:00Z' GROUP BY time(24m)

name: result
time                   written
----                   -------
1970-01-01T00:00:00Z   4

> SELECT * FROM "bottom_dweller"

name: bottom_dweller
time                   bottom   location
----                   ------   --------
2015-08-18T00:00:00Z   2.064    santa_monica
2015-08-18T00:24:00Z   7.635    coyote_creek
```

#### Issue 2: BOTTOM() and a tag key with fewer than N tag values

Queries with the syntax `SELECT BOTTOM(<field_key>,<tag_key>,<N>)` can return fewer points than expected.
If the tag key has `X` tag values, the query specifies `N` values, and `X` is smaller than `N`, then the query returns `X` points.

##### Example
<br>
The query below asks for the smallest field values of `water_level` for three tag values of the `location` tag key.
Because the `location` tag key has two tag values (`santa_monica` and `coyote_creek`), the query returns two points instead of three.
```
> SELECT BOTTOM("water_level","location",3) FROM "h2o_feet"

name: h2o_feet
time                   bottom   location
----                   ------   --------
2015-08-29T10:36:00Z   -0.243   santa_monica
2015-08-29T14:30:00Z   -0.61    coyote_creek
```

## FIRST()
Returns the [field value ](/influxdb/v1.2/concepts/glossary/#field-value) with the oldest timestamp.

### Syntax
```
SELECT FIRST(<field_key>)[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`FIRST(field_key)`  
&emsp;&emsp;&emsp;
Returns the oldest field value (determined by timestamp) associated with the field key.

`FIRST(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the oldest field value (determined by timestamp) associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`FIRST(*)`  
&emsp;&emsp;&emsp;
Returns the oldest field value (determined by timestamp) associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`FIRST(field_key),tag_key(s),field_key(s)`  
&emsp;&emsp;&emsp;
Returns the oldest field value (determined by timestamp) associated with the field key in the parentheses and the relevant [tag](/influxdb/v1.2/concepts/glossary/#tag) and/or [field](/influxdb/v1.2/concepts/glossary/#field).

`FIRST()` supports all field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples

#### Example 1: Select the first field value associated with a field key
```
> SELECT FIRST("level description") FROM "h2o_feet"

name: h2o_feet
time                   first
----                   -----
2015-08-18T00:00:00Z   between 6 and 9 feet
```
The query returns the oldest field value (determined by timestamp) associated with the `level description` field key and in the `h2o_feet` measurement.

#### Example 2: Select the first field value associated with each field key in a measurement
```
> SELECT FIRST(*) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 6 and 9 feet      8.12
```
The query returns the oldest field value (determined by timestamp) for each field key in the `h2o_feet` measurement.
The `h2o_feet` measurement has two fields: `level description` and `water_level`.

#### Example 3: Select the first field value associated with each field key that matches a regular expression
```
> SELECT FIRST(/level/) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 6 and 9 feet      8.12
```
The query returns the oldest field value for each field key that includes the word `level` in the `h2o_feet` measurement.

#### Example 4: Select the first value associated with a field key and the relevant tags and fields
```
> SELECT FIRST("level description"),"location","water_level" FROM "h2o_feet"

name: h2o_feet
time                  first                 location      water_level
----                  -----                 --------      -----------
2015-08-18T00:00:00Z  between 6 and 9 feet  coyote_creek  8.12
```
The query returns the oldest field value (determined by timestamp) in the `level description` field key and the relevant values of the `location` tag key and the `water_level` field key.

#### Example 5: Select the first field value associated with a field key and include several clauses
```
> SELECT FIRST("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   first
----                   -----
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.12
2015-08-18T00:12:00Z   7.887
2015-08-18T00:24:00Z   7.635
```
The query returns the oldest field value (determined by timestamp) in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `9.01`, and it [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to four and one.

Notice that the [`GROUP BY time()` clause](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals) overrides the points' original timestamps.
The timestamps in the results indicate the the start of each 12-minute time interval;
the first point in the results covers the time interval between `2015-08-17T23:48:00Z` and just before `2015-08-18T00:00:00Z` and the last point in the results covers the time interval between `2015-08-18T00:24:00Z` and just before `2015-08-18T00:36:00Z`.

## LAST()
Returns the [field value](/influxdb/v1.2/concepts/glossary/#field-value) with the most recent timestamp.

### Syntax
```
SELECT LAST(<field_key>)[,<tag_key(s)>|<field_keys(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`LAST(field_key)`  
&emsp;&emsp;&emsp;
Returns the newest field value (determined by timestamp) associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`LAST(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the newest field value (determined by timestamp) associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`LAST(*)`  
&emsp;&emsp;&emsp;
Returns the newest field value (determined by timestamp) associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`LAST(field_key),tag_key(s),field_key(s)`  
&emsp;&emsp;&emsp;
Returns the newest field value (determined by timestamp) associated with the field key in the parentheses and the relevant [tag](/influxdb/v1.2/concepts/glossary/#tag) and/or [field](/influxdb/v1.2/concepts/glossary/#field).

`LAST()` supports all field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples

#### Example 1: Select the last field values associated with a field key
```
> SELECT LAST("level description") FROM "h2o_feet"

name: h2o_feet
time                   last
----                   ----
2015-09-18T21:42:00Z   between 3 and 6 feet
```
The query returns the newest field value (determined by timestamp) associated with the `level description` field key and in the `h2o_feet` measurement.

#### Example 2: Select the last field values associated with each field key in a measurement
```
> SELECT LAST(*) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 3 and 6 feet      4.938
```
The query returns the newest field value (determined by timestamp) for each field key in the `h2o_feet` measurement.
The `h2o_feet` measurement has two fields: `level description` and `water_level`.

#### Example 3: Select the last field value associated with each field key that matches a regular expression
```
> SELECT LAST(/level/) FROM "h2o_feet"

name: h2o_feet
time                   first_level description   first_water_level
----                   -----------------------   -----------------
1970-01-01T00:00:00Z   between 3 and 6 feet      4.938
```
The query returns the newest field value for each field key that includes the word `level` in the `h2o_feet` measurement.

#### Example 4: Select the last field value associated with a field key and the relevant tags and fields
```
> SELECT LAST("level description"),"location","water_level" FROM "h2o_feet"

name: h2o_feet
time                  last                  location      water_level
----                  ----                  --------      -----------
2015-09-18T21:42:00Z  between 3 and 6 feet  santa_monica  4.938
```
The query returns the newest field value (determined by timestamp) in the `level description` field key and the relevant values of the `location` tag key and the `water_level` field key.

#### Example 5: Select the last field value associated with a field key and include several clauses
```
> SELECT LAST("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   last
----                   ----
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.005
2015-08-18T00:12:00Z   7.762
2015-08-18T00:24:00Z   7.5
```

The query returns the newest field value (determined by timestamp) in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results into 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `9.01`, and it [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to four and one.

Notice that the [`GROUP BY time()` clause](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals) overrides the points' original timestamps.
The timestamps in the results indicate the the start of each 12-minute time interval;
the first point in the results covers the time interval between `2015-08-17T23:48:00Z` and just before `2015-08-18T00:00:00Z` and the last point in the results covers the time interval between `2015-08-18T00:24:00Z` and just before `2015-08-18T00:36:00Z`.

## MAX()
Returns the greatest [field value](/influxdb/v1.2/concepts/glossary/#field-value).

### Syntax
```
SELECT MAX(<field_key>)[,<tag_key(s)>|<field__key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```

### Description of Syntax

`MAX(field_key)`  
&emsp;&emsp;&emsp;
Returns the greatest field value associated with the [field key](/influxdb/v1.2/concepts/glossary/#field-key).

`MAX(/regular_expression/)`  
&emsp;&emsp;&emsp;
Returns the greatest field value associated with each field key that matches the [regular expression](/influxdb/v1.2/query_language/data_exploration/#regular-expressions).

`MAX(*)`  
&emsp;&emsp;&emsp;
Returns the greatest field value associated with each field key in the [measurement](/influxdb/v1.2/concepts/glossary/#measurement).

`MAX(field_key),tag_key(s),field_key(s)`  
&emsp;&emsp;&emsp;
Returns the greatest field value associated with the field key in the parentheses and the relevant [tag](/influxdb/v1.2/concepts/glossary/#tag) and/or [field](/influxdb/v1.2/concepts/glossary/#field).

`MAX()` supports int64 and float64 field value [data types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).

### Examples

#### Example 1: Select the maximum field value associated with a field key
```
> SELECT MAX("water_level") FROM "h2o_feet"

name: h2o_feet
time                   max
----                   ---
2015-08-29T07:24:00Z   9.964
```
The query returns the greatest field value in the `water_level` field key and in the `h2o_feet` measurement.

#### Example 2: Select the maximum field value associated with each field key in a measurement
```
> SELECT MAX(*) FROM "h2o_feet"

name: h2o_feet
time                   max_water_level
----                   ---------------
2015-08-29T07:24:00Z   9.964
```
The query returns the greatest field value for each field key that stores numerical values in the `h2o_feet` measurement.
The `h2o_feet` measurement has one numerical field: `water_level`.

#### Example 3: Select the maximum field value associated with each field key that matches a regular expression
```
> SELECT MAX(/level/) FROM "h2o_feet"

name: h2o_feet
time                   max_water_level
----                   ---------------
2015-08-29T07:24:00Z   9.964
```
The query returns the greatest field value for each field key that stores numerical values and includes the word `water` in the `h2o_feet` measurement.

#### Example 4: Select the maximum field value associated with a field key and the relevant tags and fields
```
> SELECT MAX("water_level"),"location","level description" FROM "h2o_feet"

name: h2o_feet
time                  max    location      level description
----                  ---    --------      -----------------
2015-08-29T07:24:00Z  9.964  coyote_creek  at or greater than 9 feet
```
The query returns the greatest field value in the `water_level` field key and the relevant values of the `location` tag key and the `level description` field key.

#### Example 5: Select the maximum field value associated with a field key and include several clauses
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(12m),* fill(9.01) LIMIT 4 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   max
----                   ---
2015-08-17T23:48:00Z   9.01
2015-08-18T00:00:00Z   8.12
2015-08-18T00:12:00Z   7.887
2015-08-18T00:24:00Z   7.635
```
The query returns the greatest field value in the `water_level` field key.
It covers the [time range](/influxdb/v1.2/query_language/data_exploration/#time-syntax) between `2015-08-17T23:48:00Z` and `2015-08-18T00:54:00Z` and [groups](/influxdb/v1.2/query_language/data_exploration/#the-group-by-clause) results in to 12-minute time intervals and per tag.
The query [fills](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill) empty time intervals with `9.01`, and it [limits](/influxdb/v1.2/query_language/data_exploration/#the-limit-and-slimit-clauses) the number of points and series returned to four and one.

Notice that the [`GROUP BY time()` clause](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals) overrides the points’ original timestamps.
The timestamps in the results indicate the the start of each 12-minute time interval;
the first point in the results covers the time interval between `2015-08-17T23:48:00Z` and just before `2015-08-18T00:00:00Z` and the last point in the results covers the time interval between `2015-08-18T00:24:00Z` and just before `2015-08-18T00:36:00Z`.

## MIN()
Returns the lowest value in a single [field](/influxdb/v1.2/concepts/glossary/#field).
The field must be an int64, float64, or boolean.
```
SELECT MIN(<field_key>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

Examples:

* Select the minimum `water_level` in the measurement `h2o_feet`:

```
> SELECT MIN("water_level") FROM "h2o_feet"
name: h2o_feet
--------------
time			               min
2015-08-29T14:30:00Z	 -0.61
```

* Select the minimum `water_level` in the measurement `h2o_feet` and output the
relevant `location` tag:

```
> SELECT MIN("water_level"),"location" FROM "h2o_feet"
name: h2o_feet
--------------
time			              min	   location
2015-08-29T14:30:00Z	-0.61	 coyote_creek
```

* Select the minimum `water_level` in the measurement `h2o_feet` between August 18, 2015 at midnight and August 18, at 00:48 grouped at 12 minute intervals and by the `location` tag:

```
> SELECT MIN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time < '2015-08-18T00:54:00Z' GROUP BY time(12m), "location"
name: h2o_feet
tags: location = coyote_creek
time			                 min
----			                 ---
2015-08-18T00:00:00Z	   8.005
2015-08-18T00:12:00Z	   7.762
2015-08-18T00:24:00Z	   7.5
2015-08-18T00:36:00Z	   7.234
2015-08-18T00:48:00Z	   7.11

name: h2o_feet
tags: location = santa_monica
time			                 min
----			                 ---
2015-08-18T00:00:00Z	   2.064
2015-08-18T00:12:00Z	   2.028
2015-08-18T00:24:00Z	   2.041
2015-08-18T00:36:00Z	   2.057
2015-08-18T00:48:00Z	   1.991
```

## PERCENTILE()
Returns the `N`th percentile value for the sorted values of a single [field](/influxdb/v1.2/concepts/glossary/#field).
The field must be of type int64 or float64.
The percentile `N` must be an integer or floating point number between 0 and 100, inclusive.
```
SELECT PERCENTILE(<field_key>, <N>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

Examples:

* Calculate the fifth percentile of the field `water_level` where the tag `location` equals `coyote_creek`:

```
> SELECT PERCENTILE("water_level",5) FROM "h2o_feet" WHERE "location" = 'coyote_creek'
name: h2o_feet
--------------
time			               percentile
2015-09-09T11:42:00Z	 1.148
```

 The value `1.148` is larger than 5% of the values in `water_level` where `location` equals `coyote_creek`.

* Calculate the fifth percentile of the field `water_level` and output the
relevant `location` tag:

```
> SELECT PERCENTILE("water_level",5),"location" FROM "h2o_feet"
name: h2o_feet
--------------
time	                  percentile	 location
2015-08-28T12:06:00Z	  1.122		     santa_monica
```

* Calculate the 100th percentile of the field `water_level` grouped by the `location` tag:

```
> SELECT PERCENTILE("water_level", 100) FROM "h2o_feet" GROUP BY "location"
name: h2o_feet
tags: location = coyote_creek
time			               percentile
----			               ----------
2015-08-29T07:24:00Z	 9.964

name: h2o_feet
tags: location = santa_monica
time			               percentile
----			               ----------
2015-08-29T03:54:00Z	 7.205
```

Notice that `PERCENTILE(<field_key>,100)` is equivalent to `MAX(<field_key>)`.

<dt> Currently, `PERCENTILE(<field_key>,0)` is not equivalent to `MIN(<field_key>)`.
See GitHub Issue [#4418](https://github.com/influxdata/influxdb/issues/4418) for more information.
</dt>

> **Note**: `PERCENTILE(<field_key>, 50)` is nearly equivalent to `MEDIAN()`, except `MEDIAN()` returns the average of the two middle values if the field contains an even number of points.

## SAMPLE()
Returns a random sample of `N` points for the specified [field key](/influxdb/v1.2/concepts/glossary/#field).
InfluxDB uses [reservoir sampling](https://en.wikipedia.org/wiki/Reservoir_sampling) to generate the random points.
`SAMPLE()` supports all [field types](/influxdb/v1.2/write_protocols/line_protocol_reference/#data-types).
```
SELECT SAMPLE(<field_key>,<N>) FROM_clause [WHERE_clause] [GROUP_BY_clause]
```

### Examples

#### Example 1: Select a random sample of two points
```
> SELECT SAMPLE("water_level",2) FROM "h2o_feet"

name: h2o_feet
time                   sample
----                   ------
2015-09-09T21:48:00Z   5.659
2015-09-18T10:00:00Z   6.939
```

The query returns two randomly selected points from the `water_level` field
in the `h2o_feet` measurement.

#### Example 2: Select a random sample of two points per `GROUP BY time()` interval
```
> SELECT SAMPLE("water_level",1) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(18m)

name: h2o_feet
time                   sample
----                   ------
2015-08-18T00:12:00Z   2.028
2015-08-18T00:30:00Z   2.051
```

The query returns one randomly selected point per 18-minute `GROUP BY time()`
interval.
Note that the timestamps returned are the points' original timestamps.

### Common Issues with `SAMPLE()`

#### Issue 1: `SAMPLE()` with a `GROUP BY time()` clause
Queries with `SAMPLE()` and a `GROUP BY time()` clause return the specified
number of points (`N`) per `GROUP BY time()` interval.
For
[most `GROUP BY time()` queries](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals),
the returned timestamps mark the start of the `GROUP BY time()` interval.
`GROUP BY time()` queries with the `SAMPLE()` function behave differently;
they maintain the timestamp of the original data point.

The query below returns two randomly selected points per 18-minute
`GROUP BY time()` interval.
Notice that the returned timestamps are the original timestamps; they
are not forced to match the start of the `GROUP BY time()` intervals.

```
> SELECT SAMPLE("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(18m)

name: h2o_feet
time                   sample
----                   ------
                           __
2015-08-18T00:06:00Z   2.116 |
2015-08-18T00:12:00Z   2.028 | <------- Randomly-selected points for the first time interval
                           --
                           __
2015-08-18T00:18:00Z   2.126 |
2015-08-18T00:30:00Z   2.051 | <------- Randomly-selected points for the second time interval
                           --
```

## TOP()
Returns the largest `N` values in a single [field](/influxdb/v1.2/concepts/glossary/#field).
The field type must be int64 or float64.
```
SELECT TOP(<field_key>[,<tag_keys>],<N>)[,<tag_keys>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

Examples:

* Select the largest three values of `water_level`:

```
> SELECT TOP("water_level",3) FROM "h2o_feet"
name: h2o_feet
--------------
time			               top
2015-08-29T07:18:00Z	 9.957
2015-08-29T07:24:00Z	 9.964
2015-08-29T07:30:00Z	 9.954
```

* Select the largest three values of `water_level` and include the relevant `location` tag in the output:

```
> SELECT TOP("water_level",3),"location" FROM "h2o_feet"
name: h2o_feet
--------------
time			               top	   location
2015-08-29T07:18:00Z	 9.957	 coyote_creek
2015-08-29T07:24:00Z	 9.964	 coyote_creek
2015-08-29T07:30:00Z	 9.954	 coyote_creek
```

* Select the largest value of `water_level` within each tag value of `location`:

```
> SELECT TOP("water_level","location",2) FROM "h2o_feet"
name: h2o_feet
--------------
time			               top	   location
2015-08-29T03:54:00Z	 7.205	 santa_monica
2015-08-29T07:24:00Z	 9.964	 coyote_creek
```

The output shows the top values of `water_level` for each tag value of `location` (`santa_monica` and `coyote_creek`).

> **Note:** Queries with the syntax `SELECT TOP(<field_key>,<tag_key>,<N>)`, where the tag has `X` distinct values, return `N` or `X` field values, whichever is smaller, and each returned point has a unique tag value.
To demonstrate this behavior, see the results of the above example query where `N` equals `3` and `N` equals `1`.

> * `N` = `3`

>
```
> SELECT TOP("water_level","location",3) FROM "h2o_feet"
name: h2o_feet
--------------
time			               top	   location
2015-08-29T03:54:00Z	 7.205	 santa_monica
2015-08-29T07:24:00Z	 9.964	 coyote_creek
```

> InfluxDB returns two values instead of three because the `location` tag has only two values (`santa_monica` and `coyote_creek`).

> * `N` = `1`

>
```
> SELECT TOP("water_level","location",1) FROM "h2o_feet"
name: h2o_feet
--------------
time			               top	   location
2015-08-29T07:24:00Z	 9.964	 coyote_creek
```

> InfluxDB compares the top values of `water_level` within each tag value of `location` and returns the larger value of `water_level`.

* Select the largest two values of `water_level` between August 18, 2015 at 4:00:00 and August 18, 2015 at 4:18:00 for every tag value of `location`:

```
> SELECT TOP("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-18T04:00:00Z' AND time < '2015-08-18T04:24:00Z' GROUP BY "location"
name: h2o_feet
tags: location=coyote_creek
time			               top
----			               ---
2015-08-18T04:00:00Z	 2.943
2015-08-18T04:06:00Z	 2.831


name: h2o_feet
tags: location=santa_monica
time			               top
----			               ---
2015-08-18T04:06:00Z	 4.055
2015-08-18T04:18:00Z	 4.124
```

* Select the largest two values of `water_level` between August 18, 2015 at 4:00:00 and August 18, 2015 at 4:18:00 in `santa_monica`:

```
> SELECT TOP("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-18T04:00:00Z' AND time < '2015-08-18T04:24:00Z' AND "location" = 'santa_monica'
name: h2o_feet
--------------
time			               top
2015-08-18T04:06:00Z	 4.055
2015-08-18T04:18:00Z	 4.124
```

Note that in the raw data, `water_level` equals `4.055` at `2015-08-18T04:06:00Z` and at `2015-08-18T04:12:00Z`.
In the case of a tie, InfluxDB returns the value with the earlier timestamp.

# Transformations

## CEILING()
`CEILING()` is not yet functional.

<dt> See GitHub Issue [#5930](https://github.com/influxdata/influxdb/issues/5930) for more information.
</dt>

## CUMULATIVE_SUM()
Returns the cumulative sum of consecutive field values for a single
[field](/influxdb/v1.2/concepts/glossary/#field).
The field type must be int64 or float64.

### Basic CUMULATIVE_SUM() Syntax
```
SELECT CUMULATIVE_SUM(<field_key>) FROM_clause WHERE_clause
```

### Advanced CUMULATIVE_SUM() Syntax
```
SELECT CUMULATIVE_SUM(<function>(<field_key>)) FROM_clause WHERE_clause GROUP BY time(<interval>)[,<tag_key>]
```

Supported functions:
[`COUNT()`](#count),
[`MEAN()`](#mean),
[`MEDIAN()`](#median),
[`MODE()`](#mode),
[`SUM()`](#sum),
[`FIRST()`](#first),
[`LAST()`](#last),
[`MIN()`](#min),
[`MAX()`](#max), and
[`PERCENTILE()`](#percentile).

### Examples

The examples below work with the following subsample of the `NOAA_water_database`
data:
```
> SELECT "water_level" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica'

name: h2o_feet
time                   water_level
----                   -----------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   2.116
2015-08-18T00:12:00Z   2.028
2015-08-18T00:18:00Z   2.126
2015-08-18T00:24:00Z   2.041
2015-08-18T00:30:00Z   2.051
```

#### Example 1: Use cumulative sum on a single time range
```
> SELECT CUMULATIVE_SUM("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica'

name: h2o_feet
time                   cumulative_sum
----                   --------------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   4.18
2015-08-18T00:12:00Z   6.208
2015-08-18T00:18:00Z   8.334
2015-08-18T00:24:00Z   10.375
2015-08-18T00:30:00Z   12.426
```

The query returns the cumulative sum of `water_level`'s field values.
The second point in the results is the sum of `2.064` and `2.116`, the third point is the sum of `2.064`, `2.116`, and `2.028`, and so on.

#### Example 2: Use cumulative sum with a `GROUP BY time()` clause
```
> SELECT CUMULATIVE_SUM(MEAN("water_level")) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(12m)

name: h2o_feet
time                   cumulative_sum
----                   --------------
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   4.167
2015-08-18T00:24:00Z   6.213
```

The query returns the cumulative sum of average `water_level`s that are calculated at 12-minute intervals between `2015-08-18T00:00:00Z` and `2015-08-18T00:30:00Z`.

To get those results, InfluxDB first calculates average `water_level`s at 12-minute intervals.
This step is the same as using the raw `MEAN()` function:
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' AND "location" = 'santa_monica' GROUP BY time(12m)

name: h2o_feet
time                   mean
----                   ----
2015-08-18T00:00:00Z   2.09
2015-08-18T00:12:00Z   2.077
2015-08-18T00:24:00Z   2.0460000000000003
```

Next, InfluxDB calculates the cumulative sum of those averages.
The second point in the final results is the sum of `2.09` and `2.077`
and the third point is the sum of `2.09`, `2.077`, and `2.0460000000000003`.

## DERIVATIVE()
Returns the rate of change for the values in a single [field](/influxdb/v1.2/concepts/glossary/#field) in a [series](/influxdb/v1.2/concepts/glossary/#series).
InfluxDB calculates the difference between chronological field values and converts those results into the rate of change per `unit`.
The `unit` argument is optional and, if not specified, defaults to one second (`1s`).

The basic `DERIVATIVE()` query:
```
SELECT DERIVATIVE(<field_key>, [<unit>]) FROM <measurement_name> [WHERE <stuff>]
```

Valid time specifications for `unit` are:  
`u` or `µ`&emsp;microseconds  
`ms`&nbsp;&nbsp;&emsp;&emsp;&nbsp;&nbsp;&nbsp;milliseconds  
`s`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;seconds  
`m`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;minutes  
`h`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;hours  
`d`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;days  
`w`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;weeks

`DERIVATIVE()` also works with a nested function coupled with a `GROUP BY time()` clause.
For queries that include those options, InfluxDB first performs the aggregation, selection, or transformation across the time interval specified in the `GROUP BY time()` clause.
It then calculates the difference between chronological field values and
converts those results into the rate of change per `unit`.
The `unit` argument is optional and, if not specified, defaults to the same
interval as the `GROUP BY time()` interval.


The `DERIVATIVE()` query with an aggregation function and `GROUP BY time()` clause:
```
SELECT DERIVATIVE(AGGREGATION_FUNCTION(<field_key>),[<unit>]) FROM <measurement_name> WHERE <stuff> GROUP BY time(<aggregation_interval>)
```

Examples:

The following examples work with the first six observations of the `water_level` field in the measurement `h2o_feet` with the tag set `location = santa_monica`:
```bash
name: h2o_feet
--------------
time			               water_level
2015-08-18T00:00:00Z	 2.064
2015-08-18T00:06:00Z	 2.116
2015-08-18T00:12:00Z	 2.028
2015-08-18T00:18:00Z	 2.126
2015-08-18T00:24:00Z	 2.041
2015-08-18T00:30:00Z	 2.051
```

* `DERIVATIVE()` with a single argument:  
Calculate the rate of change per one second

```
> SELECT DERIVATIVE("water_level") FROM "h2o_feet" WHERE "location" = 'santa_monica' LIMIT 5
name: h2o_feet
--------------
time			               derivative
2015-08-18T00:06:00Z	 0.00014444444444444457
2015-08-18T00:12:00Z	 -0.00024444444444444465
2015-08-18T00:18:00Z	 0.0002722222222222218
2015-08-18T00:24:00Z	 -0.000236111111111111
2015-08-18T00:30:00Z	 2.777777777777842e-05
```

Notice that the first field value (`0.00014`) in the `derivative` column is **not** `0.052` (the difference between the first two field values in the raw data: `2.116` - `2.604` = `0.052`).
Because the query does not specify the `unit` option, InfluxDB automatically calculates the rate of change per one second, not the rate of change per six minutes.
The calculation of the first value in the `derivative` column looks like this:
<br>
<br>
```
(2.116 - 2.064) / (360s / 1s)
```

The numerator is the difference between chronological field values.
The denominator is the difference between the relevant timestamps in seconds (`2015-08-18T00:06:00Z` - `2015-08-18T00:00:00Z` = `360s`) divided by `unit` (`1s`).
This returns the rate of change per second from `2015-08-18T00:00:00Z` to `2015-08-18T00:06:00Z`.

* `DERIVATIVE()` with two arguments:  
Calculate the rate of change per six minutes

```
> SELECT DERIVATIVE("water_level",6m) FROM "h2o_feet" WHERE "location" = 'santa_monica' LIMIT 5
name: h2o_feet
--------------
time			               derivative
2015-08-18T00:06:00Z	 0.052000000000000046
2015-08-18T00:12:00Z	 -0.08800000000000008
2015-08-18T00:18:00Z	 0.09799999999999986
2015-08-18T00:24:00Z	 -0.08499999999999996
2015-08-18T00:30:00Z	 0.010000000000000231
```

The calculation of the first value in the `derivative` column looks like this:
<br>
<br>
```
(2.116 - 2.064) / (6m / 6m)
```

The numerator is the difference between chronological field values.
The denominator is the difference between the relevant timestamps in minutes (`2015-08-18T00:06:00Z` - `2015-08-18T00:00:00Z` = `6m`) divided by `unit` (`6m`).
This returns the rate of change per six minutes from `2015-08-18T00:00:00Z` to `2015-08-18T00:06:00Z`.

* `DERIVATIVE()` with two arguments:  
Calculate the rate of change per 12 minutes

```
> SELECT DERIVATIVE("water_level",12m) FROM "h2o_feet" WHERE "location" = 'santa_monica' LIMIT 5
name: h2o_feet
--------------
time			               derivative
2015-08-18T00:06:00Z	 0.10400000000000009
2015-08-18T00:12:00Z	 -0.17600000000000016
2015-08-18T00:18:00Z	 0.19599999999999973
2015-08-18T00:24:00Z	 -0.16999999999999993
2015-08-18T00:30:00Z	 0.020000000000000462
```

The calculation of the first value in the `derivative` column looks like this:
<br>
<br>
```
(2.116 - 2.064 / (6m / 12m)
```

The numerator is the difference between chronological field values.
The denominator is the difference between the relevant timestamps in minutes (`2015-08-18T00:06:00Z` - `2015-08-18T00:00:00Z` = `6m`) divided by `unit` (`12m`).
This returns the rate of change per 12 minutes from `2015-08-18T00:00:00Z` to `2015-08-18T00:06:00Z`.

> **Note:** Specifying `12m` as the `unit` **does not** mean that InfluxDB calculates the rate of change for every 12 minute interval of data.
Instead, InfluxDB calculates the rate of change per 12 minutes for each interval of valid data.

* `DERIVATIVE()` with one argument, a function, and a `GROUP BY time()` clause:  
Select the `MAX()` value at 12 minute intervals and calculate the rate of change per 12 minutes

```
> SELECT DERIVATIVE(MAX("water_level")) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time < '2015-08-18T00:36:00Z' GROUP BY time(12m)
name: h2o_feet
--------------
time			               derivative
2015-08-18T00:12:00Z	 0.009999999999999787
2015-08-18T00:24:00Z	 -0.07499999999999973
```

To get those results, InfluxDB first aggregates the data by calculating the `MAX()` `water_level` at the time interval specified in the `GROUP BY time()` clause (`12m`).
Those results look like this:
```bash
name: h2o_feet
--------------
time			               max
2015-08-18T00:00:00Z	 2.116
2015-08-18T00:12:00Z	 2.126
2015-08-18T00:24:00Z	 2.051
```

Second, InfluxDB calculates the rate of change per `12m` (the same interval as the `GROUP BY time()` interval) to get the results in the `derivative` column above.
The calculation of the first value in the `derivative` column looks like this:
<br>
<br>
```
(2.126 - 2.116) / (12m / 12m)
```

The numerator is the difference between chronological field values.
The denominator is the difference between the relevant timestamps in minutes (`2015-08-18T00:12:00Z` - `2015-08-18T00:00:00Z` = `12m`) divided by `unit` (`12m`).
This returns rate of change per 12 minutes for the aggregated data from `2015-08-18T00:00:00Z` to `2015-08-18T00:12:00Z`.

* `DERIVATIVE()` with two arguments, a function, and a `GROUP BY time()` clause:  
Aggregate the data to 18 minute intervals and calculate the rate of change per six minutes

```
> SELECT DERIVATIVE(SUM("water_level"),6m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time < '2015-08-18T00:36:00Z' GROUP BY time(18m)
name: h2o_feet
--------------
time			               derivative
2015-08-18T00:18:00Z	 0.0033333333333332624
```

To get those results, InfluxDB first aggregates the data by calculating the `SUM()` of `water_level` at the time interval specified in the `GROUP BY time()` clause (`18m`).
The aggregated results look like this:
```bash
name: h2o_feet
--------------
time			               sum
2015-08-18T00:00:00Z	 6.208
2015-08-18T00:18:00Z	 6.218
```

Second, InfluxDB calculates the rate of change per `unit` (`6m`) to get the results in the `derivative` column above.
The calculation of the first value in the `derivative` column looks like this:
<br>
<br>
```
(6.218 - 6.208) / (18m / 6m)
```

The numerator is the difference between chronological field values.
The denominator is the difference between the relevant timestamps in minutes (`2015-08-18T00:18:00Z` - `2015-08-18T00:00:00Z` = `18m`) divided by `unit` (`6m`).
This returns the rate of change per six minutes for the aggregated data from `2015-08-18T00:00:00Z` to `2015-08-18T00:18:00Z`.

## DIFFERENCE()
Returns the difference between consecutive chronological values in a single [field](/influxdb/v1.2/concepts/glossary/#field).
The field type must be int64 or float64.

The basic `DIFFERENCE()` query:
```
SELECT DIFFERENCE(<field_key>) FROM <measurement_name> [WHERE <stuff>]
```

The `DIFFERENCE()` query with a nested function and a `GROUP BY time()` clause:
```
SELECT DIFFERENCE(<function>(<field_key>)) FROM <measurement_name> WHERE <stuff> GROUP BY time(<time_interval>)
```

Functions that work with `DIFFERENCE()` include
[`COUNT()`](/influxdb/v1.2/query_language/functions/#count),
[`MEAN()`](/influxdb/v1.2/query_language/functions/#mean),
[`MEDIAN()`](/influxdb/v1.2/query_language/functions/#median),
[`SUM()`](/influxdb/v1.2/query_language/functions/#sum),
[`FIRST()`](/influxdb/v1.2/query_language/functions/#first),
[`LAST()`](/influxdb/v1.2/query_language/functions/#last),
[`MIN()`](/influxdb/v1.2/query_language/functions/#min),
[`MAX()`](/influxdb/v1.2/query_language/functions/#max), and
[`PERCENTILE()`](/influxdb/v1.2/query_language/functions/#percentile).

Examples:

The following examples focus on the field `water_level` in `santa_monica`
between `2015-08-18T00:00:00Z` and `2015-08-18T00:36:00Z`:
```
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time			                water_level
2015-08-18T00:00:00Z	  2.064
2015-08-18T00:06:00Z	  2.116
2015-08-18T00:12:00Z	  2.028
2015-08-18T00:18:00Z	  2.126
2015-08-18T00:24:00Z	  2.041
2015-08-18T00:30:00Z	  2.051
2015-08-18T00:36:00Z	  2.067
```

* Calculate the difference between `water_level` values:

```
> SELECT DIFFERENCE("water_level") FROM "h2o_feet" WHERE "location"='santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time			                difference
2015-08-18T00:06:00Z	  0.052000000000000046
2015-08-18T00:12:00Z	  -0.08800000000000008
2015-08-18T00:18:00Z	  0.09799999999999986
2015-08-18T00:24:00Z	  -0.08499999999999996
2015-08-18T00:30:00Z	  0.010000000000000231
2015-08-18T00:36:00Z	  0.016000000000000014
```

The first value in the `difference` column is `2.116 - 2.064`, and the second
value in the `difference` column is `2.028 - 2.116`.
Please note that the extra decimal places are the result of floating point
inaccuracies.

* Select the minimum `water_level` values at 12 minute intervals and calculate
the difference between those values:

```
> SELECT DIFFERENCE(MIN("water_level")) FROM "h2o_feet" WHERE "location"='santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z' GROUP BY time(12m)
name: h2o_feet
--------------
time			                difference
2015-08-18T00:12:00Z	  -0.03600000000000003
2015-08-18T00:24:00Z	  0.0129999999999999
2015-08-18T00:36:00Z	  0.026000000000000245
```

To get the values in the `difference` column, InfluxDB first selects the `MIN()`
values at 12 minute intervals:
```
> SELECT MIN("water_level") FROM "h2o_feet" WHERE "location"='santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z' GROUP BY time(12m)
name: h2o_feet
--------------
time			                min
2015-08-18T00:00:00Z  	2.064
2015-08-18T00:12:00Z  	2.028
2015-08-18T00:24:00Z  	2.041
2015-08-18T00:36:00Z  	2.067
```

It then uses those values to calculate the difference between chronological
values; the first value in the `difference` column is `2.028 - 2.064`.

## ELAPSED()
Returns the difference between subsequent timestamps in a single
[field](/influxdb/v1.2/concepts/glossary/#field).
The `unit` argument is an optional
[duration literal](/influxdb/v1.2/query_language/spec/#durations)
and, if not specified, defaults to one nanosecond.

```
SELECT ELAPSED(<field_key>, <unit>) FROM <measurement_name> [WHERE <stuff>]
```

Examples:

* Calculate the difference (in nanoseconds) between the timestamps in the field
`h2o_feet`:

```
> SELECT ELAPSED("water_level") FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:24:00Z'
name: h2o_feet
--------------
time			                elapsed
2015-08-18T00:06:00Z	  360000000000
2015-08-18T00:12:00Z	  360000000000
2015-08-18T00:18:00Z	  360000000000
2015-08-18T00:24:00Z	  360000000000
```

* Calculate the number of one minute intervals between the timestamps in the
field `h2o_feet`:

```
> SELECT ELAPSED("water_level",1m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:24:00Z'
name: h2o_feet
--------------
time			                elapsed
2015-08-18T00:06:00Z	  6
2015-08-18T00:12:00Z	  6
2015-08-18T00:18:00Z	  6
2015-08-18T00:24:00Z	  6
```

> **Note:** InfluxDB returns `0` if `unit` is greater than the difference
between the timestamps.
For example, the timestamps in `h2o_feet` occur at six minute intervals.
If the query asks for the number of one hour intervals between the
timestamps, InfluxDB returns `0`:
>
```
> SELECT ELAPSED("water_level",1h) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:24:00Z'
name: h2o_feet
--------------
time			                elapsed
2015-08-18T00:06:00Z	  0
2015-08-18T00:12:00Z	  0
2015-08-18T00:18:00Z	  0
2015-08-18T00:24:00Z	  0
```

## FLOOR()
`FLOOR()` is not yet functional.

<dt> See GitHub Issue [#5930](https://github.com/influxdb/influxdb/issues/5930) for more information.
</dt>

## HISTOGRAM()
`HISTOGRAM()` is not yet functional.

<dt> See GitHub Issue [#5930](https://github.com/influxdb/influxdb/issues/5930) for more information.
</dt>

## MOVING_AVERAGE()
Returns the moving average across a `window` of consecutive chronological field values for a single [field](/influxdb/v1.2/concepts/glossary/#field).
The field type must be int64 or float64.

The basic `MOVING_AVERAGE()` query:
```
SELECT MOVING_AVERAGE(<field_key>,<window>) FROM <measurement_name> [WHERE <stuff>]
```

The `MOVING_AVERAGE()` query with a nested function and a `GROUP BY time()` clause:
```
SELECT MOVING_AVERAGE(<function>(<field_key>),<window>) FROM <measurement_name> WHERE <stuff> GROUP BY time(<time_interval>)
```

Functions that work with `MOVING_AVERAGE()` include
[`COUNT()`](/influxdb/v1.2/query_language/functions/#count),
[`MEAN()`](/influxdb/v1.2/query_language/functions/#mean),
[`MEDIAN()`](/influxdb/v1.2/query_language/functions/#median),
[`SUM()`](/influxdb/v1.2/query_language/functions/#sum),
[`FIRST()`](/influxdb/v1.2/query_language/functions/#first),
[`LAST()`](/influxdb/v1.2/query_language/functions/#last),
[`MIN()`](/influxdb/v1.2/query_language/functions/#min),
[`MAX()`](/influxdb/v1.2/query_language/functions/#max), and
[`PERCENTILE()`](/influxdb/v1.2/query_language/functions/#percentile).

Examples:

The following examples focus on the field `water_level` in `santa_monica`
between `2015-08-18T00:00:00Z` and `2015-08-18T00:36:00Z`:
```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time			                water_level
2015-08-18T00:00:00Z	  2.064
2015-08-18T00:06:00Z	  2.116
2015-08-18T00:12:00Z	  2.028
2015-08-18T00:18:00Z	  2.126
2015-08-18T00:24:00Z	  2.041
2015-08-18T00:30:00Z	  2.051
2015-08-18T00:36:00Z	  2.067
```

* Calculate the moving average across every 2 field values:

```
> SELECT MOVING_AVERAGE("water_level",2) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time			                moving_average
2015-08-18T00:06:00Z	  2.09
2015-08-18T00:12:00Z	  2.072
2015-08-18T00:18:00Z	  2.077
2015-08-18T00:24:00Z	  2.0835
2015-08-18T00:30:00Z	  2.0460000000000003
2015-08-18T00:36:00Z	  2.059
```

The first value in the `moving_average` column is the average of `2.064` and
`2.116`, the second value in the `moving_average` column is the average of
`2.116` and `2.028`.

* Select the minimum `water_level` at 12 minute intervals and calculate the
moving average across every 2 field values:

```
> SELECT MOVING_AVERAGE(MIN("water_level"),2) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:36:00Z' GROUP BY time(12m)
name: h2o_feet
--------------
time			                moving_average
2015-08-18T00:12:00Z	  2.0460000000000003
2015-08-18T00:24:00Z	  2.0345000000000004
2015-08-18T00:36:00Z	  2.0540000000000003
```

To get those results, InfluxDB first selects the `MIN()` `water_level` for every
12 minute interval:
```
name: h2o_feet
--------------
time			                min
2015-08-18T00:00:00Z	  2.064
2015-08-18T00:12:00Z	  2.028
2015-08-18T00:24:00Z	  2.041
2015-08-18T00:36:00Z	  2.067
```

It then uses those values to calculate the moving average across every 2 field
values; the first result in the `moving_average` column the average of `2.064`
and `2.028`, and the second result is the average of `2.028` and `2.041`.

## NON_NEGATIVE_DERIVATIVE()
Returns the non-negative rate of change for the values in a single [field](/influxdb/v1.2/concepts/glossary/#field) in a [series](/influxdb/v1.2/concepts/glossary/#series).
InfluxDB calculates the difference between chronological field values and converts those results into the rate of change per `unit`.
The `unit` argument is optional and, if not specified, defaults to one second (`1s`).

The basic `NON_NEGATIVE_DERIVATIVE()` query:
```
SELECT NON_NEGATIVE_DERIVATIVE(<field_key>, [<unit>]) FROM <measurement_name> [WHERE <stuff>]
```

Valid time specifications for `unit` are:  
`u` or `µ`&emsp;microseconds  
`ms`&nbsp;&nbsp;&emsp;&emsp;&nbsp;&nbsp;&nbsp;milliseconds  
`s`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;seconds  
`m`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;minutes  
`h`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;hours  
`d`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;days  
`w`&nbsp;&nbsp;&emsp;&emsp;&emsp;&nbsp;weeks

`NON_NEGATIVE_DERIVATIVE()` also works with a nested function coupled with a `GROUP BY time()` clause.
For queries that include those options, InfluxDB first performs the aggregation, selection, or transformation across the time interval specified in the `GROUP BY time()` clause.
It then calculates the difference between chronological field values and
converts those results into the rate of change per `unit`.
The `unit` argument is optional and, if not specified, defaults to the same
interval as the `GROUP BY time()` interval.

The `NON_NEGATIVE_DERIVATIVE()` query with an aggregation function and `GROUP BY time()` clause:
```
SELECT NON_NEGATIVE_DERIVATIVE(AGGREGATION_FUNCTION(<field_key>),[<unit>]) FROM <measurement_name> WHERE <stuff> GROUP BY time(<aggregation_interval>)
```

See [`DERIVATIVE()`](/influxdb/v1.2/query_language/functions/#derivative) for example queries.
All query results are the same for `DERIVATIVE()` and `NON_NEGATIVE_DERIVATIVE` except that `NON_NEGATIVE_DERIVATIVE()` returns only the positive values.

## Include multiple functions in a single query
Separate multiple functions in one query with a `,`.

Calculate the [minimum](/influxdb/v1.2/query_language/functions/#min) `water_level` and the [maximum](/influxdb/v1.2/query_language/functions/#max) `water_level` with a single query:
```
> SELECT MIN("water_level"), MAX("water_level") FROM "h2o_feet"
name: h2o_feet
--------------
time			               min	   max
1970-01-01T00:00:00Z	 -0.61	 9.964
```

## Change the value reported for intervals with no data with `fill()`
By default, queries with an InfluxQL function report `null` values for intervals with no data.
Append `fill()` to the end of your query to alter that value.
For a complete discussion of `fill()`, see [Data Exploration](/influxdb/v1.2/query_language/data_exploration/#group-by-time-intervals-and-fill).

> **Note:** `fill()` works differently with `COUNT()`.
See [the documentation on `COUNT()`](/influxdb/v1.2/query_language/functions/#count-and-controlling-the-values-reported-for-intervals-with-no-data) for a function-specific use of `fill()`.

## Rename the output column's title with `AS`

By default, queries that include a function output a column that has the same name as that function.
If you'd like a different column name change it with an `AS` clause.

Before:
```
> SELECT MEAN("water_level") FROM "h2o_feet"
name: h2o_feet
--------------
time			               mean
1970-01-01T00:00:00Z	 4.442107025822521
```

After:
```
> SELECT MEAN("water_level") AS "dream_name" FROM "h2o_feet"
name: h2o_feet
--------------
time			               dream_name
1970-01-01T00:00:00Z	 4.442107025822521
```

# Predictors

## HOLT_WINTERS()
Returns N number of predicted values for a single
[field](/influxdb/v1.2/concepts/glossary/#field) using the
[Holt-Winters](https://www.otexts.org/fpp/7/5) seasonal method.
The seasonal adjustment of the predicted values is optional.
The field must be of type int64 or float64.

Use `HOLT_WINTERS()` to:

* Predict when data values will cross a given threshold.
* Compare predicted values with actual values to detect anomalies in your data.

Returns only the predicted values:
```
SELECT HOLT_WINTERS(FUNCTION(<field_key>),<N>,<S>) FROM <measurement_name> WHERE <stuff> GROUP BY time(<interval>)[,<stuff>]
```

Returns the fitted values and the predicted values:
```
SELECT HOLT_WINTERS_WITH_FIT(FUNCTION(<field_key>),<N>,<S>) FROM <measurement_name> WHERE <stuff> GROUP BY time(<interval>)[,<stuff>]
```

Syntax explanation:

`N` is the number of predicted values.
Those values occur at the same interval as the `GROUP BY time()` interval.
If your `GROUP BY time()` interval is `6m` and `N` is `3` you'll
receive three predicted values that are each six minutes apart.

`S` is the seasonal pattern parameter.
The parameter delimits the length of a seasonal pattern according to the
`GROUP BY time()` interval.
If your `GROUP BY time()` interval is `2m` and `S` is `3`, then the
seasonal pattern occurs every six minutes, that is, every three data points.
If you do not want to seasonally adjust your predicted values, set `S` to `0`
or `1.`

Notice that `HOLT_WINTERS()` requires the use of another InfluxQL function and a
`GROUP BY time()` clause.
That ensures that the function receives data that occur at consistent time
intervals.

> **Note:** In some cases, users may receive fewer predicted points than
requested by the `N` parameter.
That behavior occurs when the math becomes unstable and cannot forecast more
points.
It implies that either `HOLT_WINTERS()` is not suited for the dataset or that
the seasonal adjustment parameter is invalid and is confusing the algorithm.

**Example:**

In the following example we'll apply `HOLT_WINTERS()` to the data in the
`NOAA_water_database`, and we'll show the results using
[Chronograf](https://docs.influxdata.com/chronograf/v1.2/) visualizations.

**Raw Data**

Our query focuses on the raw `water_level` data in `santa_monica` between August
22, 2015 at 22:12 and August 28, 2015 at 03:00:
```
SELECT "water_level" FROM "h2o_feet" WHERE "location"='santa_monica' AND time >= '2015-08-22 22:12:00' AND time <= '2015-08-28 03:00:00'
```

![Raw Data](/img/influxdb/hw_raw_data.png)

**Step 1: Create a Query to Match the Trends of the Raw Data**

We start by writing a `GROUP BY time()` query to match the general trends of the
raw `water_level` data.
We could do this with almost any InfluxQL function, but here we use
[`FIRST()`](#first).

Focusing on the `GROUP BY time()` arguments in the query below, the first
argument (`379m`) matches the length of time that occurs between each peak and
trough of the `water_level` data.
The second argument (`348m`) is the
[offset interval](/influxdb/v1.2/query_language/data_exploration/#advanced-group-by-time-syntax).
The offset interval alters InfluxDB's default `GROUP BY time()` boundaries to
match the time range of the raw data.

The green line shows the results of the query:
```
SELECT first("water_level") FROM "h2o_feet" WHERE "location"='santa_monica' and time >= '2015-08-22 22:12:00' and time <= '2015-08-28 03:00:00' GROUP BY time(379m,348m)
```

![First step](/img/influxdb/hw_first_step.png)

**Step 2: Determine the Seasonal Pattern**

Now that we have a query that matches the trends of the raw `water_level` data
it's time to determine the seasonal pattern in the data.
We'll use this information when we implement the `HOLT_WINTERS()` function in
the next step.

Focusing on the green line in the chart below, notice the pattern that repeats
about every 25 hours and 15 minutes.
That's four data points per season, so `4` is our seasonal pattern argument.

![Second step](/img/influxdb/hw_second_step.png)

**Step 3: Include the HOLT_WINTERS() function**

Now we add the `HOLT_WINTERS()` function to our query.
Here, we'll use `HOLT_WINTERS_WITH_FIT()` so that the query results show both
the fitted values and the predicted values.

Focusing on the `HOLT_WINTERS_WITH_FIT()` arguments in the query below, the
first argument (`10`) tells the function to predict 10 data points.
Each point will be `379m` apart, the same interval as the first argument in the
`GROUP BY time()` clause.

The second argument (`4`) is the seasonal pattern that we determined in the
previous step.

```
SELECT holt_winters_with_fit(first(water_level),10,4) FROM h2o_feet where location='santa_monica' and time >= '2015-08-22 22:12:00' and time <= '2015-08-28 03:00:00' group by time(379m,348m)
```

![Third step](/img/influxdb/hw_third_step.png)

And that's it!
We've successfully predicted water levels in Santa Monica between August 28,
2015 at 04:32 and August 28, 2015 at 13:23.

# Other
### Sample Data
The data used in this document are available for download on the [Sample Data](/influxdb/v1.2/query_language/data_download/) page.
