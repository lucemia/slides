## BigQuery & MapReduce

Presented by David Chen @ GliaCloud
![GliaCloud](https://www.gliacloud.com/static/icons/logo_light.png)

GDE


## Use Case
Ad Tech Company

User behavior analysis based on million of logs


## What is BigQuery

Interactive data analysis tool for large datasets designed by Google

![](http://image.slidesharecdn.com/ic1ejbksmuxh9rnerx2q-signature-d1261ab99e3e3e10e8adde74fac71193dd38bcbb794483b2af11cab8d0e2057a-poli-150304233953-conversion-gate01/95/google-for-1600-kpi-fluentd-google-big-query-6-638.jpg?cb=1425857309)


## Why BigQuery

1.  Tools designed for Big Data
2.  Easy to Use
3.  Fast and Affordable



## Tools designed for Big Data

Based on Dremel, Columnar Storage & multi-level execution trees. The query is processed by thousands of servers in a multi-level execution tree structure.


![](http://image.slidesharecdn.com/jordantiganibigquerybigdataspain2012conference-121128025249-phpapp02/95/crunching-data-with-google-bigquery-jordan-tigani-at-big-data-spain-2012-15-1024.jpg?cb=1368666852)


## Simple Query

```
select
    state, count(*) count_babies
from [publicdata:samples.natality]
where
    year >= 1980 and year < 1990
group by state
order by count_babies DESC
limit 10
```


## BigQuery

![](http://image.slidesharecdn.com/jordantiganibigquerybigdataspain2012conference-121128025249-phpapp02/95/crunching-data-with-google-bigquery-jordan-tigani-at-big-data-spain-2012-17-1024.jpg?cb=1368666852)


## Columnar Storage

![](http://1.bp.blogspot.com/-QgaSLhmy098/UFS9t5Js6uI/AAAAAAAAAM4/fvenl98H6gE/s640/columnsvsrows.png)


Query optimizier database:
* uniform type, better compression rate
* only access required data
* reduce disk I/O



## Easy to Use


### no deployment and (almost) no cost while you don't need it
https://bigquery.cloud.google.com/welcome?pli=1


### use SQL
It did supports `JOIN`

    SELECT Year, Actor1Name, Actor2Name, Count FROM (
    SELECT Actor1Name, Actor2Name, Year, COUNT(*) Count,
    RANK() OVER(PARTITION BY YEAR ORDER BY Count DESC) rank
    FROM
    (
        SELECT Actor1Name, Actor2Name,  Year
        FROM [gdelt-bq:full.events]
        WHERE Actor1Name < Actor2Name
            and Actor1CountryCode != '' and Actor2CountryCode != ''
            and Actor1CountryCode!=Actor2CountryCode
    ),
    (
        SELECT Actor2Name Actor1Name, Actor1Name Actor2Name, Year
        FROM [gdelt-bq:full.events] WHERE Actor1Name > Actor2Name
        and Actor1CountryCode != '' and Actor2CountryCode != ''
        and Actor1CountryCode!=Actor2CountryCode),
    WHERE Actor1Name IS NOT null
    AND Actor2Name IS NOT null
    GROUP EACH BY 1, 2, 3
    HAVING Count > 100
    )
    WHERE rank=1
    ORDER BY Year



## Fast and Affordable


### Fast

for 1.4 TB data

Type | speed
------|------
Hadoop with Hive    |1491 sec
Amazon Redshit  |155 sec
Google BigQuery |1.8 sec

Ref: http://www.slideshare.net/DharmeshVaya/exploring-bigdata-with-google-bigquery


### Affordable

Type | Price
-----|-----
Storage |  $0.02 per GB / month
Processing | $5 per TB (first 1TB free)


## Interesting Dataset

Data | Size
------------ | -------------
Samples from US weather stations since 1929 | 115M
Measurement data of broadband connection performance | 240B
Birth information for the United States | 68M
Word index for works of Shakespeare | 164K
Revision information for Wikipedia articles | 314M
NY Taxis Log | 173M
more: https://www.reddit.com/r/bigquery/wiki/datasets


## Demo: Use BigQuery, with Google Sheet
link to google sheet template

Install *OWOX BI BigQuery Reports* plugin


## BigQuery UDF
Write Javascript with BigQuery
https://cloud.google.com/bigquery/user-defined-functions

    // UDF definition
    function urlDecode(row, emit) {
      emit({title: decodeHelper(row.title),
            requests: row.num_requests});
    }

    // Helper function with error handling
    function decodeHelper(s) {
      try {
        return decodeURI(s);
      } catch (ex) {
        return s;
      }
    }

    SELECT requests, title
    FROM
      urlDecode(
        SELECT
          title, sum(requests) AS num_requests
        FROM
          [fh-bigquery:wikipedia.pagecounts_201504]
        WHERE language = 'fr'
        GROUP EACH BY title
      )
    WHERE title LIKE '%ç%'
    ORDER BY requests DESC
    LIMIT 100



## MapReduce
There is two ways:

1. BigQuery Connector (Hadoop and Spark)
2. MapReduce on BigQuery vis UDF (trick)


## Review Mapreduce

![enter image description here](http://blog.trifork.com//wp-content/uploads/2009/08/MapReduceWordCountOverview1.png)


## MapReduce
![](http://image.slidesharecdn.com/jordantiganibigquerybigdataspain2012conference-121128025249-phpapp02/95/crunching-data-with-google-bigquery-jordan-tigani-at-big-data-spain-2012-13-1024.jpg?cb=1368666852)


## Dremel (BigQuery) vs MapReduce

* MapReduce
  * Flexible batch processing
  * High overall throughput
  * High Latency
* Dremel
  * Optimized for interactive SQL queries
  * Very Low Latency


## Map

    // UDF
    function mapper(v1, v2){
        return {"key":key, "value": value}
    }

    select key, value
        from mapper(
            select col1, col2, ....
                from [table]
        )


## MapReduce

    // UDF
    function mapper(v1, v2){
        return {"key":key, "value": value}
    }
    function reducer(key, values){
        return {"result": result}
    }

    select key, result
        from reducer(
            select key, nest(value) as result
            from mapper(
                 select col1, col2, ...
                     from [table]
           )
           group by key
       )


## Example: Word count

    function mapper(row, emit) {
      if(row.comment) {
      keywords = row.comment.split(' ');
      for(var i=0; i<keywords.length; i++) {
        emit({keyword: keywords[i], count: 1});
      }
      }
    }

    function reducer(row, emit) {
      var total = 0;
      for(var i=0;i<row.count.length; i++) {
        total += row.count[i];
      }
      emit({keyword: row.keyword, total: total});
    }

    bigquery.defineFunction(
      'mapper',
      ['comment'],
      [{'name': 'keyword', 'type': 'string'},
      {'name': 'count', 'type': 'int'}],
      mapper
    );

    bigquery.defineFunction(
      'reducer',
      ['keyword', 'count'],
      [{'name': 'keyword', 'type': 'string'},
      {'name': 'total', 'type': 'int'}],
      reducer
    )

    select keyword, total
        from reducer(
            select keyword, nest(count) as count
                from mapper(
                    select Actor1Geo_FullName as comment
                        from [gdelt-bq:gdeltv2.events]
                )
                group by keyword
        )



## Great for education


## Reference
http://www.slideshare.net/BigDataSpain/jordan-tigani-big-query-bigdata-spain-2012-conference-15382442

