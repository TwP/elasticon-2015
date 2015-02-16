Jan 2013 - Initial failure of the code search cluster

[blog post](https://github.com/blog/1381-a-whole-new-code-search)
[post mortem](https://github.com/blog/1397-recent-code-search-outages)

## background

Our initial roll-out of source code search occurred on Jan 23, 2013

![initial announcement](/images/Screen Shot 2013-01-23 at 2.03.45 PM.PNG)

Someone discovers private keys are discoverable via code search. Twitter storm
ensues.

![private keys](/images/Screen Shot 2013-01-23 at 3.43.07 PM.PNG)

Cluster falls over the following day.

## what went wrong

Cluster facts

* 26 storage nodes
* 8 client nodes
* 17 TB of source code files in the search index
* ElasticSearch 0.20.2 running on Java 6

Initial indication that something was amiss was a rise in exceptions from our
application regarding timeouts communicating with the code search cluster.

Several of the storage nodes were consuming nearly 100% of available CPU. This
was purely from the ElasticSearch process running on these machines; disk IO and
network IO was not contributing to the load.

Messages in the ElasticSearch logs indicated that machines were rapidly being
elected as master and then later removed from the master role. The master role
was rapidly being moved around the cluster.

## how we fixed it

With quite a lot of help from the ElasticSearch team. Drew Raines and Shay
Banon hopped into a campfire chat room with us, and spent the better part of
two days helping us bring the cluster back online.

The first step was to shutdown the entire cluster and disable indexing and
searching until the cluster was repaired.

As we brought the cluster back online, several shards would not allocate to any
node even though Lucene segment files were present. Some shards were corrupted
and unable to be recovered.

Two bugs in ElasticSearch were discovered by Shay Banon during the recovery
process. The ElasticSearch team cut a new release of ES (0.20.3) that we
deployed to our cluster.

Java 6 has "problems" with the NIO libraries that make it unsuitable for use
with ElasticSearch. We upgraded to Java 7.


## lessons learned

test out Java versions before using in production

realistic load testing is essential

set a `minimum_master_nodes` count in your configuration

set `ES_HEAP_SIZE` to avoid JVM pauses as it allocated more heap from the system


## segue

* measure everything
* determine what is "normal"
* look for variances from "normal"
* predict when you'll exceed "normal"

