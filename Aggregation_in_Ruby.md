This document provides a number of practical examples that display the
capabilities of the aggregation framework.

The [_Aggregations using the Zip Codes Data Set_](http://docs.mongodb.org/manual/tutorial/aggregation-examples/#aggregations-using-the-zip-code-data-set)
examples uses a publicly available data set of all zipcodes and
populations in the United States. These data are available at:
[zips.json](http://media.mongodb.org/zips.json).


## Requirements

[MongoDB](http://www.mongodb.org/downloads), version 3.2.0 or later.
[Ruby MongoDB Driver](https://github.com/mongodb/mongo-ruby-driver),
version 2.1.2 or later. Ruby, version 2.2.3 or later.

Let’s check if everything is installed.

Use the following command to load *zips.json* data set into mongod
instance:

```sh
mongoimport --drop -d zip -c codes zips.json
```

On the *irb* console run:

```ruby
require 'mongo'

# mute Logger
# Mongo::Logger.level = Logger::INFO # the default is Logger::DEBUG
Mongo::Logger.logger.level = Logger::FATAL

client = Mongo::Client.new(['localhost:27017'], {database: 'zip'})
coll = client[:codes]

coll.count     #=> should return 29353
coll.find.first
```

## Aggregations using the Zip Codes Data Set

Each document in this collection has the following form:

```ruby
{
  "_id": "01001",
  "city": "AGAWAM",
  "loc": [
    -72.622739, 42.070206
  ],
  "pop": 15338,
  "state": "MA"
}
```

In these documents:

* The `_id` field holds the zipcode as a string.
* The `city` field holds the city name.
* The `state` field holds the two letter state abbreviation.
* The `pop` field holds the population.
* The `loc` field holds the location as a `[longitude, latitude]` array.


### States with Populations Over 10 Million

To get all states with a population greater than 10 million, use
the following aggregation pipeline (na konsoli _pry_):

```ruby
ag = coll.aggregate([
  {"$group" => {_id: "$state", total_pop: {"$sum" => "$pop"}}},
  {"$match" => {total_pop: {"$gte" => 10_000_000}}}
])
```
The result:

```ruby
ag.to_a.sort_by { |d| d.to_h["total_pop"] }
[{"_id"=>"OH", "total_pop"=>10846517},
 {"_id"=>"IL", "total_pop"=>11427576},
 {"_id"=>"PA", "total_pop"=>11881643},
 {"_id"=>"FL", "total_pop"=>12686644},
 {"_id"=>"TX", "total_pop"=>16984601},
 {"_id"=>"NY", "total_pop"=>17990402},
 {"_id"=>"CA", "total_pop"=>29754890}]
 ```

The above aggregation pipeline is build from two pipeline operators:
`$group` and `$match`.

The `$group` pipeline operator requires `_id` field where we specify
grouping; remaining fields specify how to generate composite value and
must use one of
[the group aggregation functions](http://docs.mongodb.org/manual/reference/aggregation/#group-operators):
`$addToSet`, `$first`, `$last`, `$max`, `$min`, `$avg`, `$push`, `$sum`.
The `$match` pipeline operator syntax is the same as
the [read operation](http://docs.mongodb.org/manual/core/read-operations/)
query syntax.

The `$group` process reads all documents and for each state it
creates a separate document, for example:

```ruby
{"_id"=>"WA", "total_pop"=>4866692}
```

The `total_pop` field uses the `$sum` aggregation function
to sum the values of all `pop` fields in the source documents.

Documents created by `$group` are piped to the `$match` pipeline
operator. It returns the documents with the value of `total_pop` field
greater than or equal to 10 million.


### Average City Population by State

To get the first three states with the greatest average population
per city, use the following aggregation:

```ruby
ag = coll.aggregate([
  {"$group" => {_id: {state: "$state", city: "$city"}, pop: {"$sum" => "$pop"}}},
  {"$group" => {_id: "$_id.state", avg_city_pop: {"$avg" => "$pop"}}},
  {"$sort" => {avg_city_pop: -1}},
  {"$limit" => 3}
])
```

This aggregate pipeline produces:

```ruby
ag.to_a
[{"_id"=>"DC", "avg_city_pop"=>303450.0},
 {"_id"=>"CA", "avg_city_pop"=>27756.42723880597},
 {"_id"=>"FL", "avg_city_pop"=>27400.958963282937}]
```

The above aggregation pipeline is build from three pipeline operators:
`$group`, `$sort` and `$limit`.

The first `$group` operator creates the following documents:

```ruby
{"_id"=>{"state"=>"WY", "city"=>"Smoot"}, "pop"=>414}
```

Note, that the `$group` operator can’t use nested documents
except the `_id` field.

The second `$group` uses these documents to create the following
documents:

```ruby
{"_id"=>"FL", "avg_city_pop"=>27400.958963282937}
```

These documents are sorted by the `avg_city_pop` field in descending order.
Finally, the `$limit` pipeline operator returns the first 3 documents
from the sorted set.


### Largest and Smallest Cities by State

To get the smallest and largest cities by population for each
state, use the following aggregate pipeline:

```ruby
ag = coll.aggregate([
  {"$group" => {_id: {state: "$state", city: "$city"}, pop: {"$sum" => "$pop"}}},
  {"$sort" => {pop: 1}},
  {"$group" => {
                _id: "$_id.state",
      smallest_city: {"$first" => "$_id.city"},
       smallest_pop: {"$first" => "$pop"},
       biggest_city: { "$last" => "$_id.city"},
        biggest_pop: { "$last" => "$pop"}
    }
  }
])
```

The first `$group` operator creates a new document for every
combination of the `state` and `city` fields from the source
documents. Each document created at this stage has the field `pop`
which is set to the value computed by the `$sum` operator.
It sums the values of the `pop` field in the grouped documents.

The sample document created at this stage looks like:

```ruby
{"_id"=>{"state"=>"AL", "city"=>"Cottondale"}, "pop"=>4727}
```

*Note*: To preserve the values of the `state` and `city` fields
for later use in the pipeline we specify the value of `_id`
as a nested document which contains both values.

The second `$group` operator groups the documents by the value of
`_id.state`.

The sorting order is preserved within grouped documents.
So, `$first` operators return name of the city with
the smallest population and the city population.
The `$last` operators return the city name with the
biggest population and the city population.

The sample document created at this stage looks like:

```ruby
ag.to_a[1..4]
[{"_id"=>"MS",
  "smallest_city"=>"CHUNKY",
  "smallest_pop"=>79,
  "biggest_city"=>"JACKSON",
  "biggest_pop"=>204788},
 {"_id"=>"RI",
  "smallest_city"=>"CLAYVILLE",
  "smallest_pop"=>45,
  "biggest_city"=>"CRANSTON",
  "biggest_pop"=>176404},
 {"_id"=>"IL",
  "smallest_city"=>"ANCONA",
  "smallest_pop"=>38,
  "biggest_city"=>"CHICAGO",
  "biggest_pop"=>2452177},
 {"_id"=>"CT",
  "smallest_city"=>"EAST KILLINGLY",
  "smallest_pop"=>25,
  "biggest_city"=>"BRIDGEPORT",
  "biggest_pop"=>141638}]
```

## Unwinding data in the Name Days Data Set

To run the examples below you need this data set:
[name_days.json](https://raw.github.com/wiki/mongodb/mongo-ruby-driver/data/name_days.json).

Use *mongoimport* to import this data set into MongoDB:

```sh
mongoimport --drop --db test --collection cal name_days.json
```

After import, the collection *cal*  should contain
364 documents in the following format:

```ruby
{
  "_id": ObjectId("564e2894fbe8d6e3ecc81712"),
  "names": [
    "Izydora", "Bazylego", "Grzegorza"
  ],
  "date": {
    "day": 2, "month": 1
  }
}
```

### The 6 most common name days

The following aggregation pipeline computes this:

```ruby
coll = client[:cal] # switch collection

ag = coll.aggregate([
  {"$project" => {names: 1, _id: 0}},
  {"$unwind" => "$names" },
  {"$group" => {_id: "$names", count: {"$sum" => 1}}},
  {"$sort" => {count: -1}},
  {"$limit" => 6}
])
```

The sample document created by the `$project` pipeline operator
looks like:

```ruby
{"names"=>["Izydora", "Bazylego", "Grzegorza"]}
```

The `$unwind` operator creates one document for every member of
*names* array. For example, the above document is unwinded into three
documents:

```ruby
{"names"=>"Izydora"}
{"names"=>"Bazylego"}
{"names"=>"Grzegorza"}
```

These documents are grouped by the `names` field and the documents
in each group are counted by the `$sum` operator.

The sample document created at this stage looks like:

```ruby
{"_id"=>"Jacka", "count"=>4}
```

Finally, the `$sort` operator sorts these documents by the
`count` field in descending order, and  the `$limit` operator
outputs the first 6 documents:

```ruby
pp ag.to_a
[{"_id"=>"Jana", "count"=>21},
 {"_id"=>"Marii", "count"=>16},
 {"_id"=>"Grzegorza", "count"=>9},
 {"_id"=>"Piotra", "count"=>9},
 {"_id"=>"Leona", "count"=>8},
 {"_id"=>"Feliksa", "count"=>8}]
```

### Pivot date ↺ names

We want to pivot the *name_days.json* data set.
Precisely, we want to convert documents from this format:

```ruby
{ "date"=>{"day"=>1, "month"=>1}, "names"=>["Mieszka", "Mieczysława", "Marii"] }
```

into this format:

```ruby
{ "name"=>"Mateusza", "dates"=>[{"day"=>13, "month"=>11}, {"day"=>21, "month"=>9}]}
```

The following aggregation pipeline does the trick:

```ruby
ag = coll.aggregate([
  {"$project" => {_id: 0, date: 1, names: 1}},
  {"$unwind" => "$names"},
  {"$group" => {_id: "$names", dates: {"$addToSet" => "$date"}}},
  {"$project" => {name: "$_id", dates: 1, _id: 0}},
  {"$sort" => {name: 1}}
])
```

The sample document created by the unwinding stage looks like:

```ruby
{"names"=>"Eugeniusza", "date"=>{"day"=>30, "month"=>12}}
```

The `$group` pipeline operator groups these documents by the `names`
field. The `$addToSet` operator returns an array of all unique values
of the `date` field found in the set of grouped documents.
The sample document created at this stage of the pipeline looks like:

```ruby
{"_id"=>"Maksymiliana", "dates"=>[{"day"=>12, "month"=>10}, {"day"=>14, "month"=>8}]}
```

In the last two stages we sort and reshape these documents to the
requested format:

```ruby
pp ag.to_a[100]
{
  "dates"=>[
    {"day"=>29, "month"=>12}, {"day"=>15, "month"=>7}, {"day"=>1, "month"=>3}
  ],
  "name"=>"Dawida"}
```


## Quiz

1\. What is the result of running the empty aggregation pipeline:

```ruby
coll.aggregate([])
```

2\. For the *zipcodes* collection, the aggregation below computes
`248_408_400`. What does this number mean?


```ruby
puts coll.aggregate([ {"$group" => {_id: 0, sum: {"$sum" => "$pop"}}} ])
ag.count # 1
ag.to_a[0] # {"_id"=>0, "sum"=>248_408_400}
```

3\. How many different names are in the *cal* collection?
