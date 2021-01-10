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

## Kafka Producers
