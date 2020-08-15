---
layout: post
title: High availability and long term storage Prometheus using Thanos
subtitle:
tags: [monitoring]
---

When it comes to monitoring, prometheus is probably one of the option who's gonna come out.

Prometheus is a flexible monitoring tool, and can get every metrics you need. But it comes with some downside:

### Data incoherence when high availability is set

How do we upgrade a single prometheus without downtime ? You can't.

To solve it, we add one or more prometheus instances, which will all scrape the same datas, but we will now deal with another issue, which is inconsistent datas. 

For example, one prometheus get a returned value, and others has different values, because of scrape latency or application issue (one app returns data and the other one return nothing).

To avoid using many Prometheus, scraping all targets, we could use *sharding*, and have one prometheus per service(s). But this is could lead to an overhead.


### Long term storage and slow queries

Prometheus has a default retention period of 15 days, this is setting is configurable. Let's say we want a retention of 365 days, for audit purposes or to provide SLA on services.

To handle all this tsdb, we would end up with a huge storage disk, which could leads to difficulties from an operational point of view (backup, latencies).

It also means that a query within an old time range period of time could lead to a slowness as well.


# Thanos to the rescue

![Thanos Logo](https://github.com/thanos-io/thanos/blob/master/docs/img/Thanos-logo_fullmedium.png?raw=true)

*Thanos solves the problems mentionned above*


### Data incoherence when high availability is set

Thanos creates a sidecar on each prometheus server and will allow a component called **Querier** to fetch datas. 

The Querier will will fetch all datas from sidecars and uses built-in algorithms to provide a global and unified view of all tsdb from every prometheus servers. You'll want to connect this component if you want to connect it as datasource in Grafana.

### Long term storage and slow queries

Sidecar container will upload all tsdb block files every 2 hours by default on a remote object storage (s3, gcs, azure).
So your local storage don't grow as much and we get the benefit of unlimited storage like S3.

Now, if you query an old time series, the **store gateway** component will retrieve it for you from remote s3 storage. 
If the time series is recent, the query will be locally.

*But if I query a 1 year old time series from a remote storage, it will be slow, no ?*

This is why Thanos uses a **compactor**, which reduce the time series data footprint by **downsampling** the datas.
For instance if you query a time series for 1 year, instead of having thousands of 2 hours blocks, it's gonna aggregate it with 2 weeks blocks for instance. So the return query is faster.


## Conclusion

Thanos is a great tool and is part of CNCF project sandbox as well.

The implementation has been well think and make Prometheus reliable, you guys now has the infinite gauntlet to play with ;)

**What Did It Cost You? Everything**

![Thanos Img](https://specials-images.forbesimg.com/imageserve/5cc30b87a7ea436c70f3f17f/960x0.jpg?fit=scale)


## Useful links

[Thanos documentation](https://thanos.io/)

[https://prometheus.io/](https://prometheus.io/)

[https://github.com/coreos/prometheus-operator](https://github.com/coreos/prometheus-operator)