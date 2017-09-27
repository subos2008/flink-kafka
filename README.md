# Flink, IntelliJ and Scala

## Setup

Install as per the [Flink quickstart](https://ci.apache.org/projects/flink/flink-docs-release-1.3/quickstart/setup_quickstart.html)
or
<https://ci.apache.org/projects/flink/flink-docs-release-1.3/quickstart/scala_api_quickstart.html>

I cloned the repo for the example flink job. Recommendation is to use g8.

Import the Flink project into IntelliJ. Selecting the installed SDK was non-obvious - it pops open a file selection page in the home directory - but the installed SDKs are listed if you click up the top.


You can run your flink jobs locally in IntelliJ, see the README in the Flink project:

> In order to run your application from within IntelliJ, you have to select the classpath of the 'mainRunner' module in the run/debug configurations.
Simply open 'Run -> Edit configurations...' and then select 'mainRunner' from the "Use classpath of module" dropbox. You can leave this set to mainRunner when compiling jars for flink; it only affects the IDE's behaviour.


Set `main` in `build.sbt` to point to the object you want to execute.

For scala notes see Evernote.

Note the cluster I was using had Flink's logging and stdout not being captured in the Flink UI, but I could see stdout via the k8 pod's logs.

## Build the .jar

Quick info dump on sbt [here](http://xerial.org/blog/2014/03/24/sbt/).

Running build in IntelliJ doesn't seem to create the jar, run the following on the command line:

```bash
$ sbt clean assembly
```

Note this uses the sbt-assembly plugin to create fat .jars (that contain all dependencies.)

You can install this plugin by adding the following file:

```
$ cat project/assembly.sbt
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.5")
```

## Run on Flink

Select "Submit new job"

Upload the jar file from `./target/scala-2.11/Flink Project-assembly-0.1-SNAPSHOT.jar`. You need to click the green upload button to kick it off.

See the note here: <https://github.com/ziyasal/scream-processing> about fat jars and upping the size of jar files you can upload if you are hosting flink in Kubernetes.

Click the tick box next to the job you want to run and then click "Submit".

I was seeing the following error:

> org.apache.flink.client.program.ProgramInvocationException: The program plan could not be fetched - the program aborted pre-maturely.

This was because I wasn't using stream processing functions in my code - it was basically just a scala program - so it compiled to nothing from flink's perspective.

I'm having trouble finding the stdout for the job to verify it ran... There are stdout tabs on a few screens.

## Connecting Flink to Kafka

1. Using Java: <https://data-artisans.com/blog/kafka-flink-a-practical-how-to>. 
2. Example scala code: <https://github.com/mkuthan/example-flink-kafka/blob/master/src/main/scala/example/flink/FlinkExample.scala> which includes some kind of windowing code. 
3. Example with EventTime windowing: <https://stackoverflow.com/questions/45479455/flink-scala-join-between-two-streams-doesnt-seem-to-work>

### With sbt and scala

To add `flink-connector-kafka` as a dependency add the following line to `flinkDependencies` in build.sbt.

```
"org.apache.flink" % "flink-connector-kafka-0.10_2.11" % flinkVersion,
```

As per [this page](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/connectors/kafka.html) the `0.10_2.11` versioning refers to kafka version `0.10.x` - which is required for messages with timestamps. It has the class name `FlinkKafkaConsumer10`.

A basic dump of kafka to stdout [src link](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/connectors/kafka.html) and [alt src link](https://stackoverflow.com/questions/45479455/flink-scala-join-between-two-streams-doesnt-seem-to-work) (the first one wouldn't build for me)

```scala
package org.example
import java.util.Properties
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010
import org.apache.flink.streaming.util.serialization.SimpleStringSchema
import java.util.Properties

/* Dumps a kafka topic to stdout */
object KafkaTopicDump {
  def main(args: Array[String]) {

    val kafka_broker_endpoint = "kafka-broker.kafka:9092"
    val kafka_topic = "topic"
    val kafka_consumer_group = "kafka_topic_dump"

    // set up the execution environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    val properties = new Properties();
    properties.setProperty("bootstrap.servers", kafka_broker_endpoint);
    properties.setProperty("group.id", kafka_consumer_group);

    val consumer = new FlinkKafkaConsumer010[String](kafka_topic, new SimpleStringSchema(), properties)

    val stream = env.addSource(consumer)
    stream.print
    env.execute
  }
}
```

NB I was getting `could not find implicit value for evidence parameter of type org.apache.flink.api.common.typeinfo.TypeInformation[String]`. Breaking things out onto separate lines may have fixed it. 


From the docs:

> The constructor accepts the following arguments:
> 
> 1. The topic name / list of topic names
> 2. A DeserializationSchema / KeyedDeserializationSchema for deserializing the data from Kafka
> 3. Properties for the Kafka consumer. The following properties are required:
>   1. “bootstrap.servers” (comma separated list of Kafka brokers)
>   2. “zookeeper.connect” (comma separated list of Zookeeper servers) (only required for Kafka 0.8)
>   3. “group.id” the id of the consumer group

### Dumping kafka topics from inside kubernetes

Given a kafka **pod** called `tc-dp-kafka-kf-0` and a zookeper service named `tc-dp-kafka-zookeeper` you can list the kafka topics with:

```bash
$ kubectl -n kafka exec -ti tc-dp-kafka-kf-0 -- ./bin/kafka-topics.sh --zookeeper tc-dp-kafka-zookeeper:2181 --list
```

NB: this takes a minute or two to run.

To consume a topic to a command line:

```bash
$ kubectl -n kafka exec -ti tc-dp-kafka-kf-0 --  ./bin/kafka-console-consumer.sh --zookeeper tc-dp-kafka-zookeeper:2181 --topic my-topic --from-beginning
```

To create a topic:

```
./bin/kafka-topics.sh --zookeeper=tc-dp-kafka-zookeeper.kafka:2181 --create --topic tc-squeak-spark-output --partitions=5 --replication-factor=3 # see below for note on partitions
```

Note the `kafka-topics.sh` script is in `./bin/` rather than `/usr/bin`.

To post to a topic:

```
./bin/kafka-console-producer.sh --broker=localhost:9092 --topic tc-squeak-spark-output
```

#Kafka Topics and Partitions

TL;DR: just have lots of partitions. More is no problem, less will limit your maximum parralelisation.

The number of partitions sets the maximum parallelisation. Say for example you have a consumer group with 10 stateless microservices that process messages for a particular topic. If your number of the partitions for that topic is less than 10 then some of your microservices will sit idle. The result would still be correct.

Multiple consumers / consumer groups can listen to a partition and all will get a copy of every message.

The (consumer group, partition) pairing is where the current offset for the read head is stored.

Partitions are split accross and assigned to brokers on creation. Assignments are no automatically re-balanced if you change the number of partitions. Having more partitions makes node failure faster as the topic history is spread accross partitions (brokers) with less data on each broker. (c.f. replication-factor). NB replication-factor has a max of #brokers.


## Local kafka, flink docker-compose

Ziya's guide: <https://github.com/ziyasal/scream-processing

### Default posts on localhost

- --zookeeper=localhost:2181
- --zookeeper tc-dp-kafka-zookeeper:2181
- kafka 

This doesn't currently work as the `confluent/kafka` image is a year out of date and the images expect to be deployed in kubernetes by default.

```docker-compose
# Set the FLINK_DOCKER_IMAGE_NAME environment variable
# to override the image name to use

version: "2.2"
services:
  zookeeper:
    image: confluent/zookeeper
    ports:
    - "2181:2181"
    environment:
    - "zk_id=1"
  kafka:
    image: confluent/kafka
    depends_on:
    - zookeeper
    ports:
    - "9092:9092"
    links:
    - "zookeeper"
    environment:
    - "KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181"

  elasticsearch:
    image: elasticsearch:2.4.5
    ports:
    - "9200:9200"
    - "9300:9300"
    environment:
    - "cluster.name=elasticsearch"
    - "bootstrap.memory_lock=true"
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 1g
    volumes:
    - esdata1:/usr/share/elasticsearch/data

  kibana:
    image: kibana:4.6.4
    ports:
    - "5601:5601"
    environment:
    - "ELASTICSEARCH_URL=http://elasticsearch:9200"
    depends_on:
    - elasticsearch
    links:
    - "elasticsearch"

  jobmanager:
    image: ${FLINK_DOCKER_IMAGE_NAME:-flink}
    ports:
    - "8081:8081"
    command: jobmanager
    environment:
    - "JOB_MANAGER_RPC_ADDRESS=jobmanager"

  taskmanager:
    image: ${FLINK_DOCKER_IMAGE_NAME:-flink}
    expose:
    - "6121"
    - "6122"
    depends_on:
    - jobmanager
    command: taskmanager
    links:
    - "jobmanager"
    environment:
    - "JOB_MANAGER_RPC_ADDRESS=jobmanager"

  my-service:
    image: foo:latest
    depends_on:
    - kafka
    - zookeeper
    links:
    - "kafka"
    - "zookeeper"

volumes:
  esdata1:
    driver: local
```