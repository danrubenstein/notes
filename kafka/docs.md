## Apache Kafka

### Introduction

Taken from this [introduction page](https://kafka.apache.org/intro).

Kafka is a streaming platform, which means that
1. It does pub-sub for records
2. It stores streams of records with fault tolerance.
3. It processes these streams of records as they occur (this reads a lot like there is some notion of ordering).

Kafka is used typically for
- building data streaming pipelines
- building streaming applications

- Kafka is run with a cluster model, which stores streams of records in categories (topics). Each record consists of a key, a value, and a timestamp.

- Kafka has four key APIs:
- Producer API to allow an application to publish a stream of records to a topic.
- Consume API to allow an application to subscribe to a topic and process the records produced thereby.
- Stream API to allow an application to act as a stream processor -> take in from a topic and then push out to another topic.
- Connector API to allow for running producers and consumers that connect topics to existing applications.

Kafka is written over the JVM, and communication is done via TCP.

#### Topics and logs

A topic is category to which records get published. Kafka topics are multi-subscriber (topics can have multiple consumers to its data). 

Each topic consists of a partitioned log: ordered and immutable records sequence. Each record has an id number (_offset_).

The cluster persists the record for a configurable retention period. However, Kafka performance is O(1) for data size (so they claim).

The only metadata retained per consumer is the position of the consumer in the log. The log can move in any direction

#### Distribution

Log partitions are distributed over the servers in the Kafka cluster. Partitions can be replicated across servers to improve fault tolerance.

Each partition has a server that is the "leader" and the rest are "followers".

#### Geo-Replication

This exists.

#### Producers

They publish data to topics, and choosing what record to assign which partition in that topic. 

#### Consumers

Consumers label themselves with a _consumer group name_. Consumer processes can all have the same consumer group name (in which case a topic is load balanced), or all have different names (in which case a record is pushed to all consumer processes).

Topics generally have a small number of consumer groups.

Kafka provides total order for records within a partition, and not between partitions.

#### Multi-tenancy

This exists.

#### Guarantees

(I assume that @aphyr has some thoughts on this - oh look, [he does.](https://aphyr.com/posts/293-jepsen-kafka))

- Messages sent by one producer to a topic partition are appended in the order that they are sent (this seems to follow naturally from performance characteristics of TCP).
- A consumer instance sees records in the order they were stored in the log. 
- If a topic has replication N, up to N-1 server failures can occur without losing records in the log. 

#### Kafka as a messaging system

Two classic models of messaging:
- queuing: a pool of consumers reads from a server and a record goes to at most one consumer. Allows for the processing of data over lots of instance (parallelism). But aren't multi-subscriber (queue can be read at most once). 
- publish - subscribe: allows the broadcasting of data to multiple processes, but doesn't allow scaling since a message goes everywhere.

Consumer group abstracts these problems. Queue in that a topic can have its work divided over the collection of processes, but publish subscribe because multiple consumer groups can subscribe to a topic. 

Open question: _How does a consumer group name subscribe to a topic?_

Kafka also has stronger order guarantees: parallelism through partitions allows ordering to be guaranteed within the partition, and by mapping partitions to a single instance within the consumer group, each instance is getting an ordered stream.

##### Kafka as a storage system

(Aside: the sounds like a GDPR nightmare if your retention configuration is longer than the GDPR takeout request timing.)

Kafka can be viewed as a distributed file system for commit storage. 

(Aside: I do not know why you want to do this, but I suppose you can. This feels like very blockchain-esque, before blockchains were cool.)

##### Kafka for stream processing

This is the full integration of the ideas behind consumers and producers. This API promotes the idea that applications should be able to create from them. 

See this as a state machine for event records. 

#### Putting the pieces together

Kafka allows you to treat past and future data similarly - the same application can process stored data but rather than stopping when it hits the last record, it can keep going with future data.

### Use cases

Taken from these [site docs](https://kafka.apache.org/uses).

#### Messaging

Relatively low throughput, but need low end to end latency and strong durability.

Listed alternatives: RabbitMQ and ActiveMQ

#### Website Activity Tracking

Site activity published to central topics with one topic per activity per activity type. High volume!

#### Metrics

Operational data, including aggregating statistics. I'm not sure how you do this without aggregating data between partitions. Something like [Veneur](https://github.com/stripe/veneur) comes to mind here. 

#### Stream Processing

Processing pipelines with multiple stages. Kafka Streams API seems to be designed to support this use case.

#### Event Sourcing

State changes are represented as a sequence of records.

#### Commit log

No further notes here. But enjoy [this cartoon](https://xkcd.com/2030/).






