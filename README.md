# whosonfirst-external-overture-venue-us

Who's On First ancestry data (parent ID and hierarchy) for Overture venues in United States.

## Description

This repository contains CSV files mapping individual records in the [Overture Places](https://docs.overturemaps.org/guides/places/) dataset, located in the United States, to their Who's On First "parent" and "ancestor" (hierarchy) records.

This data was compiled using the [whosonfirst/go-whosonfirst-external](https://github.com/whosonfirst/go-whosonfirst-external?tab=readme-ov-file#assign-ancestors) package to "reverse geocode" each record using the [whosonfirst/go-whosonfirst-spatial-pmtiles](https://github.com/whosonfirst/go-whosonfirst-spatial-pmtiles) package.

Data is encoded as CSV rows with the following headers:

| Header | Notes |
| --- | --- |
| external:geometry | The WKT-encoded geometry for the record. |
| external:id | The unique ID assigned to the record (by Overture) |
| external:namespace | "ovtr" |
| geohash | The geohash of the centroid associated with the external geometry |
| wof:country | The Who's On First country that the external record belongs to |
| wof:hierarchies | The Who's On First hierarchies associated with the external record, derived by doing a point-in-polygon lookup against the external geometry. |
| wof:parent_id | The Who's On First parent ID associated with the external record, derived by doing a point-in-polygon lookup against the external geometry. |

For example:

```
external:geometry,external:id,external:namespace,geohash,wof:country,wof:hierarchies,wof:parent_id
POINT(-64.7255 18.3452278),08f4ce8a8c8666d403b8b8acc948a90b,ovtr,hkmpc3nd,US,"[{""continent_id"":102191575,""country_id"":-1,""dependency_id"":85632169,""empire_id"":136253057,""locality_id"":101734681,""region_id"":85680575}]",101734681
POINT(-64.7187201 18.3434015),08f4ce8aab7824a90307f395ff7b8932,ovtr,hkmpc6m6,US,"[{""continent_id"":102191575,""country_id"":-1,""dependency_id"":85632169,""empire_id"":136253057,""locality_id"":101734681,""region_id"":85680575}]",101734681
POINT(-64.715161 18.3427753),08f4ce8aab62ac720359192c8947fca1,ovtr,hkmpc6uy,US,"[{""continent_id"":102191575,""country_id"":-1,""dependency_id"":85632169,""empire_id"":136253057,""locality_id"":101734681,""region_id"":85680575}]",101734681
```

Do these records really need to store the geometry associated with the Overture ID since they are already included in the Overture exports? Maybe not. Do these records really need to each store their complete Who's On First hierachies rather than referencing a separate file mapping parent IDs to hierarchies? Maybe. Do these files really need to store a geohash? Maybe not. Should there be a "belongs to" column which is the union of all the possible values to make it easier to determine if a venue is contained by a Who's On First ID (easier than querying multiple hierarchy dictionaries)? Maybe.

All of these are valid questions since their inclusion has a meaningful impact on the size of the CSV files and this repository. These details have not been finalized.

## File structure

All data are stored as bzip2-compressed CSV files in the `data` directory using the following conventions:

```
+ data
  + {WHOSONFIRST_REGION_ID}
    - us-{WHOSONFIRST_REGION_ID}-{WHOSONFIRST_LOCALITY_ID}.csv.bz2
    - us-{WHOSONFIRST_REGION_ID}-{WHOSONFIRST_LOCALITY_ID}.csv.bz2
```

In the event that either `{WHOSONFIRST_REGION_ID}` or `{WHOSONFIRST_LOCALITY_ID}` are unknown they will be replaced by "xx". For example:

```
+ data
  + {WHOSONFIRST_REGION_ID}
    - us-{WHOSONFIRST_REGION_ID}-xx.csv.bz2
  + xx
    - us-xx-xx.csv.bz2
    - us-xx-{WHOSONFIRST_LOCALITY_ID}.csv.bz2
```

### Notes

* While it may seem counter-intuitive to be able to know a locality ID but not its region ID this happens and reflects data that needs to be corrected in the Who's On First administrative dataset. Life is complicated that way.

* The bzip2-compressed CSV files in the `data` directory are tracked and stored using [git lfs](https://git-lfs.com/).

## DuckDB

These CSV files are meant to be "useable" from [DuckDB](https://duckdb.org/docs/data/csv/overview.html) or other similar database systems.

```
D DESCRIBE(SELECT * FROM read_csv('us-85688493-85976497.csv'));
┌────────────────────┬─────────────┬─────────┬─────────┬─────────┬─────────┐
│    column_name     │ column_type │  null   │   key   │ default │  extra  │
│      varchar       │   varchar   │ varchar │ varchar │ varchar │ varchar │
├────────────────────┼─────────────┼─────────┼─────────┼─────────┼─────────┤
│ external:geometry  │ VARCHAR     │ YES     │         │         │         │
│ external:id        │ VARCHAR     │ YES     │         │         │         │
│ external:namespace │ VARCHAR     │ YES     │         │         │         │
│ geohash            │ VARCHAR     │ YES     │         │         │         │
│ wof:country        │ VARCHAR     │ YES     │         │         │         │
│ wof:hierarchies    │ VARCHAR     │ YES     │         │         │         │
│ wof:parent_id      │ VARCHAR     │ YES     │         │         │         │
└────────────────────┴─────────────┴─────────┴─────────┴─────────┴─────────┘
```

### Notes

* Unfortunately, DuckDB [does not support reading bz2-compressed CSV files yet](https://github.com/duckdb/duckdb/discussions/12232) which means you will need to decompress the files in this repository before using them. This is not ideal but because the uncompressed CSV data is so big it is a recognized "trade-off".

* Why does DuckDB think that `wof:parent_id` is a "VARCHAR" when [the same code](https://github.com/whosonfirst/go-whosonfirst-external/tree/main/app/ancestors/sort) used to generate these CSV files [for Foursquare data](https://github.com/whosonfirst-data/whosonfirst-external-foursquare-venue-us?tab=readme-ov-file#duckdb) yields a "BIGINT"? I have no idea. Any suggestions, pointers or feedback would be welcome.

### Examples

Basic SQL-like querying:

```
D SELECT "external:id"  FROM read_csv(['us-85688543-1729435243.csv','us-85688671-85937387.csv']) WHERE "wof:parent_id" = '85937387' LIMIT 10;
┌──────────────────────────────────┐
│           external:id            │
│             varchar              │
├──────────────────────────────────┤
│ 08f5d1009ad1c68c038c5c70a4150b8e │
│ 08f5d1009e116406031945cc45af653b │
│ 08f5d1009eb85a490364ded27fee085a │
│ 08f5d1009e99920c03e2f349af8c241b │
│ 08f5d1009e8eb3000363bdd0b5a6e865 │
│ 08f5d1009e8eb27403a5a4525cfb6611 │
│ 08f5d1009e8eb3b603e9136a3762619a │
│ 08f5d1009e8ebb9903fcf28867955aac │
│ 08f5d1009e8eb3b60301aaf4086c1452 │
│ 08f5d1009ebb2b95037dd902f8f91469 │
├──────────────────────────────────┤
│             10 rows              │
└──────────────────────────────────┘
```

Note: There is a "known unknown" with the `wof:hierarchy` data in a DuckDB context. Namely that one or more rows contains "bunk" data:

```
D SELECT "external:id"  FROM read_csv(['us-85688543-1729435243.csv','us-85688671-85937387.csv']) WHERE JSON("wof:hierarchies")[0]."locality_id" = '85937387' LIMIT 10;
Invalid Input Error: Malformed JSON at byte 0 of input: unexpected character.  Input: wof:hierarchies
```

For some as-yet-unknown definition of "bunk" because although the previous query fails, this one works just fine:

```
D SELECT JSON("wof:hierarchies") FROM read_csv(['us-85688543-1729435243.csv','us-85688671-85937387.csv']) WHERE "wof:parent_id" = '85937387' LIMIT 10;
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                              "json"("wof:hierarchies")                                               │
│                                                         json                                                         │
├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
│ [{"continent_id":102191575,"country_id":85633793,"county_id":102085433,"locality_id":85937387,"region_id":85688671}] │
├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                       10 rows                                                        │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

_Computers, amirite?_

So that needs to be figured out but in the meantime the data are meant to be something which can be used in conjunction with the source Overture data. For example here are 10 Overture places that are in the locality of [Puako](https://spelunker.whosonfirst.org/id/85937387), in Hawaii:

```
D SELECT w."external:id", o.names.primary  FROM read_csv(['us-85688543-1729435243.csv','us-85688671-85937387.csv']) w, read_parquet('/usr/local/data/overture/parquet/*.parquet') o  WHERE o.id = w."external:id" AND w."wof:parent_id" = '85937387' LIMIT 10;
┌──────────────────────────────────┬──────────────────────────────────────────┐
│           external:id            │                 primary                  │
│             varchar              │                 varchar                  │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ 08f5d1009ad1c68c038c5c70a4150b8e │ Rocks In Stock                           │
│ 08f5d1009e116406031945cc45af653b │ Kiholo-Puako Trail                       │
│ 08f5d1009eb85a490364ded27fee085a │ Puuanahulu Park                          │
│ 08f5d1009e99920c03e2f349af8c241b │ Anaehoomalu Bay                          │
│ 08f5d1009e8eb3000363bdd0b5a6e865 │ Waikoloa Beach GC - Beach Course         │
│ 08f5d1009e8eb27403a5a4525cfb6611 │ Lava Lava Beach Club                     │
│ 08f5d1009e8eb3b603e9136a3762619a │ Lava Lava Resturaunt At Anaehoomalu      │
│ 08f5d1009e8ebb9903fcf28867955aac │ Beachside at Lava Lava                   │
│ 08f5d1009e8eb3b60301aaf4086c1452 │ Island Lava Java                         │
│ 08f5d1009ebb2b95037dd902f8f91469 │ The Full Body Elixir with Calley O’Neill │
├──────────────────────────────────┴──────────────────────────────────────────┤
│ 10 rows                                                           2 columns │
└─────────────────────────────────────────────────────────────────────────────┘
```

## See also

* https://docs.overturemaps.org/guides/places/
* https://github.com/whosonfirst/go-whosonfirst-external