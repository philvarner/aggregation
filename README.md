# Aggregation Extension Specification (API)

- **Title:** Aggregation
- **Identifier:** <https://stac-extensions.github.io/aggregation/v1.0.0/api.yml>
- **Field Name Prefix:** agg
- **Scope:** TBD
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @philvarner

This document explains the Aggregation Extension to the [SpatioTemporal Asset Catalog API](https://github.com/radiantearth/stac-api-spec) (STAC API) specification.
The purpose of the Aggregation Extension is to provide an endpoint similar to the Search endpoint (`/search`), but which will provide aggregated information on matching Items rather than the Items themselves. This is useful when a dataset is very large, and it is infeasible to example all results.  This is particularly useful in data exploration, whereby a user of the data can change queries to see the "shape" of the results for a given query. 

This is highly influenced by the Elasticsearch aggregation endpoint, but with a more regular structure for responses.

- Examples:
  - [Item example](examples/item.json): Shows the basic usage of the extension in a STAC Item
  - [Collection example](examples/collection.json): Shows the basic usage of the extension in a STAC Collection
- [OpenAPI](openapi/api.yml)
- [Changelog](./CHANGELOG.md)

## STAC Endpoints

| Endpoint     | Returns                                                        | Description |
| ------------ | -------------------------------------------------------------- | ----------- |
| `/aggregate` | AggregationCollection | Retrieves an aggregation of the group of Items matching the provided predicates |
| `/aggregateables` | TBD | TBD |

The `/aggregate` endpoint behaves similarly to the `/search` endpoint, but instead of returning an ItemCollection of Items, it instead returns aggregated information over the same matching Items in the form of an **AggregationCollection** of **Aggregation** entities.

aggregateables tbd
## Relation types

The following types should be used as applicable `rel` types in the
[Link Object](https://github.com/radiantearth/stac-spec/tree/master/item-spec/item-spec.md#link-object).

| Type                | Endpoint | Description |
| ------------------- | -- | ----------- |
| aggregate      | `/aggregate` | Aggregate endpoint. |


## Filter Parameters and Fields

The filters for `/aggregate` are the same as those for `/search` that are semantically meaningful (e.g., limit has no meaning when doing aggregations). These filters are passed as query string parameters or JSON 
entity fields.  For filters that represent a set of values, query parameters should use comma-separated 
string values and JSON entity attributes should use JSON Arrays. 

| Parameter    | Type             | Description |
| -----------  | ---------------- | ----------- |
| datetime     | string           | Single date+time, or a range ('/' seperator), formatted to [RFC 3339, section 5.6](https://tools.ietf.org/html/rfc3339#section-5.6). Use double dots `..` for open date ranges. |
| bbox         | \[number]        | Requested bounding box.  Represented using either 2D or 3D geometries. The length of the array must be 2*n where n is the number of dimensions. The array contains all axes of the southwesterly most extent followed by all axes of the northeasterly most extent specified in Longitude/Latitude or Longitude/Latitude/Elevation based on [WGS 84](http://www.opengis.net/def/crs/OGC/1.3/CRS84). When using 3D geometries, the elevation of the southwesterly most extent is the minimum elevation in meters and the elevation of the northeasterly most extent is the maximum. |
| intersects   | GeoJSON Geometry | Searches items by performing intersection between their geometry and provided GeoJSON geometry.  All GeoJSON geometry types must be supported. |
| ids          | \[string]        | Array of Item ids to return. All other filter parameters that further restrict the number of search results (except `next` and `limit`) are ignored |
| collections  | \[string]        | Array of Collection IDs to include in the search for items. Only Items in one of the provided Collections will be searched |
| aggregations | \[string]        | A list of aggregations to compute and return |

Only one of either **intersects** or **bbox** should be specified.  If both are specified, a 400 Bad Request response should be returned. 

**aggregations**: There are no named aggregations that must be implemented. All aggregations which are available should be advertised in the root `rel="aggregate"` link. 

This is a list of recommended aggregations to implement:
* count (Single Value of integer)
* datetime_min (Single Value of datetime)
* datetime_max (Single Value of datetime)
* collection (Term Count)
* cloud_cover (Discrete Range)
* datetime_auto (Datetime Range, automatic interval detection) -- detect a reasonable interval based on the datetime range and distribution of data. Implementation specific.
* datetime_yearly (Datetime Range, interval=year)
* datetime_quarterly (Datetime Range, interval=quarter)
* datetime_monthly (Datetime Range, interval=month)
* datetime_weekly (Datetime Range, interval=week)
* datetime_daily (Datetime Range, interval=day)
* datetime_hourly (Datetime Range, interval=hour)
* datetime_minutes (Datetime Range, interval=minute)
* datetime_seconds (Datetime Range, interval=second)

## AggregationCollection fields

This object describes a STAC AggregationCollection, which is the analog of an ItemCollection for the `/aggregate` operation.

| Field Name      | Type           | Description |
| --------------- | -------------- | ----------- |
| stac_version    | string         | **REQUIRED** The STAC version the AggregationCollection implements. |
| type            | string         | **REQUIRED** Always "AggregationCollection". |
| aggregations    | \[Aggregation] | **REQUIRED** A possibly-empty array of Aggregations. |

**stac_version**: In general, STAC versions can be mixed, but please keep the [recommended best practices](../best-practices.md#mixing-stac-versions) in mind.

## Aggregation fields

| Field Name      | Type           | Description |
| --------------- | -------------- | ----------- |
| key             | string         | **REQUIRED** The unique indentifier of the aggregation. |         
| buckets         | \[Bucket]      | If the aggregation bucketizes Items, they are defined here. |         
| overflow        | integer        | The count of Items that were not categorized into any of the buckets defined by the `buckets` field |         
| interval        | string         | \["year", "quarter", "month", "week", "day", "hour", "minute", "second"] |   
| value           | string         | For a Single Value aggregation, the string representation of the result value. |
| value_as_type   | string or number or datetime | For a Single Value aggregation, a JSON-type represenation of the result value. |     

One of either **buckets** or **value** is required.

**key** An identifier for the aggregation result. Should be identical to the value passed to the `aggregations` query parameter.

**buckets** If the aggregation is a Term Count, Datetime Range, or Discrete Range, these are the "buckets" into which each matching Item is categorized.

**overflow** Some implemenation data stores may have limitations on the aggregation queries that can be performed on them. For example, Elasticsearch limits the number of buckets for a query to 10,000 for performance reasons.  Overflow indicates that there were Items matched by the query that are not accounted for in the count of any of the response buckets. 

**interval** Aggregations over datetime typed values that return a Datetime Range have a slightly different format than Discrete Range. For these, only the start datetime for the bucket is set to the `key` field. The interval determines how much time from that starting datetime the bucket represents.

**value** For Single Value aggregations, this is the string value of the result. If the type of the value being aggregated over is a datetime, this is an RFC 3339 datetime, e.g., "2020-08-12T19:06:09Z". 

**value_as_type** For Single Value aggregations, this is a representation of the result value as the equivalent JSON type. TBD: what about datetimes? 

## Bucket fields

| Field Name      | Type           | Aggregation Types | Description |
| --------------- | -------------- | ----------------- | ----------- |
| key             | string         | all               | |
| key_as_type     | ?              | all               | |
| value           | string         | all               | |
| value_as_type   | ?              | all               | |
| from            | ?              | all               | |
| to              | ?              | all               | |

## Aggregation Types

### Single Value Aggregation

effectively a single Term Count Bucket lifted up one level

**todo** (diff for String, Numeric, and Datetime)

Example:
    {   
        "stac_version": "1.0.0",
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "datetime_min",
                "value": "2000-02-16T00:00:00.000Z",
                "value_as_type": 1.506592E+11
            }
        ]
    }

### Term Count Aggregation

- enumeration count multi bucket one per unique value

Example:

```json
    {   
        "stac_version": "1.0.0",
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "collections",
                "buckets": [
                    { 
                        "key": "sentinel2_l1c",
                        "value": "12649072",
                        "value_as_type": 12649072
                    },
                    {
                        "key": "landsat8_l1tp",
                        "value" : "1071997",
                        "value_as_type": 1071997
                    }
                ],
                "overflow": 23414
            }
        ]
    }
```

### Discrete Range Aggregation

Fields:
* key (string)
* key_as_type ()
* from (optional, missing indicates an open interval) inclusive
* to (optional, missing indicates an open interval) exclusive
* value (integer)

Example:

```json
    {   
        "stac_version": "1.0.0",
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "cloud_cover",
                "buckets": [
                    { 
                        "key": "*-5.0",
                        "to": 5,
                        "value" : "8644819",
                        "value_as_type" : 8644819
                    },
                    {
                        "key": "5.0-10.0",
                        "from": 5,
                        "to": 10,
                        "value" : "5644819",
                        "value_as_type" : 5644819
                    },
                    {
                        "key": "10.0-*",
                        "from": 10,
                        "value" : "7644819",
                        "value_as_type" : 7644819
                    }
                ]
            }
        ]
    }
```

### Datetime Range Aggregation

Fields:
datetimes are RFC 3339 string values

* key (string) 
* key_as_type (datetime in milliseconds?)
* value (integer) (ES: doc_count)

Example:

```json
    {   
        "stac_version": "1.0.0",
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "datetime_yearly",
                "buckets": [
                    { 
                        "key": "2000-01-01T00:00:00.000Z",
                        "key_as_type": 946684800000,
                        "to": 5,
                        "value" : "8644819",
                        "value_as_type" : 8644819
                    },
                    {
                        "key": "2001-01-01T00:00:00.000Z,
                        "key_as_type": 978307200000,
                        "from": 5,
                        "to": 10,
                        "value" : "5644819",
                        "value_as_type" : 5644819
                    },
                    {
                        "key": "2002-01-01T00:00:00.000Z,
                        "key_as_type": 1009843200000,
                        "from": 10,
                        "value" : "7644819",
                        "value_as_type" : 7644819
                    }
                ],
                "interval": "year",
                "overflow": 98373 
            }
        ]
    }
```

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md) Instructions
for running tests are copied here for convenience.

### Running tests

The same checks that run as checks on PR's are part of the repository and can be run locally to verify that changes are valid. 
To run tests locally, you'll need `npm`, which is a standard part of any [node.js installation](https://nodejs.org/en/download/).

First you'll need to install everything with npm once. Just navigate to the root of this repository and on 
your command line run:
```bash
npm install
```

Then to check markdown formatting and test the examples against the JSON schema, you can run:
```bash
npm test
```

This will spit out the same texts that you see online, and you can then go and fix your markdown or examples.

If the tests reveal formatting problems with the examples, you can fix them with:
```bash
npm run format-examples
```