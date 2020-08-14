---
layout: post
title: Hot-warm-cold architecture with Elasticsearch
subtitle:
tags: [data]
---



Running an Elasticsearch cluster could be a real nighmare when you've got a lot of datas to ingest, design and configuration optimization needs to be think upstream. We're gonna use a feature included in x-pack: Index Lifecycle Management (ILM)

![Elastic Logo](https://static-www.elastic.co/v3/assets/bltefdd0b53724fa2ce/blt5ebe80fb665aef6b/5ea8c8f26b62d4563b6ecec2/brand-elasticsearch-220x130.svg)


ILM has a concept of [hot-warm-cold](https://www.elastic.co/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management).

- A hot node will host all new indexes, and will get all read and writes. Having SSD and high cpu is recommended.

- A warm node will host less recent indexes, 1 week old for instance, and will get all reads. A less performant hardware could be use here.

- A cold node will host old indexes, 3 weeks old for instance, and will get all read actions as well. Having a cheap configuration could be use here.

ILM give to the user the ability to manage his indexes by triggering an action when a condition is met. These conditions are called ILM policies.

Actions could be:

- Move index from hot nodes to warm nodes, then to cold nodes
- Reduce the number of primary shards (shrink)
- Change the number of replica shards
- Reduce the number of segments (force merge)

Policies could be:

- I don't want a document to be bigger than 50G (maximum index side)
- I don't want more than 5000 documents (maximum documents)
- I don't want an index older than (maximum age)

Benefits of using ILM:

- Optimize writes by spreading shards on performant nodes only (hot nodes)
- Optimize read by spreading shards on read-only nodes (warm or cold nodes, and only a few on hot nodes as well.)
- Optimize cost by moving index in appropriate hardware (robust hardware on hot and less robust on warn and cold)
- Optimize overall cluster health by keeping a small shard size and reducing I/O, network bandwidth and make cluster operations faster.


## My cluster overview

I'll skip the configuration details for simplicity.

I used a 4 nodes cluster with attributes below set during my installation:

- master-1 has no attribute set
- data-1 has attribute "temp" set to hot
- data-2 has attribute "temp" set to warm
- data-3 has attribute "temp" set to cold


## Create an ILM for a pseudo app index

We'll create an index for a pseudo app, and create documents to trigger a rollover action.

### Create the ILM policies

New indexes documents will be considered as "hot". 

New Indexes will be create only on dedicated nodes with the attribute "hot" (Index creation to target hot nodes are not defined in ILM policies below, but in "create index templates" part.).

Warm and cold node targets are defined in ILM policies below.

Baiscally what we do below is:

- define four phases: hot, warm, cold and delete.
- move from hot to warm if one of the document condition is met (older than 30d, size reached 50gb, more than 2 documents)
- move from warm to cold of older than 30days
- delete cold documents if older than 60days
- I don't use any replicas here, because my lab setup doesn't have enough node to allow it.

```
PUT _ilm/policy/applogs-ilm
  {
    "policy": {
        "phases": {
            "hot": {
                "actions": {
                    "rollover": {
                        "max_age": "30d",
                        "max_size": "50gb",
                        "max_docs": 2
                    },
                    "set_priority": {
                        "priority": 100
                    }
                }
            },
            "warm": {
                "actions": {
                    "allocate": {
                        "number_of_replicas": 0,
                        "include": {},
                        "exclude": {},
                        "require": {
                            "temp": "warm"
                        }
                    },
                    "set_priority": {
                        "priority": 50
                    }
                }
            },
            "cold": {
                "min_age": "30d",
                "actions": {
                    "allocate": {
                        "require": {
                            "temp": "cold"
                        },
                        "number_of_replicas": 0
                    },
                    "freeze": {}
                }
            },
            "delete": {
                "min_age": "60d",
                "actions": {
                    "delete": {}
                }
            }
        }
    }
}
```


### Create index template

We need now to link an index template to policy we created, and we define were to route our new indexes: node which math the tag "temp:hot".
Again, I don't use replicas here as my lab setup only has one "hot data node".

```
PUT _template/applogs
{
  "index_patterns": [
    "applogs-*"
    ],
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "lifecycle.name": "applogs-ilm",
      "lifecycle.rollover_alias": "applogs-alias",
      "routing.allocation.require.temp": "hot"
    }
  }
}
```

### Create an index which match our template.

There's the concept of alias here, which is how ES can determine where to write, and where to read.
Basically, when a new index is created, we define an alias linked to it with "index_patterns" set above, followed by "-000001". e.g: applogs-000001

When a rollover happen (index moves to warm node for instance), a new index is created, and ES will append the name by one. E.g: applogs-000002. The alias will be link to this new index and mark it as write index.

**important** 

First index name need to match index_pattern defined above, plus 6 digits and end with a 1. If for example you setup "applogs-01" as first index name, Elasticsearch will rollover the second index like this: "applogs-000002"

lifecycle.rollover_alias has to be define to point to the alias that we will create in next section.

```
PUT applogs-000001
{
  "aliases": {
    "applogs-alias": {
      "is_write_index": true
    }
  }
}
```

Verify that the index is actually on hot node (data-1), according to our routing allocation rule.

```
GET _cat/shards/applogs-000001

applogs-000001 0 p STARTED 0 230b 172.31.36.188 data-1
```

### Verify ILM works

Verify that the index has managed_set to true, is in phase hot and the correct policy attached.

```
GET applogs-alias/_ilm/explain (truncate snippet)

{
  "indices" : {
    "applogs-000001" : {
      "index" : "applogs-000001",
      "managed" : true,
      "policy" : "applogs-ilm",
      "phase" : "hot",
      "step" : "wait-for-follow-shard-tasks",
      "phase_execution" : {
        "policy" : "applogs-ilm",
        "phase_definition" : {
          "min_age" : "0ms",
          "actions" : {
            "rollover" : {
              "max_size" : "50gb",
              "max_age" : "30d",
              "max_docs" : 2
                ....
}
```


### Trigger a rollover

Now we need to index three documents to trigger a rollover (the rule of max_doc set to 2 that we previously defined)

Execute the command 3 times, change the doc number each time. (each document has a unique id)

```
PUT applogs-alias/_doc/1
{
  "id": 1
}
```

Now wait for the rollover to happen, it takes 10 min by default but you can override this behaviour and set it to 1 minute.

```
PUT _cluster/settings
{
  "transient": {
    "indices": {
      "lifecycle": {
        "poll_interval": "1m"
      }
    }
  }
}
```

Verify that the rollover happened:

- applogs-000001 has been moved to warm node
- applogs-000002 has been created and is in hot phase


```
GET applogs-alias/_ilm/explain (truncate snippet)

{
  "indices" : {
    "applogs-000002" : {
      "index" : "applogs-000002",
      "managed" : true,
      "policy" : "applogs-ilm",
      "lifecycle_date_millis" : 1596926766848,
      "phase" : "hot",
       ......
      }
    },
    "applogs-000001" : {
      "index" : "applogs-000001",
      "managed" : true,
      "policy" : "applogs-ilm",
      "lifecycle_date_millis" : 1596926766855,
      "phase" : "warm",
      "phase_time_millis" : 1596926767360,
      "action" : "complete",
      "action_time_millis" : 1596926767721,
      "step" : "complete",
      "step_time_millis" : 1596926767721,
      .....
    }
  }
```

Verify that the write index is now applogs-000002.

```
GET applogs-000002
{
  ...
  "aliases": {
    "applogs-alias": {
      "is_write_index": true
  ...
    }
  }
}
```

## Conclusion

ILM has huge benetits if you want to operate an ES cluster with a lot of timeseries datas, it's an official and free tool provided by Elasticsearch.

By using ILM, the ultimate goal is to have improvements on read and writes, and a better cluster stability. My cluster doesn't have enough resource to notice this improvement, but in a real environment, you should see an improvements on these metrics below.

![ES Monitoring](https://github.com/ptran32/ptran32.github.io/blob/master/_posts/img/04-es-monitoring.png?raw=true)


<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://ptran32.github.io/2020-08-08-hot-warm-cold-elasticsearch/"
  },
  "headline": "Hot-warm-cold architecture with Elasticsearch",
  "description": "Hot-warm-cold architecture with Elasticsearch",
  "image": "https://static-www.elastic.co/v3/assets/bltefdd0b53724fa2ce/blt5ebe80fb665aef6b/5ea8c8f26b62d4563b6ecec2/brand-elasticsearch-220x130.svg",  
  "author": {
    "@type": "Person",
    "name": "Patrice"
  },  
  "publisher": {
    "@type": "Organization",
    "name": "Patrice",
    "logo": {
      "@type": "ImageObject",
      "url": ""
    }
  },
  "datePublished": "2020-08-08",
  "dateModified": "2020-08-08"
}
</script>