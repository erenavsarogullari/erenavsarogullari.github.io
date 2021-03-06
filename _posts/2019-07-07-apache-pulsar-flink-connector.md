---
layout: post
title: "Apache Pulsar Flink Connector"
category: Dev
tags: [apache pulsar, apache flink, connector, stream processing, batch processing, pub-sub system, distributed systems]
date: 2019-07-07
---

<p>This article aims to show how to write Apache Flink DataSets to Apache Pulsar topic from scratch. Both Flink and Pulsar projects are main layers of Data Pipelines for data processing and message store & transmission tiers . Let' s have a look Flink and Pulsar.</p>

### Apache Flink
<p>Apache Flink is an open-source unified distributed compute engine by supporting Streaming, Batch, ML and Graph Processing use cases. Project moved to Apache Software Foundation(ASF) as incubator in 2014 and graduated from incubator stage as top-level Apache project in same year.</p>

<p>Flink’s some of the key features are as follows:</p>

1. **Stateless & Stateful Stream Processing:** Stream Processing focuses unbounded data processing by supporting different window types such as: time, count, session etc. Data needs to be processed when it is arrived. For state view, stream processing can be classified as stateless and stateful. Stateless Stream Processing means that there is no dependency between current event and previous events so each incoming event will be processed directly without keeping state. However, stateful stream processing means that there is shared state between current event and previous events so each incoming event so past events will influence to be processed current events in the light of current state. Flink exposes **DataStream API** for Stream processing.
1. **Batch Processing:** Data is bounded and has start and end time at Batch Processing. It is one of use case. Flink exposes **DataSet API** for Batch processing.
1. **Complex Event Processing:** Pattern Detection is one of the key challenge in Stream Processing. Flink exposes Complex Event Processing(CEP) API to be detected patterns on current stream. Anomaly and Fraud Detection can be thought as some of the potential use-cases for CEP API.
1. **Exactly-Once Delivery Semantics:** Incoming event will affect final state at once even if failures occurs.

### Apache Pulsar
<p>Apache Pulsar is open-source distributed pub/sub messaging system. It has been created by Yahoo originally and moved to Apache Software Foundation(ASF) as incubator in 2016. Graduated from incubator stage as top-level Apache project in 2018.</p>

<p>Pulsar’s some of the key features are as follows:</p>

