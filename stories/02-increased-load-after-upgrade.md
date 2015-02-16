Jan 2015 - Increased load after 1.4.2 upgrade

## background

We have a search cluster than handles all the production indices that are not
code search - things like issues, pull requests, users, repositories, and audit
logs. We are excited to upgrade all our clusters from ES 1.2.3 to ES 1.4.2 -
much work has been done to reduce the JVM heap consumption used to maintain
indices.

This is a 5 node cluster with 2 storage nodes and 3 dedicated master nodes.

## what went wrong

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
IO available. Eventually the process became unresponsive and the machine dropped
out of the cluster. This stopped the upgrade process, and the machine then
rejoined the cluster.

## how we fixed it

## lessons learned

beware the `_upgrade` command - the documentation is unclear on how the upgrade
progresses

you can halt an upgrade by closing the index (dangerous!)

streaming shards instead of recovering from local data - more of an annoyance
than a problem

measure everything

things that worked previously might not work in the future as ElasticSearch
changes


