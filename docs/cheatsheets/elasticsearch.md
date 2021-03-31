# Elasticsearch Cheatsheet

[![Say Thanks!](https://img.shields.io/badge/Say%20Thanks-%3A%29-green.svg)](https://saythanks.io/to/ruan.ru.bekker@gmail.com) [![made-with-Markdown](https://img.shields.io/badge/Visit%20my-Website-orange.svg)](https://ruan.dev) [![FollowMe](https://img.shields.io/badge/Follow%20Me-@ruanbekker-00ACEE.svg)](https://twitter.com/ruanbekker)

This **elasticsearch cheatsheet** will show you how to deploy a 3 node cluster on docker, and a walk through of all the most frequent commands that I use when working with elasticsearch clusters.

### Pre Requisites

Note: This step requires [docker and docker-compose](https://docs.docker.com/get-docker/). Get the docker-compose:

```sh
wget https://gist.githubusercontent.com/ruanbekker/fafbffe597202951207ca0b21d826c1d/raw/3e857b40f6a6a7576cba073fa4b4adc14ade3be7/docker-compose.yml
```

Deploy the elasticsearch 3 node cluster:

```sh
docker-compose up -d
```

Once all the containers are running you should be able to see 3 nodes using:

```sh
curl -XGET "http://localhost:9200/_cat/nodes?v"
```

For deploying a 3 node elasticsearch cluster on servers, view [this post](https://devconnected.com/how-to-install-an-elasticsearch-cluster-on-ubuntu-18-04/)

### Basic Concepts

Basic concepts of Elasticsearch:

  - a Elasticsearch Cluster is made up of a number of nodes
  - Each Node contains Indexes, where a Index is a Collection of Documents
  - Master nodes are responsible for Cluster related tasks, creating / deleting indices, tracking of nodes, allocate shards to nodes
  - Data nodes are responsible for hosting the actual shards that has the indexed data also handles data related operations like CRUD, search, and aggregations
  - Indices are split into Multiple Shards
  - Shards exists of Primary Shards and Replica Shards
  - A Replica Shard is a Copy of a Primary Shard which is used for HA/Redundancy
  - Shards gets placed on random nodes throughout the cluster
  - A Replica Shard will NEVER be on the same node as the Primary Shard’s associated shard-id.
  - Green Cluster Status is good. 
  - Yellow would essentially mean that one or more replica shards are in a unassigned state. 
  - Red status means that some or all primary shards are unassigned which is really bad.

More information can be retrieved from [their documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html)

### Health 

View the cluster health on a cluster level:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cluster/health?pretty"
```

View the cluster health on a index level:

```sh
curl -XGET "http://127.0.0.1:9200/_cluster/health?level=indices&pretty"
```

Check all indices in yellow status:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cat/indices?v&health=yellow"
```

View recovery process:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cat/recovery?detailed&h=index,stage,time,bytes_percent"
```

### Cluster Level

View the cluster level health:

```sh
curl http://127.0.0.1:9200/_cluster/health?pretty
```

To view all your node info from you es nodes:

```sh
curl http://127.0.0.1:9200/_cat/nodes?v
```

To view your disk usage information:

```sh
curl http://127.0.0.1:9200/_cat/allocation?v
```

To view all your indices:

```sh
curl http://127.0.0.1:9200/_cat/indices?v
```

To view all your shards from your indices:

```sh
curl http://127.0.0.1:9200/_cat/shards?v
```

Use the explain API of shard allocation info:

```sh
curl http://127.0.0.1:9200/_cluster/allocation/explain?pretty
```

To return yes in explain decisions:

```sh
curl 'http://127.0.0.1:9200/_cluster/allocation/explain?include_yes_decisions=true&pretty'
```

View stats on your index or indices:

```sh
curl -s -XGET http://127.0.0.1:9200/logs-2020.09.*/_stats?pretty
```

To view lucene segments in index shards:

```sh
curl -XGET http://127.0.0.1:9200/_cat/segments/logs-2020.09.*?v
```

### View Indices

View all your indices:

```sh
curl -s -XGET http://127.0.0.1:9200/_cat/indices?v
```

View all indices from 2019.05:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cat/indices/*2019.05*?v"
```

View all your indices, sort by size:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cat/indices?v&s=pri.store.size"
```

View all indices, but return only the `index.name` value:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cat/indices?v&h=index"
```

View all indices in red status:

```sh
curl -s -XGET "http://127.0.0.1:9200/_cat/indices?v&health=red"
```

### Create Index

Create a empty Index:

```sh
curl -XPOST -H "Content-Type: application/json" "http://localhost:9200/my-test-index"
```

Create a Index with 5 Primary Shards, 1 Replica Shard and Refresh Interval of 30 seconds:

```sh
curl -XPUT -H "Content-Type: application/json" \
  http://localhost:9200/my-test-index \
  -d '{"settings": {"index": {"number_of_shards":"5","number_of_replicas": 1, "refresh_interval": "30s"}}}'
```

If you want to manually refresh your index to see the data:

```sh
curl -XPOST -H "Content-Type: application/json" "http://localhost:9200/my-test-index/_refresh"
```

### Update Index Settings

View the index settings:

```sh
curl -XGET -H "Content-Type: application/json" "http://127.0.0.1:9200/my-test-index/_settings?pretty"
```

Update the settings, disable refresh for example:

```sh
curl -XPUT -H "Content-Type: application/json" "http://127.0.0.1:9200/my-test-index/_settings" -d '{"index": {"refresh_interval": "-1"}}'
```

Increase the replica shards:

```sh
curl -XPUT -H "Content-Type: application/json" "http://127.0.0.1:9200/my-test-index/_settings" -d '{"index": {"number_of_replicas": "2"}}'
```

Reduce the replica shards:

```sh
curl -XPUT -H "Content-Type: application/json" "http://127.0.0.1:9200/my-test-index/_settings" -d '{"index": {"number_of_replicas": "1"}}'
```

Remember that primary shards can only be set on index creation.

### Index Management

Reduce the number of segments in each shard by merging some of them together:

```sh
curl -s -H 'Content-Type: application/json' -XPOST 'http://127.0.0.1:9200/logs-2020.08.08/_cache/clear'
curl -s -H 'Content-Type: application/json' -XPOST 'http://127.0.0.1:9200/logs-2020.08.08/_forcemerge?max_num_segments=1'
```

To view the task progress:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_cat/tasks?detailed'
```

To get detailed output from the task:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_tasks?actions=*indices:admin/forcemerge&detailed&pretty'
```

### Ingest Data

Ingest a document into our index, write a single document and specify the ID, then we use a `PUT`:

```sh
curl -XPUT -H 'Content-Type: application/json' \
  http://127.0.0.1:9200/my-test-index/_doc/1 -d '
  {"name":"pete", "country":"south africa", "gender": "male", "age": 24}'
```

Ingest another document, but this time we want elasticsearch to generate a document ID for us, therefore we are using `POST`:

```sh
curl -XPOST -H 'Content-Type: application/json' \
  http://127.0.0.1:9200/my-test-index/_doc/ -d '
  {"name": "kevin", "country": "new zealand", "gender": "male", "age": 29}'

curl -XPOST -H 'Content-Type: application/json' \
  http://127.0.0.1:9200/my-test-index/_doc/ -d '
  {"name": "sarah", "country": "ireland", "gender": "female", "age": 32}'
```

### Bulk Ingest Data

We can use the bulk api to ingest data, or documents in `docs.json`:

```json
{ "index": {"_id": "2"}}
{ "name": "michelle", "country": "mexico", "gender": "female", "age": 36 }
{ "index": {"_id": "3"}}
{ "name": "tom", "country": "america", "gender": "male", "age": 30 }
{ "index": {"_id": "4"}}
{ "name": "jamie", "country": "new zealand", "gender": "male", "age": 29 }
{ "index": {"_id": "5"}}
{ "name": "susan", "country": "canada", "gender": "female", "age": 27 }
{ "index": {"_id": "6"}}
{ "name": "frank", "country": "america", "gender": "male", "age": 32 }
{ "index": {"_id": "7"}}
{ "name": "phillip", "country": "south africa", "gender": "male", "age": 22 }
```

Then ingest them into the index:

```sh
curl -XPOST http://localhost:9200/my-test-index/log/_bulk --data-binary @posts.json
```

### Search

#### Searcing with Query Parameters 

All documents where `name` = `kevin`

```sh
curl -XGET 'http://127.0.0.1:9200/my-test-index/_search?q=name:kevin&pretty'
```

And where `age` < `30`:

```sh
curl -XGET 'http://127.0.0.1:9200/my-test-index/_search?q=age:<30&pretty'
```

And where `name` = `sarah` AND `age` > `30`

```sh
curl -XGET 'http://127.0.0.1:9200/my-test-index/_search?q=name:sarah%20AND%20age:>30&pretty'
```

Search and sort by age:

```sh
curl -XGET 'http://127.0.0.1:9200/my-test-index/_search?q=*&sort=age:asc&pretty'
```

#### Term Query

Search for all documents where `country` = `ireland`:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query" : {"term" : {"country": "ireland"}
}‘
```

#### Match All Query

Match all documents:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "match_all": {} }
}'
```

Match all documents, but set from and the amount of documents to be included in the response:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "match_all": {} }, "from": 10, "size": 10
}'
```

Match all documents and sort by age:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "match_all": {} }, "sort": [{ "age": "asc" }]
}'
```

Match all, then filter results to a field:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "match_all": {} }, "sort": [{ "age": "asc" }]
}'
```

Match all and filter the age between two given values:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { 
    "match_all": {} 
  }, 
  "filter": "range": {
    "age": {"gte": 28,"lte": 36}
  }}}}
}'
```