1. **Multi Language Support:** Currently, simple client APIs support such as **Java**, **Go**, **Python** and **C++**.
1. **Multi Tenancy:** Multiple services can be supported by same Pulsar cluster through separated authentication and authorization isolation.
1. **Streaming and Queueing Support:** Pulsar supports traditional Queueuing and Streaming use-case by enabling different subscription mode: such as **exclusive**, **shared** or **failover**.
1. **Pulsar Functions:** Serverless based computing framework by providing stream-native data processing.
1. **Pulsar I/O:** Connector framework has been built on top of Pulsar Functions by moving data in and out through ecosystem.
1. **Geo Replication:** This feature provides sync/async message replication between clusters located in different geo locations.
1. **Persistent and Isolated Message Storage:** Pulsar decouples serving and storage tiers. Also, it uses [Apache Bookkeeper](https://bookkeeper.apache.org/) as message-store.


### Apache Pulsar - Flink Connector
<p>Apache Pulsar - Flink Connector brings Flink DataSet and DataStream API support to Apache Pulsar for batch and streaming use-cases. With this connector, Flink data batches and streams can be written to user-defined Pulsar topic. This article mostly focuses Pulsar-Flink DataSet API Support.</p>

<p>Pulsar - Flink Batch API currently supports the following output formats:</p>

1. **PulsarOutputFormat:** Writes Flink DataSet rows as messages in plain text to Pulsar
1. **PulsarCsvOutputFormat:** Writes Flink DataSet rows as messages in Csv format to Pulsar
1. **PulsarJsonOutputFormat:** Writes Flink DataSet rows as messages in Json format to Pulsar
1. **PulsarAvroOutputFormat:** Writes Flink DataSet rows as messages in Avro format to Pulsar

Lets have a look example:

<p>In this example Scala has been used. However, both Java and Scala examples can also be found under Pulsar Repo as follows:</p>

[Java Examples](https://github.com/apache/pulsar/tree/master/examples/flink/src/main/java/org/apache/flink/batch/connectors/pulsar/example/) <br/>
[Scala Examples](https://github.com/apache/pulsar/tree/master/examples/flink/src/main/scala/org/apache/flink/batch/connectors/pulsar/example/)

<p>In this example project, we will be creating Flink NasaMission Dataset and writing each row to Pulsar as separated message in Csv format. Lets start:</p>

##### Setup:
1. JDK v1.8
1. Scala v2.11
1. Apache Flink: v1.8
1. Apache Pulsar: v2.3.2

**1-** Apache Pulsar can be downloaded at [here](https://pulsar.apache.org/en/download/)
```yaml
$ tar xvfz apache-pulsar-2.3.2-bin.tar.gz
$ cd apache-pulsar-2.3.2
```

**2-** We need to run Pulsar in standalone mode (locally) to simplify the example:
```yaml
$ bin/pulsar standalone
```

**3-** Object model needs to be defined before creating Dataset:
```scala
/**
  * NasaMission Model
  */
case class NasaMission(id: Int, missionName: String, startYear: Int, endYear: Int)
  extends Tuple4(id, missionName, startYear, endYear)
```

**4-** Each Flink program needs to have `ExecutionEnvironment` defining application context. It provides the functions to control Flink job execution.
```scala
// set up the execution environment
val env = ExecutionEnvironment.getExecutionEnvironment
```

**5-** `PulsarCsvOutputFormat` needs to be defined by setting Pulsar `Service URL`(e.g: pulsar://127.0.0.1:6650) and `Topic Name` in order to write Flink Datasets to Pulsar topic.
```scala
private val SERVICE_URL = "pulsar://127.0.0.1:6650"
private val TOPIC_NAME = "my-flink-topic"

// create PulsarCsvOutputFormat instance
val pulsarCsvOutputFormat = new PulsarCsvOutputFormat[NasaMission](SERVICE_URL, TOPIC_NAME)
```

**6-** In this sample, Nasa Missions' data have been used to create sample dataset.
```scala
private val nasaMissions = List(
    NasaMission(1, "Mercury program", 1959, 1963),
    NasaMission(2, "Apollo program", 1961, 1972),
    NasaMission(3, "Gemini program", 1963, 1966),
    NasaMission(4, "Skylab", 1973, 1974),
    NasaMission(5, "Apollo–Soyuz Test Project", 1975, 1975))

// create DataSet
val nasaMissionsDS = env.fromCollection(nasaMissions)
```

**7-** This step aims to be processed `nasaMissionsDS` dataset by applying easy `map-filter` operations.
```scala
// map nasa mission names to upper-case
nasaMissionsDS.map(nasaMission => NasaMission(
  nasaMission.id,
  nasaMission.missionName.toUpperCase,
  nasaMission.startYear,
  nasaMission.endYear))

// filter missions which started after 1970
.filter(_.startYear > 1970)
```

**8-** After dataset is processed, it can be written to Pulsar on user defined topic.
```scala
// write batch data to Pulsar as Csv
.output(pulsarCsvOutputFormat)
```

**9-** Parallelism level can be defined to write dataset to Pulsar in parallel. However, dataset order can be changed.
```scala
// set parallelism to write Pulsar in parallel (optional)
env.setParallelism(2)

// execute program
env.execute("Flink - Pulsar Batch Csv Example")
```

**10-** Please see the complete example as follows:
[here](https://github.com/apache/pulsar/tree/master/examples/flink/src/main/scala/org/apache/flink/batch/connectors/pulsar/example/FlinkPulsarBatchCsvSinkScalaExample.scala)

**11-** Also, for verification, written messages can be tailed through terminal as follows. Otherwise, Pulsar Consumer(e.g: Java based) can be enabled.
```yaml
$ bin/pulsar-client consume -n 0 -s test my-flink-topic
```

**12-** Please find the expected results as follows:
```yaml
----- got message -----
4,SKYLAB,1973,1974
----- got message -----
5,APOLLO–SOYUZ TEST PROJECT,1975,19
```

### References
1. [Apache Pulsar](https://pulsar.apache.org/)
1. [Apache Flink](https://flink.apache.org/)
1. [Exactly Once Stream Processing with Apache Flink](https://www.ververica.com/blog/high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink)
1. [Exactly and Effectively-Once Processing Semantics](https://streaml.io/blog/exactly-once)
