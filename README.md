## Elastic{on} 2015 Talk

This repository exists to collect our GitHub ElasticSearch stories and then
craft them into coherent talk for Elastic{on} 2015. The stories are being
collected in the `stories` directory. Any images and supporting graphs are going
into the `images` folder. The talk itself will live in the `talk` folder.

Please contribute stories if you have any. The basic format of a story follows
the post-mortem format:

* background
* what went wrong
* how we fixed it
* lessons learned

There is an optional "segue" section if there are some extra notes that will
help the talk transition from one story to the next.

### Theme

**Elasticsearch in Anger**<br/>
_stories from the GitHub search clusters_

* Where have we come from
* What are we doing
* Where are we going

The talk will walk through a code-search query metrics graph over the past 80 /
90 weeks or so. We’ll highlight major events that appear on this graph. Along
the way we’ll talk about how we use graphs and metrics to see past performance
and predict future load and scaling needs. We’ll talk about the tools we use
each day to handle index upgrades, cluster migrations, and other operational
tasks.
