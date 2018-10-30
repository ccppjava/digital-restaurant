## Application/s layer

This is a thin layer which coordinates the application activity. It does not contain business logic. It does not hold the state of the business objects

We have created more 'web' applications (standalone Spring Boot applications) to demonstrate the use of different architectural styles, API designs and deployment strategies by utilizing components from the domain layer in different way:

**Monolithic**

 - [Monolith 1](#monolith-1-http-and-websockets-api-by-segregating-command-and-query) 
    - **HTTP and WebSockets API** by segregating Command and Query
    - we don't synchronize on the backend
    - we provide WebSockets for the frontend to handle async nature of the backend
 - [Monolith 2](#monolith-2-rest-api-by-not-segregating-command-and-query)
    - **REST API** by not segregating Command and Query
    - we synchronize on the backend side
 - [Monolith 3](#monolith-3-stomp-over-websockets-api-we-are-async-all-the-way)
    - **WebSockets API**
    - we are async all the way

**Microservices**

 - [Microservices 1](#microservices-1-http-websockets-apache-kafka)
    - **HTTP and WebSockets API** by segregating Command and Query
    - we don't synchronize on the backend
    - we provide WebSockets for the frontend to handle async nature of the backend
    - we use Apache Kafka to distribute events between services
    - we use Spring Cloud discovery and registry service (Eureka) to distribute commands between services (bounded contexts)
 - [Microservices 2](#microservices-2-rest-rabbitmq)
    - **REST API** by not segregating Command and Query
    - we synchronize on the backend side
    - we use RabbitMQ to distribute events between services (bounded contexts)
    - we use Spring Cloud discovery and registry service (Eureka) to distribute commands between services (bounded contexts)
 - [Microservices 3](#microservices-3-websockets-axonserver)
    - **WebSockets API**
    - we are async all the way
    - we use [AxonServer](https://axoniq.io/product-overview/axon-server) to route and distribute all type of messages (commands, events, queries) between services (bounded contexts)
          
### Monolith 1 (HTTP and WebSockets API by segregating Command and Query)

Source code: [https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith)

A recurring question with CQRS and EventSourcing is how to put a synchronous HTTP front-end on top of an asynchronous CQRS back-end.

In general there are two approaches:

 - **segregating Command and Query** - resources representing Commands (request for changes) and resources representing Query Models (the state of the domain) are decoupled
 - **not segregating Command and Query** - one-to-one relation between a Command Model resource and a Query Model resource
 
 This application is using the first approach ('segregating Command and Query') by exposing capabilities of our 'domain' via the HTTP/REST API components that are responsible for
 - dispatching commands - [CommandController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith/src/main/kotlin/com/drestaurant/web/CommandController.kt)
 - querying the 'query model' (materialized views) - [Spring REST repositories](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith/src/main/kotlin/com/drestaurant/query/repository)

**There is no one-to-one relation between a Command resource and a Query Model resource. This makes easier to implement multiple representations of the same underlying domain entity as separate resources.**


[Event handler](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith/src/main/kotlin/com/drestaurant/query/handler) is a central component.
It consumes events, and creates 'query models' (materialized views) of aggregates.
This makes querying of event-sourced aggregates easy.

Event handler is publishing a WebSocket events on every update of a query model. 
This can be useful on the front-end to re-fetch the data via HTTP/REST endpoints. 

Each event handler allows 'reply' of events. Please note that 'reset handler' will be called before replay/reset starts to clear out the query model tables.
[AdminController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith/src/main/kotlin/com/drestaurant/web/AdminController.kt) expose endpoints for reseting tracking event processors/handlers.


### Monolith 2 (REST API by not segregating Command and Query)

Source code: [https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest)

This application is using the second approach (**not segregating Command and Query**) by exposing capabilities of our 'domain' via the REST API components that are responsible for
 
 - dispatching commands - [CommandController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest/src/main/kotlin/com/drestaurant/web/CommandController.kt)
 - querying the 'query model' (materialized views) - [Spring REST repositories](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest/src/main/kotlin/com/drestaurant/query/repository)


**We create one-to-one relation between a Command Model resource and a Query Model (materialized view) resource.**
Note that we are utilizing Spring Rest Data project to implement REST API, and that will position us on the third level of [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html)

[Event handler](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest/src/main/kotlin/com/drestaurant/query/handler) is a central component. It consumes events, and creates Query Model / materialized views of aggregates.
Additionally, it will emit 'any change on Query Model' to Axon subscription queries, and let us subscribe on them within our [CommandController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest/src/main/kotlin/com/drestaurant/web/CommandController.kt) keeping our architecture clean.

Each event handler allows 'reply' of events. Please note that 'reset handler' will be called before replay/reset starts to clear out the query model tables.
[AdminController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-rest/src/main/kotlin/com/drestaurant/web/AdminController.kt) expose endpoints for reseting tracking event processors/handlers.

Although fully asynchronous designs may be preferable for a number of reasons, it is a common scenario that back-end teams are forced to provide a synchronous REST API on top of asynchronous CQRS+ES back-ends.


### Monolith 3 (STOMP over WebSockets API. We are async all the way)

Source code: [https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-websockets](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-websockets)


The WebSocket protocol (RFC 6455) defines an important new capability for web applications: full-duplex, two-way communication between client and server. It is an exciting new capability on the heels of a long history of techniques to make the web more interactive including Java Applets, XMLHttpRequest, Adobe Flash, ActiveXObject, various Comet techniques, server-sent events, and others.

This application is utilizing STOMP over Websockets protocol to expose capabilities of our 'domain' via components:
 
 - [WebController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-websockets/src/main/kotlin/com/drestaurant/web/WebController.kt)
 - [Event handler](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-websockets/src/main/kotlin/com/drestaurant/query/handler) is a central component. It consumes domain events, and creates 'query models' (materialized views) of aggregates. Aditonally, our event listener is publishing a WebSocket messages to topics on every update of a query model.
 - [AdminController](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-monolith-websockets/src/main/kotlin/com/drestaurant/web/AdminController.kt) expose endpoints for reseting/replying tracking event processors/handlers.  Each event handler allows 'reply' of events. Please note that 'reset handler' will be called before replay/reset starts to clear out the query model tables.



### Microservices 1 (HTTP, Websockets, Apache Kafka)

Each [microservice](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-microservices):

 - has its own bounded context,
 - has its own JPA event store (we are not sharing the JPA Event Store between service)
 - we distribute events between them via Apache Kafka (we do not use Kafka as event(sourcing) store)
 - and we distribute commands (Command Bus) by Spring Cloud discovery and registry service (Eureka)

#### Apache Kafka & event messages

Apache Kafka is a distributed streaming platform. It is used to route and distribute `events`.

##### Order of events (kafka topics & partitions)

The order of events matters in our scenario (eventsourcing).
For example, we might expect that a customer is created before anything else can happen to a customer.
When using Kafka, you can preserve the order of those events by putting them all in the same Kafka **partition**.
They must be in the same Kafka **topic** because different topics mean different partitions.

We [configured our Kafka instance](https://github.com/idugalic/digital-restaurant/blob/master/drestaurant-apps/drestaurant-microservices/docker-compose.yml) to crate only one topic (**axon-events**) with one partition initially.

##### Queue vs publish-subscribe (kafka groups)

If all consumers are from the same group, the Kafka model functions as a traditional **message queue** would.
All the records and processing is then load balanced.
**Each message would be consumed by one consumer of the group only.**
Each partition is connected to at most one consumer from a group.

When multiple consumer groups exist, the flow of the data consumption model aligns with the traditional **publish-subscribe** model.
**The messages are broadcast to all consumer groups.**

We [configured our (micro)services](https://github.com/idugalic/digital-restaurant/blob/master/drestaurant-apps/drestaurant-microservices/drestaurant-microservices-command-customer/src/main/resources/application.yml) to use publish-subscribe model, by setting unique consumer group id for each (micro)service.

#### Spring Cloud connector & command messages

The [Spring Cloud connector](https://docs.axoniq.io/reference-guide/1.3-infrastructure-components/command-dispatching#spring-cloud-connector) setup uses the service registration and discovery mechanism described by Spring Cloud for distributing the command bus (`commands`).
You are thus left free to choose which Spring Cloud implementation to use to distribute your commands.
An example implementation is the Eureka Discovery/Eureka Server combination.


### Microservices 2 (REST, RabbitMQ)

Each [microservice](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-microservices-rest):

 - has its own bounded context,
 - has its own JPA event(sourcing) store (we are not sharing the JPA Event Store)
 - we distribute events between them via RabbitMQ
 - and we distribute commands (Command Bus) by Spring Cloud discovery and registry service (Eureka)

 
#### RabbitMQ & event messages

RabbitMQ is the most popular open source message broker. It is used to route and distribute `events`.
It supports several messaging protocols, directly and through the use of plugins:

 - AMQP
 - STOMP
 - MQTT
 - HTTP ...

##### Publish-subscribe

This messaging pattern supports delivering a message to multiple consumers.

We [configured our (micro)services](https://github.com/idugalic/digital-restaurant/blob/master/drestaurant-apps/drestaurant-microservices-rest/drestaurant-microservices-rest-customer/src/main/resources/application.yml) to use publish-subscribe model, by setting unique queue for each (micro)service.
This queues are bind to one common exchange (`events.fanout.exchange`).

RabbitMQ allows more sophisticated message routing then Apache Kafka can offer.
Having one exchange bind to every service queue covered our scenario, but you can do more if you like.

#### Spring Cloud connector & command messages

The [Spring Cloud connector](https://docs.axoniq.io/reference-guide/1.3-infrastructure-components/command-dispatching#spring-cloud-connector) setup uses the service registration and discovery mechanism described by Spring Cloud for distributing the command bus (`commands`).
You are thus left free to choose which Spring Cloud implementation to use to distribute your commands.
An example implementation is the Eureka Discovery/Eureka Server combination.



### Microservices 3 (Websockets, AxonServer)

Each [microservice](https://github.com/idugalic/digital-restaurant/tree/master/drestaurant-apps/drestaurant-microservices-rest):

 - has its own bounded context,
 - has shared event(sourcing) storage (AxonServer)
 - and we distribute messages between them via AxonServer
 

#### AxonServer & event messages & command messages & query messages

The key characteristics of [AxonServer](https://axoniq.io/product-overview/axon-server) are:
 - Dedicated infrastructure for exchanging three types of messages (`commands`, `events`, `queries`) in a message-driven micro-services environment
 - Purpose-built database system optimized for the storage of event data of the type that is generated by applications that use the event sourcing architecture pattern
 - Built-in knowledge on CQRS message patterns
 - Easy-to-use and easy-to-manage

Both [AxonFramework](https://axoniq.io/product-overview/axon-framework) and [AxonServer](https://axoniq.io/product-overview/axon-server) form [Axon platform](https://docs.axoniq.io/reference-guide/).


Axon platform is end-to-end development and infrastructure platform for smoothly evolving Event-Driven microservices focused on CQRS and Event Sourcing.

AxonFramework and AxonServer are `open source`. [Axon Server Enterprise](https://axoniq.io/product-overview/axon-enterprise) is targeted towards mission-critical, medium to large scale production deployments of Axon. AxonEnterprise is not open source.

Axon is integrated with Spring Boot platform. You can choose many Spring Boot starters, and start with minimal configuration.
AxonServer is configured by default.

```
<dependency>
    <groupId>org.axonframework</groupId>
    <artifactId>axon-spring-boot-starter</artifactId>
    <version>${axon.version}</version>
</dependency>

```

## Development

This project is driven using [Maven][mvn].

### Clone
```bash
$ git clone https://github.com/idugalic/digital-restaurant
```

### Build

```bash
$ cd digital-restaurant
$ mvn clean install
```
### Run monolith 1 (HTTP and WebSockets API by segregating Command and Query)

```bash
$ cd digital-restaurant/drestaurant-apps/drestaurant-monolith
$ mvn spring-boot:run
```

### Run monolith 2 (REST API by not segregating Command and Query)

```bash
$ cd digital-restaurant/drestaurant-apps/drestaurant-monolith-rest
$ mvn spring-boot:run
```

### Run monolith 3 (STOMP over WebSockets API. We are async all the way)

```bash
$ cd digital-restaurant/drestaurant-apps/drestaurant-monolith-websockets
$ mvn spring-boot:run
```

### Run microservices 1 (HTTP, Websockets, Apache Kafka)

NOTE: Docker is required. We use it to start Apache Kafka with Zookeeper

```bash
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices
$ docker-compose up -d
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices/drestaurant-microservices-command-courier
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices/drestaurant-microservices-command-customer
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices/drestaurant-microservices-command-restaurant
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices/drestaurant-microservices-command-order
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices/drestaurant-microservices-query
$ mvn spring-boot:run
```

### Run microservices 2 (REST, RabbitMQ)

NOTE: Docker is required. We use it to start RabbitMQ

```bash
$ docker run -d --hostname my-rabbit --name some-rabbit -p 15672:15672 -p 5672:5672 rabbitmq:3-management
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-rest/drestaurant-microservices-rest-courier
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-rest/drestaurant-microservices-rest-customer
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-rest/drestaurant-microservices-rest-restaurant
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-rest/drestaurant-microservices-rest-order
$ mvn spring-boot:run
```
### Run microservices 3 (Websockets, AxonDB and AxonHub)

[AxonServer](https://axoniq.io/product-overview/axon-server) is required.

```bash
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-websockets/drestaurant-microservices-websockets-comand-courier
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-websockets/drestaurant-microservices-websockets-comand-customer
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-websockets/drestaurant-microservices-websockets-comand-restaurant
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-websockets/drestaurant-microservices-websockets-comand-order
$ mvn spring-boot:run
$ cd digital-restaurant/drestaurant-apps/drestaurant-microservices-websockets/drestaurant-microservices-websockets-query
$ mvn spring-boot:run
```

## Continuous delivery

We have one deployment pipeline for all applications and libraries within this repository. In addition, all projects in the repository share the same dependencies. Hence, there are no version conflicts because everyone has to use the same/the latest (SNAPSHOTS) version. And you don't need to deal with a private NPM (JavaScript) or Maven (Java) registry when you just want to use your own libraries.
This setup and project structure is usually addressed as a [monorepo](https://medium.com/@maoberlehner/monorepos-in-the-wild-33c6eb246cb9).

## Technology

### Language
- [Kotlin][kotlin]

### Frameworks and Platforms
- [Spring (spring boot, spring cloud, spring data, spring data rest)][spring]
- [Axon][axonframework]

### Continuous Integration and Delivery 
- Travis

### Infrastructure and Platform (As A Service)
- [H2 - java SQL database][h2]
- [Apache Kafka][kafka]
- [RabbitMQ][rabbitMQ]
- [AxonServer](https://axoniq.io/product-overview/axon-server)

## References and further reading

  - https://github.com/microservice-patterns/ftgo-application
  - https://docs.axonframework.org/
  - https://spring.io/blog/2018/04/11/event-storming-and-spring-with-a-splash-of-ddd
  - http://www.codingthearchitecture.com/2016/04/25/layers_hexagons_features_and_components.html


[mvn]: https://maven.apache.org/
[kotlin]: https://kotlinlang.org/
[spring]: https://spring.io/
[axonframework]: https://axoniq.io/
[mysql]: https://www.mysql.com/
[h2]: http://h2database.com/html/main.html
[rabbitMQ]: https://www.rabbitmq.com/
[kafka]: https://kafka.apache.org/
[pivotalCF]: https://run.pivotal.io/