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

Something has definitely changed in ElasticSearch after the upgrade to ES 1.4.2;
the question is "what is different". And more importantly "can we fix it".

* open a [support request ticket](https://support.elasticsearch.com/requests/7200)
* get `hot_threads`
* see that 95% of our thread time is spent in IOWait
* realize we are calling `/_nodes/_local/stats` a lot!

![not tracking management threads](/images/2015-02-05-githubsearch3-management-threads.png)

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


