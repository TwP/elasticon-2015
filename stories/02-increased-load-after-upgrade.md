Jan 2015 - Increased load after 1.4.2 upgrade

## background

We have a search cluster than handles all the production indices that are not
code search - things like issues, pull requests, users, repositories, and audit
logs. We are excited to upgrade all our clusters from ES 1.2.3 to ES 1.4.2 -
much work has been done to reduce the JVM heap consumption used to maintain
indices.

This is a 5 node cluster with 2 storage nodes and 3 dedicated master nodes.

The rolling restart went just fine. We started with the three master nodes and
then proceeded to the two storage nodes. The first storage node did not recover
shards from local storage; all shards were streamed from the other storage node.
We closely monitored disk utilization and adjusted the
`cluster.routing.allocation.node_concurrent_recoveries` to prevent the other
node from being overwhelmed. The restart of the second node went much more
quickly as shards were recovered from local storage.

The trouble started when we began to `_upgrade` indices. Our first problem was
the hourly cron job we have in place to take snapshots of repositories. After
the restart all the primaries were located on one storage node. The `_upgrade`
and the snapshots were competing with one another and caused degraded
performance.

After disabling the snapshot cron job, we then tried running `_upgrade` on
several indicies at once. The documentation states that the upgrade will only
happen on one shard per node in the cluster at a time. What the documentation
failed to mention was that this is only the case if you set the
`wait_for_completion` flag on the request.

Ten indices started an `_upgrade` at the same time and quickly consumed all disk
IO available. Eventually the ES process became unresponsive and the machine
dropped out of the cluster. This stopped the upgrade process, and the machine
then rejoined the cluster again.

Fortunately there was no data loss and no shard corruption. Having recent
snapshots also reduces worry; we could easily restore from snapshots.

We finished the `_upgrade` process by upgrading each index one by one. The
remainder of the indices we upgraded by re-indexing from the information stored
in our database.

## what went wrong

Believe it or not, the upgrade process is not what we want to focus on here. The
real problems started after the upgrade was complete. Under ES 1.2.3, the
nominal load on this cluster was in the ~0.5 to ~1.5 range. After the
upgrade to ES 1.4.2, the nominal load on this cluster was in the ~4.5 to ~5.5
range.

![](/images/2015-02-05-githubsearch3-load.png)
![](/images/2015-02-05-githubsearch3-disk-utilization.png)
![](/images/2015-02-05-githubsearch3-cpu.png)

https://github-images.s3.amazonaws.com/s3-image-uploader/ed442c8b911d1a816cb82d8ef0675fc28062e6893a652403a6/graph.png

Obviously a factor of five increase in load falls outside the realm of normal.
The question now is: is this the new normal, or is something else going on?

Disk utilization was similar before and after the upgrade. So we are not
experiencing higher load due to the new Lucene segment file format.

The increase in load was solely an increase in system CPU usage and an increase
in user process CPU usage. We also saw an increase JVM old-gen heap usage.
Memory was being promoted to the old-gen heap at a much higher rate after the
upgrade than before the upgrade.

## how we fixed it

Something has definitely changed in ElasticSearch after the upgrade to ES 1.4.2.
The question is "what is different", and more importantly "can we fix it".

