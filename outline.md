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

* show the last 60 weeks of code search query metrics
* this is our check engine light
* each "feature" on this graph has it's own story to tell - so let's dive into
  these stories

## Migration away from AWS to real hardware

* AWS is expensive
* much better performance from real hardware

## Understand your queries

### 