#### Match Query

Match field where `age` = `29`:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "match": { "age": 29 } }
}'
```

Match a phrase:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "match_phrase": { "country": "south" } }
}'
```

Boolean, should match the gender AND country from query:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "bool": {"must": [
    { "match": { "gender": "female" } },
    { "match": { "country": "ireland" } }
  ]}}
}'
```

Boolean, should match the gender OR country from query:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "bool": {"should": [
    { "match": { "gender": "female" } },
    { "match": { "country": "ireland" } }
  ]}}
}'
```

Boolean, should not match the gender NOR country from query:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "bool": {"must_not": [
    { "match": { "gender": "female" } },
    { "match": { "country": "ireland" } }
  ]}}
}'
```

Boolean, must match the gender, `must_not` match the country from query:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "query": { "bool": {
    "must": [
      { "match": { "gender": "male" } },
    ],
    "must_not": [
      { "match": { "country": "new zealand" } },
    ]  
  }}
}'
```

#### Aggregations

Groups documents by country, similar like: 

```sql
SELECT country, COUNT(*) FROM docs GROUP BY country ORDER BY COUNT(*);
```

which will be in elasticsearch:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "aggs": {
    "group_by_country": {"terms": {"field": "country.keyword"}}
  }, 
  "size": 0
}'
```

Calculate the average age by gender, we nest the average_age aggregation inside the group_by_gender aggregation:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "size": 0,
  "aggs": {
    "group_by_gender": {
      "terms": {
        "field": "gender.keyword"
      },
      "aggs": {
        "average_age": {"avg": {"field": "age"}}
      }}}
}'
```

