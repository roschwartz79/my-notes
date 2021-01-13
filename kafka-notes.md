# Kafka notes
Taken from the intersphere, my personal learnings and [O'Reilly](https://learning.oreilly.com/library/view/kafka-the-definitive/9781492043072/ch01.html)

## What is Kafka? 
Kafka is a publish/subscribe messaging system designed to provide a durable record of all transactions. Data is stored in order, so that it can be replayed back with confidence. The unit of data within Kafka is called a message, similar to a row or a record. 

To Kafka, a message is just an array of bytes so it doesn't matter what the message is. But you can add some metadata with is referred to as a key, which is also just a byte array and can be anything. Keys are used when messages are written to partitions in a more controlled manner, so we can generate a hash of the key and then select the partition number for that message... more on this later.

Messages are written into Kafka in batches, or a collection of messages, all of which are being produced to the same topic AND partition. While the message is irrelevant to Kafka, it can be anything, we still want to have a set schema so it is easy to read and write from the topic. 

### Topics, Partitions and Clusters

Topics can be thought of as a folder in a filesystem, and messages are written to each partition in a commit log. Each partition is a single log, so a topic with multiple partitions has mutliple commit logs. Each partition can be on a different server too, so it could be scaled horizontally across multiple servers. 

There are two types of users- producers and consumers. Producers create new messages to the topic. By default the producer doesn't care what partition the message goes to but it can be directed to a specific partition. The consumer reads messages in the order in which they were produced. It keeps track of which ones it's read by using an offset, an int that keeps increasing that kafka adds to as each message is produced. Each message in a specific partition has a unique offset, so by storing the offset of the last consumed message for each partition the consumer can know where it left off, even if it gets disconnected. 

A broker is a single Kafka server, which serves as the receiver from producers and assigns offsets to them as well as servicing consumers responding to fetch requests for partitions and responding with the messages. Brokers are desinged to be apart of a cluster, and within a cluster of brokers one broker will also act as the cluster controller. This controller is responsible for a plethora of admisnistrative duties like assigning paritions ot brokers and monitoring for broker failures. All consumers and producers must be connecting to the leader. 

Data retention is a key feature of kafka, where you can store messages with confidence for a set amount of time, or until a certain size has been reached by the topic. Once the limits are reached, messages are expired and deleted so the retention configuration is a minimum amoubt of data available at any time. You can even configure individual topics with custom retention policies as it applies to your system. If you want you can configure it as log compacted which is where Kafka will retain only the last message produced with a specific key (useful for changelog type data).

### Multiple Clusters

As your deployments and data grows you'll want to have multiple clusters.
- Segregate different types of data
- Isolate for various security requirements
- Multiple datacenters can mitigate disaster recovery

With multiple datacenters, it is required for the data in each to be in both locations. Replication will not work outside of a single cluster, so a tool that is included with Kafka is called MirrorMaker is used for this purpose. It is basically a consumer and producer linked together with a queue. Messages are consumed from one cluster and produced to another and vice versa.

You can also use multiple producers, to produce to the same topic or different topic. And you can have multiple consumers consuming from the same topic. Since it is disk based retention, a consumer can fall behind or even go offline without fear of losing messages. It will just pick up where it left off. Kafka is also extremely scalable so it canbe used with a wide variety of amounts of data. You can start with a single broker as a POC and then exapand to three brokers and then to hundreds if need be. Expansions can be performed while the cluster is online so there is no impact on the system. It also means that a cluster with multiple brokers can handle an impact where a broker would go down so kakfa can continue to serve it's clients. On top of all of this it is a high performance pub/sub system. Large scale message streams can be built with subsecond message latency from producer to subscriber. 

### Inside the kakfa data ecosystem

Kafka provides the cicrulatory system for data. It carries messages between the various members of the infrastructure so the interface is consistent for all clients of the system. This makes it easy to connect producers and consumers to the system because they are no longer tightly coupled with the system.

### Use cases

So we see how great Kfaka is, where is it a good idea to use kafka? 
1. Activity Tracking
2. Messaging
3. Metrics and Logging
4. Commit Log
5. Stream Processing

Kafka was created to address data pipeline issues. It is so flexible and powerful it can be used for so many different applications.

## Setting up Kafka

To use Kafka we need a few things
- Java: Kafka is a java application
- Zookeeper: Kafka uses zookeeper to store metadata about the kafka cluster as well as consume client details. 

Setting up Zookeeper is a process. See chapter 2 of the O'Reilly Book for more details. However, we need a Zookeeper instance running on port 2181 and a Kafka Broker running in order to get started. Once these are both up and running, we can test it out. 
`/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test`

and 

`/usr/local/kafka/bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test`

We should be able to see that our topic is up and running now. We named it test, how fitting. There is also 1 partition... more on that later. Now lets produce a message to the test topic. 

`/usr/local/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test`

And... consume that message!

`/usr/local/kafka/bin/kafka-console-consumer.sh --topic test --from-beginning --bootstrap-server localhost:9092`

We can see our messages! Note that the --zookeeper flag has been replaced by --bootstrap-server. Woohoo! 

### Broker Config

There are lots of config params when we deploy kafka for any environment other than a standalone broker on a ingle server, which won't be the case everytime. The config options are:
- Broker.ID: Every broker must have an int id, and it must be unique within each cluster. 
- Port: The example config uses 9092, it can be set to any available port by changing this param.
- Zookeeper.connect: The location or storing the broker metadata is set using this param 
- Log.dirs Since kafka persists all messages to disk the log segments are stored in the directories specified in the log.dirs config.
- Num.recovery.threads.per.data.dir: Kafka uses a configurable pool of threads for handling log segments. It is used when starting, after a failure and when shutting down. By default one thread per log dir is used, but this can be changed. You can parallelize operations and recover from an unclean shutdown much quicker. 
- Auto.create.topics.enable: The default config spcifies the broker should automatically create a topic under the the circumstances when a producer starts writing to the topic, a consumer starts reading and when any client requests metadata. This might be undesireable though and can make topic creation explicit by setting this to false.

There are topic defaults that are important that can be set on a topic by topic basis. 
- Num.partitions: Default is 1
  - How to choose the Num of partitions?
    - What is the throughput
    - What is the max throughput needed
    - Adding partitions can sometimes be very tricky
    - Avoid overestimating
- Log.retention.ms: How long to retain messages in Millis. Default is 1 week (Similar to log.retention.minutes and log.retention.hours)
- Log.retention.bytes: Another way to expire messages is based on the number of bytes contained. 
- Log.segment.bytes: The size of each log segment. When this max is reached the segment is closed and a new log is opened. 
- Log.segmet.ms: the amount of time until a log is closed and new one opened. 

### Hardware selection

This doesn't apply to me as much as it might others, but still cool stuff as an EE!

The bottom line is that Kafka will run without issue on any system, until performance becomes an issue. Factors to take into account:
- Disk throughout
- Disk Capacity
- Memory
- Networking
- CPU

kafka can be used in the cloud, such as with AWS. Here you can pick all your parameters, data retention is a good place to start. In AWS... the m4 and r3 instance types are a common choice as m4 allows for greater retention periods and r3 will have better throughput but will limit the amount of data retained. 

A single kafka server works great for local dev work or poc systems but there will be significant benefits to having multiple borkers configured as a cluster. An even bigger benefit is that you can scale the load across multiple servers. 

Zookeeper stores metadata info about the brokers topics and partitions... and therefore we can have 1 zookeeper cluster for a single kafka cluster. There are lots of other concerns with getting kafka running for a production environment, but that is not applicable right now. A brief foundation of this info is good for now!


## Kafka Producers

There is a native kafka client that ships with Kafka, and you can use many developed clients with lots of different programming languages. 

To produce a message we start by creating a ProducerRecord. This ust include the Topic and Value and we optionally can add a partition and key specification. The first thing the producer will do is serialize the key and value objects to byte arrays to be sent over the network. Then the data is sent to the partitioner. If we specified a partition then the partitioner doesn't do anything and returns the partition we said. If we didn't then the partitioner will pick a partition and adds the record to the batch of recrods that will be sent to the topic and partition. When the broker receives the messages it sends a response back- if successful it return a RecordMetadata object with the topic partition and offset of the record in the partition it was written to. If it failed it will return an error message, it may retry to send the message before giving up. 

There are 3 solid ways to send a message:
1. Fire and Forget
2. Synchronous (We wait until we know it succeeded)
3. Asychronous (We don't need to wait to succeed but will callback if it failed)

These ways are shown in my kafka-playground. 

### Configuring Producers 

There are lots of config params and most are documented but the significant ones, I'll write down here.
- acks: controls how many partition replicas must receive the record before the producer can consider the write successful. acks can be 0, 1 or all. 0 means the producer won't wait for a reply from the broker, so we won't know if a message fails but this makes for high throughput. 1 means the producer gets a successful response when the leader replica recieved the message. All means that the producer receives a successful response when all in sync replicas have received the message. The most safe way to handle data but latency will be high.
- Buffer.memory: The amount of memory the producer will use to buffer messages waiting to be sent to brokers.
- Compression.type: By default messages are not compresssed but we can set it to snappy, gzip or lz4 where the compression algorithms will be applied before it's sent to the brokers. 
- Retries: How many times the producer will attempt to retry the sending of a message before giving up. Not all errors will be retried.
- Batch.size: The amount of memory in bytes that will be used for each batch, when the batch is full the messages get sent. This doesn't mean that messages only get sent when batches are full though. 
- Linger.ms amount of time to wait for additional messages before sending current batch. Default is 0 (e.g. not waiting)
- client.id: Any string that identifies messages sent from the client. 
- in.flight.request.per.session: When set to 1, if the first batch fails, the second batch will not go until the first batch is succesful. If the second batch went when the first failed that could reverse the order!

There are lots of others so check the docs!

### Serializers

The built in serializers work great, but we may need more. There are lots of good general serializers, or we can write our own!

There can be lots of issues with writing our own serializers, so it is usually best to use a general serializer if we can get away with it. One of the serializers we can use is Apache Avro. The cool part about avro is that when the application writing messages changes schemas the applications reading the data can continue processing messages without any changes. If the reader tries to deserialize a key that is no longer present in the schema it will simply return as null instead of breaking. However:
- The schema used for writing the data and the schema expected by the reader must be compatible. 
- The deserializer will need access to the schema that was used when writing the data even when it is different than the schema expected by the application that accesses the data. 

We need to access the schema from the reader, we do this in a schema registry, which is not part of Kafka but there are lots of options for this. Then we can store all of the schemas ever used with a schema id in the schema registry and send messages through kafka with the schema id, so the deserializer can deserialize the messages with the correct schema from the registry.

We can also send generic Avro objects rather than the generated ones, but it requires some more work. 

### Partitions
