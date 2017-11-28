---
title: Spring Cloud Dataflow
tags: ["spring cloud","spring cloud dataflow"]
categories: ["MQ","spring cloud"]
---
* TOC
{:toc}

```
app register --name "source" --type source --uri maven://com.suimi.hello:dataflow-streams-source:0.0.1-SNAPSHOT
app register --name "processor" --type processor --uri maven://com.suimi.hello:dataflow-streams-processor:0.0.1-SNAPSHOT
app register --name "sink" --type sink --uri maven://com.suimi.hello:dataflow-streams-sink:0.0.1-SNAPSHOT

stream create --name "stream" --definition 'source | processor | sink'

stream deploy --name stream

app register --name "task" --type task --uri maven://com.suimi.hello:dataflow-task:0.0.1-SNAPSHOT
task create myjob --difination task
task launch myjob

```