Then we can also sort on the average age in descending order:

```sh
curl -s -XGET 'http://127.0.0.1:9200/my-test-index/_search' -d '
{
  "size": 0,
  "aggs": {
    "group_by_gender": {
      "terms": {
        "field": "gender.keyword", 
        "order": {"average_age": "desc"}
      },
      "aggs": {
        "average_age": {"avg": {"field": "age"}}
      }}}
}'
```

### Delete

To delete a single document from a index, by it's document id:

```sh
curl -XDELETE http://127.0.0.1:9200/my-test-index/_doc/1
```

To delete a index:

```sh
curl -XDELETE http://127.0.0.1:9200/my-foo-index
```

To delete by query:

```sh
curl -s -H 'Content-Type: application/json' \
  -XPOST 'http://127.0.0.1:9200/my-foo-index/_delete_by_query?scroll_size=1000' -d '
{
  "query" : {"term" : { "country": "south africa"}}
}'
```

### Reindex

Reindex all the data from a source index to a target index:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex' -d '
  {
    "source": {
      "index": ["my-metrics-2019.01.03"]
    }, 
    "dest": {
      "index": "archived-metrics-2019.01.03", 
    }
}'
```

Reindex multiple source indices to one target index:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex' -d '
  {
    "source": {
      "index": ["my-metrics-2019.01.03", "my-metrics-2019.01.04"]
    }, 
    "dest": {
      "index": "archived-metrics-2019.01", 
    }
}'
```

Reindex only missing documents from source to target index. You will receive conflicts for existing documents, but the proceed value will ignore the conflicts.

```sh 
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex' -d '
  {
    "conflicts": "proceed", 
    "source": {
      "index": ["my-metrics-2019.01.03"]
    }, 
    "dest": {
      "index": "archived-metrics-2019.01.03", "op_type": "create"}
}'
```

