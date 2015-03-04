## Title

Elasticsearch in Anger
stories from the GitHub search clusters

## Overview

* where we have come from
* what we are doing
* where we are going

## Failure of the code search cluster

### code search relaunch

* relaunched on Jan 23, 2013
  ![](/images/Screen Shot 2013-01-23 at 2.03.45 PM.PNG)

### secret keys

* twitter announcement that secret keys in public repositories are discoverable
  ![private keys](/images/Screen Shot 2013-01-23 at 3.43.07 PM.PNG)

### cluster failure

* cluster falls over the following day
* terrible way to relaunch a feature

### Drew Raines & Shay Banon

* we contact Drew Raines and asked for help
* he and other ES staff spent 48 hours with us in a campfire chat room bringing
  the cluster back online
* Elasticsearch support staff are amazing

### what went wrong?

* insufficient load testing on our part
* Java 6 has bugs with NIO that make it unsuitable for use with Elasticsearch
* Elasticsearch had some bugs with shard recovery and allocation
  * Shay Banon wrote some quick fixes and cut us new releases
* minimum master nodes prevent split brain and rapid master election cycles

## Elasticsearch check engine light

* slide with a check engine light
* wouldn't it be nice if Elasticsearch had a check engine light?

## Code search query metrics

* SLIDE show the last 60 weeks of code search query metrics
* this is our check engine light
* each "feature" on this graph has it's own story to tell - so let's dive into
  these stories

## Migration away from AWS to real hardware

* SLIDE - highlight the dip when we switched over to AWS (skitch the previous graph)
* AWS is expensive
* much better performance from real hardware

### how we manage search indices

* SLIDE - stafftools/search_indexes page
* all indices have a numeric name
* we use an alias to point to the current index that is serving production queries

### index versions

* SLIDE - mapping => `{"index-meta":{"_meta":{ ... }}}`
* index settings and mappings are maintained in source code
* we calculate a SHA1 hash of the data
* and store that hash in mapping metadata in the index

### scientist for load testing

* SLIDE - scientist `control` / `candidate` block
* we can route production queries to another cluster for load testing
* chatops commands to control percentage of queries that flow to the candidate

### why such fancy tools?

* SLIDE - show our stafftools/search_indexes page (again)
* automatic upgrades for GitHub Enterprise

## Understand your queries

* SLIDE - highlight the dip where we used the proper filter level (skitch the previous graph)
* top-level filters vs `filtered_query`
* top-level filters and those same filters at the facet level
* use filtered queries instead

* ANOTHER SLIDE - show the same dip in Issues queries

## And then ignore your graphs

* putting electrical tape over your check engine light
* seriously, it is the best way to get a heap exhaustion
* and an out of memory error

## Heap exhaustion and OOM

### what happened

* SLIDE - highlight the rise in query times and the cliff (skitch the previous graph)
* July 23, 2014
* storage5 ran out of heap and threw an `OutOfMemory` exception
* too much data on this particular machine - an overburdened cluster
* cluster remained yellow after the restart and refused to allocate shards

### segment data and heap usage

* SLIDE - show an `/es jvm heap` graph
* filter cache and field cache
* Lucene segment files use old-gen heap
* adding more data uses more heap (gotten better with ES 1.4)

### disk watermarks

* SLIDE - show the disk usage table
* all machines were above the low watermark
* Elasticsearch won't allocate shards unless disk capacity is below the low
  watermark on that machine even if the shards are already on disk

### have to add capacity

* SLIDE - gif of hard drive platters racing across the floor
* LVM to the rescue
* allocate reserved drive space to the raid0 data partition
* cluster begins to recover

### monitor the things that are limited

* pull the tape off the check engine light
* we now have nagios alerts for low and high disk watermark events

### forecasting is even better than monitoring

* SLIDE - show the `/es forecast disk` command

## We :heart: graphs!

* SLIDE - the three "where/what/where" bullet points below
* they help us figure out
  * where we have come from
  * what we are doing
  * where we are going
* essential to operating our clusters

* "where" questions are strategic - great for planning
* "what" question is tactical - how is it going right now?

### graph hygiene

* measure everything
* dashboards for strategic data
* ad-hoc graphs for tactical actions - chatops & /graph me

## Elasticsearch 1.4 upgrade on githubsearch3

### research1 cluster

* SLIDE - stafftools/search_indexes page
* spun up a research cluster to test out 1.4.2
* used our index creation / backfilling to put production queries online
* let the research1 cluster handle production queries during the upgrade

### pause indexing queues

* SLIDE - `/gh resque pause {index_high,index_low,index_bulk}`
* allow jobs to queue up and then we can process them after the upgrade is complete

### high load post upgrade

* SLIDE - load
* SLIDE - cpu
* SLIDE - disk IO
* what's going on here? is this the new "normal" under 1.4.2?

### hot threads

```
97.4% (487.1ms out of 500ms) cpu usage by thread 'elasticsearch[githubsearch3-storage1-cp1-prd][management][T#2]'
 9/10 snapshots sharing following 9 elements
   org.elasticsearch.action.admin.indices.stats.ShardStats.<init>(ShardStats.java:49)

97.3% (486.3ms out of 500ms) cpu usage by thread 'elasticsearch[githubsearch3-storage1-cp1-prd][management][T#3]'
 2/10 snapshots sharing following 20 elements
   java.io.UnixFileSystem.getLength(Native Method)

96.4% (482.1ms out of 500ms) cpu usage by thread 'elasticsearch[githubsearch3-storage1-cp1-prd][management][T#4]'
 2/10 snapshots sharing following 19 elements
   org.apache.lucene.store.FSDirectory.listAll(FSDirectory.java:223)
```

* these are all threads from the **management** thread pool
* `/_nodes/stats` is the primary user of the **management** thread pool

### look at the management thread pool stats

* SLIDE - empty management thread pool graphs!
* doh!
* add management thread pool stats to our ES sampler

### that's a lot of sampling

* SLIDE - management thread pool stats
* obviously our ES sampler is calling the `/_nodes/_local/stats` endpoint
* but that much?
* oh! haproxy is also calling this endpoint
* the stats endpoint changed to include _all_ stats now hence the extra load

### haproxy setup

* each cluster is assigned a port
* haproxy on "localhost" forwards that port to the cluster storage nodes
* so we have an haproxy running on _every_ host in our data center hitting the
  `/_nodes/_local/stats` endpoint for availability checks :rage4:
* we should use a better availability check like the ping endpoint

### decrease in graphs as haproxy changes roll out

* SLIDE - decrease in management threads
* SLIDE - decrease in load
* SLIDE - decrease in CPU
* yay! we fixed it


## Where do metrics come from

### hardware level

* collectd
* log files

### Elasticsearch

* Amen samplers

### application

* query timing from `elastomer-client`


## App level stuff

### show off our staff tools page

### push button index creation

### index naming and index versions

### writing to multiple indices

### production queries

### "scientist" for load testing

### Research clusters

