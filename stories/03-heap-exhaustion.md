July 2014 - Heap exhaustion on a code search storage node

[support 3813](https://support.elasticsearch.com/requests/3813)

## background

At this point in time our `codesearch` cluster is running ES 0.90.10. There are
10 Dell R720 servers with ~7TB of SSD storage and 10GB network connectivity. We
also have 3 smaller machines that serve only as masters. This cluster has been
running happily over the past eight months, and performance has been great.

This cluster has gone through several upgrades of the installed ES version.
There have been some hiccups, but overall upgrades have gone smoothly.

## what went wrong

On the morning of July 23rd, `codesearch-storage5` exhausted the JVM heap and
dropped out of the cluster. Our nagios alerting system sent out the alert that
the cluster was in distress.

![](/images/2015-07-23-codesearch-heap.png)

The cluster went into a yellow state (as expected given the circumstances), but
it failed to start allocating shards to the other storage nodes. Something else
was amiss. We restarted the ElasticSearch process on the storage node, and it
rejoined the cluster. However, the cluster remained yellow and still refused to
allocate shards to any machine.

## how we fixed it

The original problem - the JVM heap exhaustion - points to an overburdened
cluster. The other nodes in the cluster also show high old generation JVM heap
usage and lackluster old generation GC events. The disk usage is also telling
for this cluster.

```
node name               | disk | used | free | percent
------------------------+------+------+------+--------
codesearch-storage1       6.9T   5.9T  1022G   86%
codesearch-storage2       6.9T   6.2T   699G   91%
codesearch-storage3       6.9T   6.1T   841G   89%
codesearch-storage4       6.9T   6.0T   935G   87%
codesearch-storage5       6.9T   6.3T   630G   92%
codesearch-storage6       6.9T   6.2T   672G   91%
codesearch-storage7       6.9T   6.1T   859G   88%
codesearch-storage8       6.9T   6.1T   843G   88%
codesearch-storage9       6.9T   6.1T   870G   88%
codesearch-storage10      6.9T   6.0T   921G   87%
```

Each storage node is above 85% disk usage. This is important because we are
using the `cra.disk.watermark.low` and `cra.disk.watermark.high` settings to
control shard allocation to the machines. If the used disk space on a storage
node goes above the low watermark of 85% then ES will not allocate shards to
that node. If the used disk space goes above the high watermark of 90% then ES
will actively try to move shards off that node.

The problem here is that all of our storage nodes are above the low watermark.
ES will not allocate shards to our newly restarted `codesearch-storage5` node
even though it has shard data on disk; after the restart it has data on disk but
zero shards are assigned to the node.

How do we fix this problem? Do we delete all 6.3TB of data on disk and allow ES
to replicate shards to this machine? That is certainly an option, but
fortunately we have brilliant operations engineers who are gifted with
foresight. They also have a magic bag filled with hardware tricks.

The Dell R720 machines are configured with 16 SSD disks. Two are paired into a
raid-1 configuration; this is where the operating system lives. The remaining 14
drives are configured as a single raid-0 volume; this is where the ElasticSearch
data lives. When this raid-0 partition was created, 10% of the disk space was
held in reserve to be used later when the SSD disks started exhibiting bad
sectors.

The trick we played was to shutdown the `codesearch-storage5` machine and expand
the raid-0 partition via LVM commands. When ElasticSearch was restarted on the
machine, the amount of disk used was below the low watermark threshold of 85%.
ElasticSearch then began recovering shards onto this node.

The cluster returned to a green state and we performed a rolling restart of each
node in the cluster. As we restarted each node we expanded the raid-0 partition
to give us more disk space. This is only a short term solution. The longer term
solution is to add more machines to the cluster. With each machine added we gain
more JVM heap which is what the cluster really needs.

## lessons learned

Keep an eyes on your key metrics and watch for long term growth trends.

Monitoring is great, but you really need to have alerting on the metrics that
you monitor. A nagios alert when these machines reached the low watermark would
have saved us quite a bit of trouble.

Forecasting growth is essential when you are running a search cluster on your
own hardware. Physical hardware can be a long lead time item. You do not want to
be in a position where your cluster goes down and the only solution is to wait
six weeks for hardware to be delivered and installed.

