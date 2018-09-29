## "The Log"

This is an [article from Jay Kreps on Linkedin's engineering blog](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying), that was referenced in the site docs for Apache Kafka.

### What is a log?

A series of entries that are assigned a unique log entry number. This decouples from physical time (see Lamport, etc). Ordering is defined by the log entry number (presumably this maps to something like the natural numbers).

#### Logs in databases

Logs are used in databases to keep a record of what happened (e.g. "insert a row with these contents into this table"), and then the table is the manifestation of that action. The log represents an authoritative source if there is a crash.

ACID transactions: atomic, consistent, isolated, durable

Logs eventually became a way to replicate databases: ship the log to replica databases which act as followers. 

#### Logs in distributed systems

How to agree on order updates is a core design problem in distributed systems. 

The State Machine Replication Principle
```
If two identical deterministic processes start in the same state and get the same inputs in the same order, they will produce the same output...
```

Physical logging (logging the contents of the row that has changed) vs. (logging the SQL commands that led to the changes)

Two broad approaches to replication:
- "State Machine Model" -> all instances are active where there is a log of incoming requests and each replica processes each request (Kafka: size of the consumer group is 1).
- "Primary-backup model" -> one replica is the leader that processes all of the requests, and then logs out changes to state, which the followers subscribe to (not really a Kafka analogue)

#### Changelog 101: tables and events are dual

Tables can be developed from a changelog, and vice versa. They are isomorphic to the state of the system. 

### Data integration

About making all of the data accessible to all services and systems. ETL is a specific subset that, but more generally applicable to flows.

Two things that are making this hard:
- Event data firehose: these things are all orders of magnitude too big for normal database use cases.
- Specialized data systems

#### Log-structured data flow

Each data source can be modeled as its own log. Logs also add a buffer to make production detached from consumption.

#### At LinkedIn

They have a lot of data systems. Various observations here: 
- pipelines are valuable
- reliable data loads required support from the pipeline.
- Low data coverage

What lead to building Kafka was the development of a single log sources for events, and then production services could read off of that log. 

#### Relationship to ETL and data warehouse

In the previous model, data producers are not concerned with making data easily mungible, etc. But now, the central pipeline, lets the warehouse be responsible for reading a section of the log, and then updating a specific transformation for that part of the warehouse.

Having a central log also makes it easier to adopt additional consumers on the data for alternate forms of data search that might have different operational requirements.

Cleanup and transformation options for new data systems
- By producer before adding to the company log
- As a transformation on the log
- As part of load into a new system 

Recommended model: do clean up before publishing the data onto the log by the publisher of that data.

Toy example: a view of page - how many different systems want to subscribe to this event, and how complex is it if these systems all now about each other?

#### Building a scalable log

Kafka tricks: partition the log, batch reads and writes, and avoid needless copies.

Lack of a global order across partitions, but this was not found to be a major limitation. The order from a single partition is guaranteed, which tends to be more important if all the actions of a single user (for example) also occur on the same partition.

### Logs & real-time stream processing

Stream processing as infrastructure for continuous data processing. There is a clear match between how a data is collected (stream vs. batch) and how it is processed (stream vs. batch).

#### Data flow graphs

Sometimes jobs involve multiple logs, with multiple stages. It's important that the log allows for buffering, otherwise the graph could grind to a halt.

#### Stateful real-time processing

Streams can maintain a local state across a single partition and achieve tolerance by keeping its state in a local table.

This feels sort of like [Wallaroo](https://www.wallaroolabs.com/) is doing.

#### Log compaction

We cannot keep the total log for all time. Instead we can figure out the latest update to a key and keep the state ("table" from the table and log duality), while throwing away earlier records.

### System Building

#### Unbundling?

There is currently an explosion of types of data systems. One possible simplification of systems: moving lots of little instances to a single big cluster.

Three direction for this to go:
- status quo and the continued separation of systems.
- re-consolidation into a super-system, that superficially looks like a relational database. 
- building blocks via various open source components. 

#### The place of the log in system architecture

The log can assume the implementation burden of a lot of guarantees that individual system components no longer have to worry about, like:
- handling data consistency
- data replication between nodes
- commit semantics
- subscription feed
- restoring or creating new replicas
- re-balancing between nodes. 

Logs -> serving nodes: this layer allows to decide what indexes are important. 

Log centric infrastructure stacks: provides data streams for other systems.

### The end
- There are a bunch more useful resources in here, that aren't strictly related to logs. But I'd read them.

