Reindex filtered data to a target index, by using a query:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex' -d '
  {
    "source": {
      "index": "my-metrics-2019.01.03",
      "type": "log",
      "query": {
        "term": {
          "status": "ERROR"
        }
      }
    },
    "dest": {
      "index": "archived-error-metrics-2019.01.03"
    }
}'
```

Reindex the last 500 documents based on timestamp to a target index:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex' -d '
  {
    "size": 500, 
    "source": {
      "index": "my-metrics-2019.01.03",
      "sort": {
        "timestamp": "desc"
      }
    }, 
    "dest": {
      "index": "archived-last500-metrics-2019.01.03", 
      "op_type": "create"
    }
}'
```

Reindex only specific fields to a target index:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex' -d '
  {
    "source": {
      "index": "my-metrics-2019.01.03",
      "_source": [
        "response_code", "request_method", "referral"
      ]
    }, 
    "dest": {
      "index": "archived-subset-metrics-2019.01.03"
    }
}'
```

### Update Replicas

Increase/Decrease the number of Replica Shards using the Settings API:

```sh
curl -XPUT -H 'Content-Type: application/json' \   'http://127.0.0.1:9200/my-test-index/_settings' \
  -d '{"index": {"number_of_replicas": 1, "refresh_interval": "30s"}}'
```

### Snapshots

View snapshot repositories:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_snapshot?format=json'
```

View snapshots under repository (table view):

```sh
curl -s -XGET 'http://127.0.0.1:9200/_cat/snapshots/index-backups?v'
```

View snapshots under repository (json view):

```sh
curl -s -XGET 'http://127.0.0.1:9200/_cat/snapshots/es-index-backups?format=json'
[{"id":"snapshot_2020.10.22","status":"SUCCESS"....
```

Create a snapshot with all indices and wait for completion:

```sh
curl -XPUT -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_snapshot/index-backups/my-es-snapshot-latest?wait_for_completion=true'
```

View snapshot status:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_cat/tasks?detailed'
# cluster:admin/snapshot/create ..
```

View snapshot info:

```sh
curl -s 'http://127.0.0.1:9200/_snapshot/es-index-backups/my-es-snapshot-latest' | jq .
```

For more info on snapshots, view [this post](https://sysadmins.co.za/snapshot-and-restore-indices-on-elasticsearch/)

### Restore Snapshots

Restore with original names:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_snapshot/es-index-backups/test-snapshot-latest/_restore' -d '
{
  "indices": [
    "kibana_sample_data_ecommerce", "kibana_sample_data_logs"
  ], 
  "ignore_unavailable": false, 
  "include_global_state": false 
}'
```

View the restored indices:

```sh
curl 'http://127.0.0.1:9200/_cat/indices/kibana_sample*?v'
health status index
green  open   kibana_sample_data_logs
green  open   kibana_sample_data_ecommerce
```

Restore and rename:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_snapshot/es-index-backups/test-snapshot-latest/_restore' -d '
{
  "indices": [
    "kibana_sample_data_ecommerce", "kibana_sample_data_logs"
  ], 
  "ignore_unavailable": false, 
  "include_global_state": false, 
  "rename_pattern": "(.+)", 
  "rename_replacement": "restored_index_$1" 
}'
```

View the restored indices:

```sh
curl 'http://127.0.0.1:9200/_cat/indices/*restored*?v'
```
```
health status index
green  open   restored_index_kibana_sample_data_ecommerce 
green  open   restored_index_kibana_sample_data_logs
```

Restore and rename with a different name pattern:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_snapshot/es-index-backups/test-snapshot-latest/_restore' -d '
{ 
  "indices": [
    "kibana_sample_data_ecommerce", "kibana_sample_data_logs"
  ], 
  "ignore_unavailable": false, 
  "include_global_state": false, 
  "rename_pattern": 
  "kibana_sample_data_(.+)", 
  "rename_replacement": "restored_index_$1" 
}'
```

View the restored indices:

```sh
curl 'http://127.0.0.1:9200/_cat/indices/*restored*?v'
```
```
health status index                                       
green  open   restored_index_ecommerce                    
green  open   restored_index_logs                         
```

