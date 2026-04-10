# Databases

## General

In an application with heavy write pressure, say an e-commerce on Black Friday, closing several purchases per second, you should program your application to use **queues**. Every new purchase enters the queue and gets written to the database as capacity allows. You should never just increase the number of web application servers and let everyone connect to the same database at the same time — that will only increase contention and in the end nobody can write.

It's better for everyone to enter a queue and let only the queue manage the database. Reads use Redis caches, which takes the search load off the database.

The major bottleneck in a web application is the database, because even with replication and sharding, you will always have contention due to ACID guarantees.

## NoSQL Databases

Unlike a relational database, most are neither a transactional database nor an analytical database where you can run queries like SQL with any criteria at any time.

Searches in NoSQL databases need to be **planned and indexed in advance**. NoSQL databases tend to be databases that run in clusters, with shards, meaning no single server has all the data you need at the same time, and if your search is poorly planned, it will have to scan across all nodes in the network, which costs processing and time — a lot of time.

## Relational Databases

In a relational database, what costs the most in terms of performance is writing, more than reading. Writes need to have ACID guaranteed: **atomicity, consistency, isolation, and durability**. Written data must be guaranteed to be on disk so that an unexpected crash doesn't cause data loss.

Multiple connections writing at the same time cannot step on each other's toes — you need a concurrency scheme like **multiversion concurrency control or MVCC**, which most databases use today. And you need **transaction logs** to be able to rollback or return to the original state of the data if a multi-operation transaction fails midway.

## 1. MongoDB

It is a NoSQL database. All operations are done in RAM, which is what guarantees its performance. But this means that if the machine crashes before RAM has a chance to flush to disk, **you can lose data**.

You can configure it to guarantee more durability. For example, today when you send a write to Mongo it returns OK before having actually written. Mongo's OK is more like "alright, I received the order to write, I'll write when I can." This is different from a relational database that only returns OK if the data was guaranteed to be written.

You can configure Mongo to return OK only if a replica master instance has guaranteed to receive the data. But then you lose the performance that was precisely why you chose Mongo in the first place.

## 2. Cassandra

A NoSQL wide column store. Despite having surface-level similarities with relational databases — especially in schemas and CQL which is similar to SQL — it has absolutely nothing in common. They are like **multi-dimensional key-value stores with column families** and were built to be highly distributed databases, with multiple nodes in a cluster, preferably across multiple different regions.

Cassandra **was not made to be used on a single instance**, like a blog database. Its sweet spot is running in a cluster, with multiple nodes, and with heavy writes.

## 3. Redis

In practice most of us will always end up using a **combination of a relational database with a cache**, using something like Redis. In this configuration Redis can be a bit more relaxed because the right approach is to have data queried from Redis and whatever isn't there you load from the relational database and write to Redis as needed.

Therefore Redis can be rebuilt from scratch even if there's a crash and you need to tear everything down. And you use Redis to keep things that are expensive to calculate and write in a relational database like **aggregations**, things with counters, averages, and other metrics that aggregate multiple rows from the database.

## Performance Tips

1. **Controlled denormalization** — The more normalized the table, the slower it will be as it demands more from the database. E.g.: instead of creating a STATES table (Sao Paulo, Minas Gerais), the state abbreviation and name can be saved directly in the user table. This will reduce joins and increase performance.

2. **Cache** — Add caching to avoid hitting the database as much as possible.

3. **Queues for writes** — Program the application to put database write requests into a queue (like RabbitMQ), to avoid bottlenecks and timeouts for the user.

4. **Full-text search** — When you need to do text search and show results by relevance (like e-commerce search), the most correct and performant approach is to use something like **ElasticSearch**.

5. **Indexes** — Create the necessary indexes, but don't create too many as it can hurt performance (each index needs to be updated on every write).

---

[← Previous: Entity Framework](02-entity-framework.md) | [Next: Query Optimization →](04-otimizacao-de-queries.md) | [Back to index](README.md)