Our first thought was that this is the new normal with ES 1.4.2 and we just have
to get used to it. We opened a [support ticket](https://support.elasticsearch.com/requests/7200)
with ElasticSearch to see if this really was the new normal or if something was
wrong with this cluster.

The first thing ElasticSearch support asked for was a listing of the
`/_nodes/hot_threads` from the two storage nodes.

```
97.4% (487.1ms out of 500ms) cpu usage by thread 'elasticsearch[githubsearch3-storage1-cp1-prd][management][T#2]'
 9/10 snapshots sharing following 9 elements
   org.elasticsearch.action.admin.indices.stats.ShardStats.<init>(ShardStats.java:49)
   org.elasticsearch.indices.InternalIndicesService.stats(InternalIndicesService.java:212)
   org.elasticsearch.node.service.NodeService.stats(NodeService.java:156)
   org.elasticsearch.action.admin.cluster.node.stats.TransportNodesStatsAction.nodeOperation(TransportNodesStatsAction.java:96)
   org.elasticsearch.action.admin.cluster.node.stats.TransportNodesStatsAction.nodeOperation(TransportNodesStatsAction.java:44)
   org.elasticsearch.action.support.nodes.TransportNodesOperationAction$AsyncAction$2.run(TransportNodesOperationAction.java:141)
   java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
   java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
   java.lang.Thread.run(Thread.java:745)

97.3% (486.3ms out of 500ms) cpu usage by thread 'elasticsearch[githubsearch3-storage1-cp1-prd][management][T#3]'
 2/10 snapshots sharing following 20 elements
   java.io.UnixFileSystem.getLength(Native Method)
   java.io.File.length(File.java:968)
   org.apache.lucene.store.FSDirectory.fileLength(FSDirectory.java:258)
   org.apache.lucene.store.FileSwitchDirectory.fileLength(FileSwitchDirectory.java:147)
   org.apache.lucene.store.FilterDirectory.fileLength(FilterDirectory.java:63)
   org.elasticsearch.index.store.DistributorDirectory.fileLength(DistributorDirectory.java:113)
   org.apache.lucene.store.FilterDirectory.fileLength(FilterDirectory.java:63)
   org.elasticsearch.common.lucene.Directories.estimateSize(Directories.java:43)

96.4% (482.1ms out of 500ms) cpu usage by thread 'elasticsearch[githubsearch3-storage1-cp1-prd][management][T#4]'
 2/10 snapshots sharing following 19 elements
   org.apache.lucene.store.FSDirectory.listAll(FSDirectory.java:223)
   org.apache.lucene.store.FSDirectory.listAll(FSDirectory.java:242)
   org.apache.lucene.store.FileSwitchDirectory.listAll(FileSwitchDirectory.java:87)
   org.apache.lucene.store.FilterDirectory.listAll(FilterDirectory.java:48)
   org.elasticsearch.index.store.DistributorDirectory.listAll(DistributorDirectory.java:88)
   org.apache.lucene.store.FilterDirectory.listAll(FilterDirectory.java:48)
   org.elasticsearch.common.lucene.Directories.estimateSize(Directories.java:40)
```

The top three active threads on this storage node are all *management* threads.
The first thread is collecting shard statistics. The next two threads are
estimating the index size by looking at the Lucene segment files on disk.

Why are these management threads consuming so much of the application time, CPU,
and heap? There is no way that ES 1.4.2 could be this horribly inefficient and
be released by ElasticSearch.

Let's look through out application and see where we are calling `/_nodes/stats`
against this cluster. There are two places where this happens regularly. The
first is our metrics collection framework. Every 10 seconds we request stats
from each node in the cluster and then store those stats in graphite (which
generates all these pretty graphs). It would be terrible news if those stats
calls are causing this amount of overhead on the cluster. We would have to
rethink our fundamental approach to monitoring and operating our ES clusters.

The second place where we are calling `/_nodes/stats` is via haproxy. When an
application connects to an ElasticSearch cluster, it always connects to a
designated port on `localhost` - `http://localhost:9210` in the case of the
`githubsearch3` cluster. This is a connection to haproxy that round-robins
requests to the two storage nodes. haproxy is configured to check that the
storage nodes are online by making a request to `/_nodes/_local/stats`. We have
this haproxy configuration setup across ~100 machines; this means we have ~100
machines making these `/_nodes/_local/stats` requests to the githubsearch3
cluster.

ElasticSearch provides statistics on thread pool usage. So let's look at the
management thread pool and see if we have increased our haproxy stats calls.

![not tracking management threads](/images/2015-02-05-githubsearch3-management-threads.png)

Shoot! We are not collecting these metrics from our stats sampler (that is the
process referred to above where we are sampling metrics every 10 seconds from
our ElasticSearch clusters.). So our first order of business is to add the
management thread pool to the list of metrics we are going to sample. After that
is done we can generate graphs, but we do not have any indication of "normal"
prior to the upgrade.




* add samplers for the management thread stats
* change our haproxy checks to call the ping endpoint instead
* load drops back down to pre-upgrade levels
* the number of management requests drops by a factor of five

## lessons learned

beware the `_upgrade` command - the documentation is unclear on how the upgrade
progresses

you can halt an upgrade by closing the index (dangerous!)

streaming shards instead of recovering from local data - more of an annoyance
than a problem

measure everything

things that worked previously might not work in the future as ElasticSearch
changes


