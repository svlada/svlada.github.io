---
title: 'Durable request reply'
description: Interprocess communication in distributed systems using Kafka.
date: 2020-01-29
tags:
  - software architecture
  - microservices
  - kafka
layout: layouts/post.njk
permalink: "microservices-durable-request-reply/index.html"
---

The article explains a simplified distributed system based on the message passing with a durable request-reply paradigm. The examples given are based on the Kafka, which is distributed commit log implementation. [Reactive Manifesto](https://www.reactivemanifesto.org/glossary) explains the main characteristics of a message-driven architecture.

The following system design diagram shows a bird's-eye view of the exemplary microservice architecture:

![microserices](/img/microservices-durable-request-reply/microservices.png)

Each of the following sections describes the important system components.

## API Gateway

The API Gateway is an infrastructural pattern for interfacing external clients with internal services. This component can be implemented using one of the non-blocking frameworks (elixir, spring-reactive, node.js, zuul 2, etc.). Almost every cloud provider like Amazon or Google offers API Gateway as a managed service. The development team should select the appropriate tool based on project budget and architectural decisions.

The responsibilities of an API gateway can vary from system to system. The most common ones are:

1. Request routing
2. Rate limit/throttling
3. Security 
   1. SSL termination
   2. Token translation
4. Logging
5. Caching
6. Load balancing
7. API monitoring and analytics
8. Resource aggregation

The important part of the API Gateway to focus on is the Request router. Request router routes API requests to internal services. Introducing the facade layer in front of services improves the isolation of each service. This means that the contract with external clients can remain unchanged while internal services evolve. Services should be configured to communicate through message passing. Messages are commands that are dispatched to services.

## Request handling flow

The overall request handling flow is described as follows:
1. **Accepting API request** is the first step in the request routing flow. The next step is to create an in-memory request descriptor. Request descriptor is a data structure holding: correlation id and Http request handle. Correlation id is used to uniquely identify the Http request. Http request handle is a simple promise object use to send the response back to the client.  This pair can be stored in the Map for fast lookups.
2. **Creating command**. The command has two parts: payload and headers. The payload is extracted from the incoming request and headers are populated with origin details: communication type (synchronous, asynchronous) and reply-to location (Rest URL or reply topic name).
3. **Sending command to a service**. The command is serialized to Kafka record and sent to an appropriate request topic on a Kafka broker.
4. **Command handling**. Internal services are consuming commands from designated request/reply Kafka topics and execute appropriate business logic.
5. **Sending Reply command to the originator**. After the service has finished with processing the command, the Reply command is sent back to the originator (e.g. Request router or another service).
6. **Flush Http response**. Replier interface (i.e. one of the Request Router components) handles service replies and sends Http responses back to the clients. Appropriate client Http handle is selected based on the correlation id extracted from the Reply command (check Accepting API request step for more details). This interface is typically closed for public clients and open to internal services.

Using a non-blocking approach provides the ability to handle a higher number of Http connections with fewer resources, thus increasing system availability with a benefit of greater system elasticity.

The next section describes a durable request/reply mechanism for inter-process communication.

## Durable request reply flow

Internal services can use two types of communication: **synchronous** or **asynchronous**.

The majority of development teams favor synchronous (i.e. Rest) communication between internal and external services. Rest communication between internal services is troublesome as it imposes the following problems:

- Reduces availability.
- Introduces complexity when joining data owned by different services.
- Handling cascading failures and Http retries.
- Tight coupling between services.

Reliable message exchange between internal services should be asynchronous, durable and transactional. To fulfill these characteristics the following patterns are used:
- **Commit log**
  - Kafka serves as a backplane for durable and asynchronous message exchange. 
- **Transactional outbox and log mining**
  - Solves the problem of sending commands as a part of a business transaction. The common use case is to store data into the local database and emit a command or event to Kafka or to invoke Rest API. Debezium is an excellent tool that can be used for transaction log mining.

To implement a durable request/reply mechanism, one request and reply topic must be created for each of the internal services. Developers should be aware of how many partitions are allocated per service. The number of partitions should be carefully designed, having in mind the scalability needs of each service. The durable request/reply flow is presented with the steps listed below:

- **Service A (Kafka Producer): Sending command to Service B**
  - Set the following message headers: reply-topic-id of Service A and correlation-id.
  - Serialize and post a message to request-topic-id of Service B.
- **Service B (Kafka Consumer): Handle command**
  - Deserialize message.
  - Process command and send ack/nack back.
  - Send reply command to reply-topic-id of Service A.
- **Consumer responsibilities: Reply topic**
  - Extract correlation-id from the Reply command.
  - Use correlation-id to find Http connection handler. Note that the Http connection handler can run on a separate node or be a part of API Gateway.

When solutioning design for inter process communication, it's advisable to follow all or nothing approach (i.e. atomicity). This pattern is well known and used for decades in database servers. The good definition is the one from wikipedia:

> _All the write operations within a transaction have an all-or-nothing effect, that is, either the transaction succeeds and all writes take effect, or otherwise, the database is  brought to a state that does not include any of the writes of the transaction._

Common use case is a need to commit business transaction against local database and reliably dispatch command to a Kafka broker. Command should be dispatched only if transaction suceeds. The following are possible approaches for tackling this problem:

### Use KafkaProducer to send command to a broker inside of business logic transactional context

```java
@Transactional
fun businessTransaction() {
  saveToDatabase()
  dispatchCommandToKafka()
}
```

**Pros**

1. Easy to implement.

**Cons**

1. Network failures can cause rollback of local transaction.
2. Code is occuping expensive database connection for non-db work accross network.

### Use KafkaProducer to send command to a broker outside of business logic transactional context

```java
@Transactional
fun businessTransaction() {
  saveToDatabase()
}

fun methodA() {
  saveToDatabase()
  dispatchCommandToKafka()
}
```

**Pros**

1. Easy to implement.

**Cons**

1. Leads to error-prone code needed to check if local transaction succeeded or not.
2. In case of service or Kafka broker failures no way to have a strong guarantee that command will be delivered.

### Proper solution: Use transactional outbox

Commands are persisted to outbox table in the same transaction with business logic. Debezium, which is a transaction log miner, picks up the record and pushes it to Kafka.

One might argue that we don't need Debezium and that Kafka Connect can be used to query for commands from the Outbox table. Big drawback of this approach is possibility to skip some commands due to transaction isolation. If you don't want to lose any commands go with transactional log miner.

The following image shows a transactional outbox pattern:

![microserices](/img/microservices-durable-request-reply/request-reply.png)

**Pros**

1. Atomicity
2. Strong guarantees that if local transaction succeeded command will be eventually delivered to Kafka broker.

**Cons**

1. Another component in system to manage (e.g. Debezium)

## Sagas
Handling business transactions that span different services is done through orchestration and usage of the Saga concept defined by Garzia in early 1960. Transactional outbox is a useful pattern used to implement orchestration using Sagas.

How to handle and implement distributed transactions using Sagas will be explained in the next article. Stay tuned!

References

- [Reliable Microservices Data Exchange With the Outbox Pattern](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)
- [Missing Updates with Timestamp+incrementing mode in Postgres](https://github.com/confluentinc/kafka-connect-jdbc/issues/172)
- [Sagas](http://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)
- [Request-Reply](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3435995/)