For more info on snapshots, view [this post](https://sysadmins.co.za/snapshot-and-restore-indices-on-elasticsearch/)

### Tasks

View tasks in table format:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_cat/tasks?v&detailed' 
```

View tasks in json format:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_tasks?detailed&format=json' 
```

View tasks in json format and pretty print:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_tasks?detailed&pretty&format=json' 
```

View all tasks relating to snapshots being created:

```sh
curl -s -XGET 'http://127.0.0.1:9200/_tasks?detailed=true&pretty&actions=cluster:admin/snapshot/create'
```

View all tasks relating to write actions:

```sh
curl -s -XGET "http://127.0.0.1:9200/_tasks?detailed=true&pretty&actions=indices:*/write*"
```

Create a Task:

```sh
curl -XPOST -H 'Content-Type: application/json' 'http://127.0.0.1:9200/_reindex?wait_for_completion=false' -d '{"source": {"index": "metricbeat-2020.*"}, "dest": {"index": "metricbeat-2020"}}'
{"task":"-thJvCFgQlusd2vVFZGOfg:26962"}
```

View Task Status by TaskId:

```sh
curl http://localhost:9200/_tasks/-thJvCFgQlusd2vVFZGOfg:26962?pretty
```

Cancel a Task by TaskId:

```sh
curl -s -H 'Content-Type: application/json' -XPOST "http://localhost:9200/_tasks/-thJvCFgQlusd2vVFZGOfg:26962/_cancel"
```

Some of the other actions:

```
"action": "cluster:monitor/tasks/lists
"action": "cluster:monitor/tasks/lists
"action": "cluster:monitor/nodes/stats"
"action": "cluster:admin/snapshot/create"
"action": "internal:cluster/snapshot/update_snapshot_status"
"action": "indices:data/read/search
 - "description": (context of query)
"action" : "indices:data/read/msearch"
"action" : "indices:data/write/bulk
```

### Retire Old Timeseries Indices

This will show how to retire old indices by reducing the number of replicas and reduce the number of segments from the shards by force merging them. Ensure no clients are writing to these indices.

View the indices we want to reindex:

```sh
curl -XGET http://localhost:9200/_cat/indices/myapp-metrics-2020.08*?v
```

Ensure that any data that is currently stored in the transaction log is also permanently stored in the index:

```sh
curl -s -H 'Content-Type: application/json' -XPOST 'http://localhost:9200/myapp-metrics-2020.08.09/_flush'
```

Create the inactive index, which will contain one primary and one replica shard:

```sh
curl -XPUT -H "Content-Type: application/json" \
  http://localhost:9200/myapp-metrics-2020.08_inactive \
  -d '{"index": {"number_of_shards":"1","number_of_replicas": 1, "refresh_interval": "-1"}}'
```

Reindex all the data from our selection of source indices to our inactive index:

```sh
curl -s -H 'Content-Type: application/json' -XPOST 'http://localhost:9200/_reindex' -d '{"source": {"index": "myapp-metrics-2020.08.*"}, "dest": {"index": "myapp-metrics-2020.08_inactive"}}'
```

We can monitor the status of our reindex task:

```sh
curl -s -XGET 'http://localhost:9200/_tasks?actions=*data/write/reindex&detailed&pretty'
```

As we have disabled our refresh interval, once the task has completed, refresh the index:

```sh
curl -s -H 'Content-Type: application/json' -XPOST 'http://localhost:9200/myapp-metrics-2020.08_inactive/_refresh'
```

Now we can view our indices:

```sh
curl -XGET http://localhost:9200/_cat/indices/myapp-metrics-2020.08*?v
```

And delete the source indices:

```sh
curl -XDELETE 'http://localhost:9200/myapp-metrics-2020.08.*'
```

View the segments from our target index:

```sh
curl -XGET http://localhost:9200/_cat/segments/myapp-metrics-2020.08_inactive?v
```

Also view the segments information from the stats:

```sh
curl -s -XGET http://localhost:9200/myapp-metrics-2020.08_inactive/_stats?pretty | jq ._all.primaries.segments
```

Clear the cache:

```sh
curl -s -H 'Content-Type: application/json' -XPOST 'http://localhost:9200/myapp-metrics-2020.08_inactive/_cache/clear'
```

Force merge all the segments from the shards:

```sh
curl -s -H 'Content-Type: application/json' -XPOST 'http://localhost:9200/myapp-metrics-2020.08/_forcemerge?max_num_segments=1'
```

### Extra Resources

  - https://sysadmins.co.za/tag/elasticsearch/
  - https://blog.ruanbekker.com/blog/categories/elasticsearch/

### Thank You

Make sure to say hi on Twitter at [@ruanbekker](https://twitter.com/ruanbekker)
 
