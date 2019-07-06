---
layout: post
title: "Apache Pulsar Flink Connector"
category: Dev
tags: [apache pulsar, apache flink, connector, streaming, batch processing, pub-sub system, distributed systems]
date: 2019-07-07
---

<p>This article aims to show how to write Apache Flink DataSets to Apache Pulsar topic from scratch. Both Apache Flink and Pulsar projects are main components of Data Pipelines. Let' s have a quick look Apache Flink and Apache Pulsar.</p>

### Apache Flink
<p>Apache Flink is an open-source unified distributed compute engine by supporting Streaming, Batch, ML and Graph Processing use cases.</p>

<p>Flink’s some of the key features are as follows:</p>

1. `Stateful Stream Processing`: \
1. `Complex Event Processing`: \
1. `Batch Processing`: \
1. `ML`: \
1. `Graph Processing`: 

### Apache Pulsar
<p>Apache Pulsar is open-source distributed pub/sub messaging system. It has been created by Yahoo originally and moved to Apache Software Foundation(ASF) as incubator in 2016. Graduated from incubator stage as top-level Apache project in 2018.</p>

<p>Pulsar’s some of the key features are as follows:</p>

1. `Multi Language Support`: Currently, simple client APIs support such as Java, Python, C++ \
1. `Multi Tenancy`: Multiple services can be supported by same Pulsar cluster through separated authentication and authorization isolation. \
1. `Streaming and Queueing Support`: Pulsar supports traditional Queueuing and Streaming use-case by supporting different subscription mode: such as exclusive, shared or failover \
1. `Pulsar Functions`: Serverless based computing framework by providing stream-native data processing. \
1. `Pulsar I/O`: Connector framework built on top of Pulsar Functions by moving data in and out through ecosystem. \
1. `Geo Replication`: This feature provides sync/async message replication between clusters located in different geo locations. \
1. `Persistent and Isolated Message Storage`: Pulsar decouples serving and storage tiers. Pulsar uses Apache Bookkeeper as message-store \
1. `Seamless scalability out to over a million topics`: 

### Apache Pulsar - Flink Connector
<p>Apache Pulsar - Flink Connector brings Flink DataSet and DataStream API support to Apache Pulsar for batch and streaming use-cases. With this connector, Flink data batches and streams can be written to user-defined Pulsar topic. This article mostly focuses Pulsar-Flink DataSet API Support.</p>

<p>Pulsar - Flink Batch API currently supports the following output formats:</p>

1. `PulsarOutputFormat`: Writes Flink DataSet rows as messages in plain text to Pulsar \ 
1. `PulsarCsvOutputFormat`: Writes Flink DataSet rows as messages in Csv format to Pulsar \
1. `PulsarJsonOutputFormat`: Writes Flink DataSet rows as messages in Json format to Pulsar \
1. `PulsarAvroOutputFormat`: Writes Flink DataSet rows as messages in Avro format to Pulsar

Lets have a look example:

In this example Scala has been used. However, both Java and Scala examples can also be found under Apache Pulsar Repo as follows: \
[Java Examples](https://github.com/apache/pulsar/tree/master/examples/flink/src/main/java/org/apache/flink/batch/connectors/pulsar/example/) \
[Scala Examples](https://github.com/apache/pulsar/tree/master/examples/flink/src/main/scala/org/apache/flink/batch/connectors/pulsar/example/)

In this example project, we will be creating Flink NasaMission Dataset and writing each row to Pulsar as separated message in Csv format. Lets start:

##### Setup:
1. JDK v1.8
1. Scala v2.11
1. Apache Flink: v1.8
1. Apache Pulsar: v2.3.2

1- Apache Pulsar can be downloaded via https://pulsar.apache.org/en/download/
```shell
$ tar xvfz apache-pulsar-2.3.2-bin.tar.gz
$ cd apache-pulsar-2.3.2
```

2- We need to run Pulsar in standalone mode (locally) to simplify the example:
```shell
$ bin/pulsar standalone
```

3- Object model needs to be defined before creating Dataset:
```scala
/**
  * NasaMission Model
  */
case class NasaMission(id: Int, missionName: String, startYear: Int, endYear: Int)
  extends Tuple4(id, missionName, startYear, endYear)
```

4- Each Flink program needs to have `ExecutionEnvironment` defining application context. It provides the functions to control Flink job execution.
```scala
// set up the execution environment
val env = ExecutionEnvironment.getExecutionEnvironment
```

5- `PulsarCsvOutputFormat` needs to be defined by setting Pulsar `Service URL`(e.g: pulsar://127.0.0.1:6650) and `Topic Name` in order to write Flink Datasets to Pulsar topic.
```scala
private val SERVICE_URL = "pulsar://127.0.0.1:6650"
private val TOPIC_NAME = "my-flink-topic"

// create PulsarCsvOutputFormat instance
val pulsarCsvOutputFormat = new PulsarCsvOutputFormat[NasaMission](SERVICE_URL, TOPIC_NAME)
```

6- In this sample, Nasa Missions' data have been used to create sample dataset.
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

7- This step aims to be processed `nasaMissionsDS` dataset by applying easy `map-filter` operations.
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

8- After dataset is processed, it can be written to Pulsar on user defined topic.
```scala
// write batch data to Pulsar as Csv
.output(pulsarCsvOutputFormat)
```

9- Parallelism level can be defined to write dataset to Pulsar in parallel. However, dataset order can be changed.
```scala
// set parallelism to write Pulsar in parallel (optional)
env.setParallelism(2)

// execute program
env.execute("Flink - Pulsar Batch Csv Example")
```

10- Please see the complete example as follows:
[here](https://github.com/apache/pulsar/tree/master/examples/flink/src/main/scala/org/apache/flink/batch/connectors/pulsar/example/FlinkPulsarBatchCsvSinkScalaExample.scala)

11- Also, for verification, written messages can be tailed through terminal as follows. Otherwise, Pulsar Consumer(e.g: Java based) can be enabled.
```shell
$ bin/pulsar-client consume -n 0 -s test my-flink-topic
```

12- Please find the expected results as follows:
```text
----- got message -----
4,SKYLAB,1973,1974
----- got message -----
5,APOLLO–SOYUZ TEST PROJECT,1975,1975
```

### References
1. [Apache Flink](https://flink.apache.org/) \
1. [Apache Pulsar](https://pulsar.apache.org/) \
1. [Apache Bookkeeper](https://bookkeeper.apache.org/)
