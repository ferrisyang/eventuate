[![Build Status](https://travis-ci.org/RBMHTechnology/eventuate.svg?branch=master)](https://travis-ci.org/RBMHTechnology/eventuate)

Eventuate
=========

Introduction
------------

Eventuate is a toolkit for building distributed, highly-available and partition-tolerant event-sourced applications. It is written in Scala and built on top of [Akka](http://akka.io/), a toolkit for building highly concurrent, distributed, and resilient message-driven applications on the JVM.

[Event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) captures all changes to application state as a sequence of events. These events are persisted in an event log and can be replayed to recover application state. Events are immutable facts that are only ever appended to a log which allows for very high transaction rates and efficient replication.

Eventuate supports replication of application state through asynchronous event replication between multiple _locations_. These can be geographically distinct locations, nodes within a data center or even processes on the same node, for example. Locations consume replicated events to re-construct application state locally. Eventuate allows multiple locations to concurrently update replicated state (master-master setup) and supports pre-defined or custom conflict resolution strategies in case of conflicting updates. This is known as operation-based [optimistic replication](http://en.wikipedia.org/wiki/Optimistic_replication) where operations are represented by application-defined events.

Individual locations remain available for local writes even in the presence of network partitions. Events that have been captured locally during a network partition are replicated later when the partition heals. Storing events locally and replicating them later can also be useful for distributed applications deployed on temporarily connected devices, for example.

At the core of Eventuate is a replicated event log. Events captured at a location are stored locally and replicated asynchronously to other locations based on a replication protocol that preserves the [causal ordering](http://krasserm.github.io/2015/01/13/event-sourcing-at-global-scale/#event-log) of events where causality is tracked with [vector clocks](http://en.wikipedia.org/wiki/Vector_clock). For any two events, applications can determine if they have a causal relationship or if they are concurrent by comparing their vector timestamps. This is important to achieve eventual consistency of replicated application state. An event log can also be partitioned for scaling writes.

Storage technologies at individual locations are pluggable (SPI not public yet). A location deployed on a mobile device, for example, will probably choose to write events to the local filesystem whereas a location deployed in a data center could choose to write events to an [Apache Kafka](http://kafka.apache.org/) cluster. Event replication across locations is independent of the storage technologies used at individual locations, so that distributed applications with hybrid event stores are possible.

To model application state, any custom data types can be used. Applications just need to ensure that projecting events from a causally ordered event stream onto these data types yield a consistent result. This may involve detection of conflicts from concurrent events, selecting one of the conflicting versions as the winner or even merging them. Conflict resolution can be automated or interactive. Eventuate also provides implementations of [operation-based CRDTs](https://krasserm.github.io/2015/02/17/Implementing-operation-based-CRDTs/), specially-designed data structures used to achieve [strong eventual consistency](http://en.wikipedia.org/wiki/Eventual_consistency#Strong_eventual_consistency).

Eventuate’s approach to optimistic replication and eventual consistency is nothing new. It is implemented in many distributed database systems that choose AP from [CAP](http://en.wikipedia.org/wiki/CAP_theorem). These database systems internally maintain a transaction log that is replicated to re-construct state at different replicas. Besides offering a programming model for storing state changes instead of current state, Eventuate also differs from these database systems in the following ways:

- The transaction log is exposed to applications as event log. Although many distributed database systems provide change feeds, these deliver rather technical events (DocumentCreated, DocumentUpdated, ... for example) compared to domain-specific events in an event log with clear semantics in the application domain (CustomerRelocated, PaymentDelayed, ...). Change feeds with domain-specific events can have significant advantages for building custom read models in [CQRS](http://martinfowler.com/bliki/CQRS.html) or for application integration where semantic integration is often a major challenge.

- Application state can be any application-defined data type together with an event projection function to update and recover state from an event stream. To recover application state at a particular location, the event stream can be deterministically replayed from a local event log. For application state that can be updated at multiple locations, Eventuate provides utilities to track concurrent versions which can be used as input for automated or interactive conflict resolution.

Articles
--------

- [Event sourcing at global scale](https://krasserm.github.io/2015/01/13/event-sourcing-at-global-scale/)
- [Implementing operation-based CRDTs in Scala](https://krasserm.github.io/2015/02/17/Implementing-operation-based-CRDTs/)

Status
------

The project has _early access_ status and will gradually evolve into a production-ready toolkit over the next months. Documentation in the following sections is also work in progress.

Community
---------

- [User mailing list](https://groups.google.com/forum/#!forum/eventuate)
- [![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/RBMHTechnology/eventuate?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

Dependencies
------------

### Sbt

    resolvers += "OJO Snapshots" at "https://oss.jfrog.org/oss-snapshot-local"

    libraryDependencies += "com.rbmhtechnology" %% "eventuate" % "0.1-SNAPSHOT"

### Maven

    <repository>
        <id>ojo-snapshots</id>
        <name>OJO Snapshots</name>
        <url>https://oss.jfrog.org/oss-snapshot-local</url>
    </repository>

    <dependency>
        <groupId>com.rbmhtechnology</groupId>
        <artifactId>eventuate_2.11</artifactId>
        <version>0.1-SNAPSHOT</version>
    </dependency>

Locations
---------

Locations share a replicated event log. For instance, the [example application](#example-application) has 6 locations (A - F), each represented by a separate process on localhost. By changing the configuration, locations can also be distributed on multiple hosts. Locations A - F are connected via bi-directional, asynchronous event replication connections as shown in the following figure.  
  
    A        E
     \      /    
      C -- D
     /      \
    B        F

Each location must configure

- a hostname and port for Akka Remoting (`akka.remote.netty.tcp.hostname` and `akka.remote.netty.tcp.port`)
- the id of the local replication endpoint (`endpoint.id` which is equal to the location id)
- a list of replication connections to other locations (`endpoint.connections`)

For example, this is the configuration of location C:
  
    akka.remote.netty.tcp.hostname = "127.0.0.1"
    akka.remote.netty.tcp.port=2554
  
    endpoint.id = "C"
    endpoint.connections = ["127.0.0.1:2552", "127.0.0.1:2553", "127.0.0.1:2555"]

Later versions will support the automated setup of replication connections when a location joins or leaves a replication network.

Event log
---------

The replicated event log implementation of the prototype is backed by a LevelDB instance at each location i.e. each location has its own local event log. A local event log exposes a replication endpoint that is used by the inter-location replication protocol for replicating events. A local event log and its replication endpoint can be instantiated as follows:
  
```scala
import akka.actor._
  
import com.rbmhtechnology.eventuate._
import com.rbmhtechnology.eventuate.log.LeveldbEventLog

val system = ActorSystem("site")
val endpoint = ReplicationEndpoint(logId => LeveldbEventLog.props(logId))(system)
```

The log actor can be obtained with

```scala
val log: ActorRef = endpoint.logs("default")
```

Replication endpoints and their configuration (`endpoint.id` and `endpoint.connections`) at all locations implement the replicated event log. Please note that the current implementation doesn't allow cycles in the replication network to preserve the happens-before relationship (= potential causal relationship) of events. This is a limitation that will be removed in later versions (which will be able re-order events based on their vector timestamps). 

### Advanced configuration

As already shown in section [Locations](#locations), replication connections (connecting to remote replication endpoints) can be configured with the `log.connections` configuration key. Alternatively, applications can also set up replication connections programmatically:

```scala
val connections = Seq(
  ReplicationConnection("127.0.0.1", 2552),
  ReplicationConnection("127.0.0.1", 2553),
  ReplicationConnection("127.0.0.1", 2555))
  
val localEndpoint = ReplicationEndpoint(id => LeveldbEventLog.props(id), connections)(system)
```

A `ReplicationConnection` connects a local replication endpoint to a remote replication endpoint. The connection is unidirectional and events are replicated from the remote endpoint to the local endpoint. For bi-directional replication, the remote endpoint must additionally set up a replication connection in the other direction.  

A `ReplicationConnection` can also specify a `ReplicationFilter`. Only events that pass the filter are actually replicated. Replication filters are configured on the local endpoint but applied on the remote endpoint. Hence, they must be serializable. In the following example, only events of type `String` that start with `"foo"` will be replicated from the remote to the local endpoint:

```scala
class MyFilter extends ReplicationFilter {
  override def apply(event: DurableEvent): Boolean = event.payload match {
    case s: String if s.startsWith("foo") => true
    case _ => false
  }
}

val connections = Seq(
  ReplicationConnection("localhost", 2553, Some(new MyFilter)))

val localEndpoint = ReplicationEndpoint(id => LeveldbEventLog.props(id), connections)(system)
```

### Multiple logs

A replication endpoint can also maintain and replicate multiple logs. Each log represents a partition and event causality is preserved only within a partition. The following example creates a replication endpoint with id `ep1` and two logs with names `log1` and `log2`: 

```scala
val logNames = Set("log1", "log2")
val localEndpoint = new ReplicationEndpoint("ep1", logNames, id => LeveldbEventLog.props(id), connections)(system)
```

Only logs with matching names across endpoints are actually replicated. Log actors can be obtained with

```scala
val log1: ActorRef = localEndpoint.logs("log1")
val log2: ActorRef = localEndpoint.logs("log2")
```

Replication connections may also specify replication filters for each log name individually, as shown in the following example:

```scala
val connections = Seq(
  ReplicationConnection("127.0.0.1", 2552, Map("log1" -> new MyFilter1, "log2" -> new MyFilter2)),
  ReplicationConnection("127.0.0.1", 2553, Map("log1" -> new MyFilter3, "log2" -> new MyFilter4)),
  ReplicationConnection("127.0.0.1", 2555))
```

### Failure detection

Replication connections have a built-in failure detector that publishes 

- `Available(endpointId: String, logName: String)` messages to the actor system's event stream if the remote replication endpoint is available and
- `Unavailable(endpointId: String, logName: String)` messages to the actor system's event stream if the remote replication endpoint is unavailable
  
Both messages are defined in object `ReplicationEndpoint`. Their `endpointId` parameter is set the to the remote endpoint id. The failure detector can be configured with the `log.replication.failure-detection-limit` configuration key. For example,

    log.replication.failure-detection-limit = 100s 
    
instructs the failure detector to publish an `Unavailable` message if there is no heartbeat from the remote endpoint within 100 seconds. `Available` and `Unavailable` messages are published periodically at intervals of `log.replication.failure-detection-limit`.

### Testing

For testing purposes, a local event log can also be instantiated without a replication endpoint:
  
```scala
val log = system.actorOf(LeveldbEventLog.props("my-log-id"))
```

Event-sourced actors
--------------------

An event-sourced actor (EA) can be implemented by extending the `EventsourcedActor` trait.

```scala
class MyEA(override val log: ActorRef, override val processId: String) extends EventsourcedActor {
  override def onCommand: Receive = {
    case "some-cmd" => persist("some-event") {
      case Success(evt) => onEvent(evt)
      case Failure(err) => // persistence failed ...
    }
  }

  override def onEvent: Receive = {
    case "some-event" => // ...
  }
}
```

Similar to akka-persistence, commands are processed by a command handler (`onCommand`), events are processed by an event handler (`onEvent`) and events are written to the event log with the `persist` method whose signature is `persist[A](event: A)(handler: Try[A] => Unit)`. An EA must also implement `log` which is the event log that the EA shares with other EAs. The event log `ActorRef` of a location can be obtained from the replication endpoint with `endpoint.log`. 

The `processId` of an EA must be globally unique as it contributes to vector clocks. In the current implementation, each EA instance maintains its own vector clock, as it represents a light-weight process that exchanges events with other EA instances via a shared distributed `log`. Therefore, vector clock sizes linearly grow with the number of EA instances. Future versions will provide means to limit vector clock sizes, for example, by partitioning EAs in the event space or by grouping several EAs into the same "process", so that the system can scale to a large number of EAs.  

An EA does not only receive events persisted by itself but also those persisted by other EAs (at the same or other locations) as long as they share a replicated `log` and can process those in its event handler. With the following definition of `MyEA`

```scala
class MyEA(val log: ActorRef, val processId: String) extends EventsourcedActor {
  def onCommand: Receive = {
    case cmd: String => persist(s"${cmd}-${processId}") {
      case Success(evt) => onEvent(evt)
      case Failure(err) => // ...
    }
  }

  def onEvent: Receive = {
    case evt: String => println(s"[${processId}] event = ${evt} (event timestamp = ${lastVectorTimestamp})")
  }
}
```

and two instances of `MyEA`, `ea1` and `ea2` (with `processId`s `ea1` and `ea2`)

```scala
val ea1 = system.actorOf(Props(new MyEA(endpoint.log, "ea1")))
val ea2 = system.actorOf(Props(new MyEA(endpoint.log, "ea2")))
```

we send `ea1` and `ea2` a `test` command.

```scala
ea1 ! "test"
ea2 ! "test"
```

If `ea1` and `ea2` receive the `test` command before having received an event from the other actor then the generated events are concurrent and the output should look like

    [ea1] event = test-ea1 (event timestamp = VectorTime(ea1 -> 1))
    [ea2] event = test-ea1 (event timestamp = VectorTime(ea1 -> 1))
    [ea1] event = test-ea2 (event timestamp = VectorTime(ea2 -> 1))
    [ea2] event = test-ea2 (event timestamp = VectorTime(ea2 -> 1))

Please note that the timestamps in the output are event timestamps and not the current time of the vector clocks of `ea1` and `ea2`. If, on the other hand, `ea2` receives the `test` command after having having received an event from `ea1` then the event written by `ea2` causally depends on that from `ea1` and the output should look like

    [ea1] event = test-ea1 (event timestamp = VectorTime(ea1 -> 1))
    [ea2] event = test-ea1 (event timestamp = VectorTime(ea1 -> 1))
    [ea2] event = test-ea2 (event timestamp = VectorTime(ea1 -> 1,ea2 -> 2))
    [ea1] event = test-ea2 (event timestamp = VectorTime(ea1 -> 1,ea2 -> 2))
    
Maybe you'll see a different ordering of lines as `ea1` and `ea2` are running concurrently.

Event-sourced views
-------------------

Classes that extend the `EventsourcedView` (EV) trait (instead of `EventsourcedActor`) only implement the event consumer part of an EA but not the event producer part (i.e. they do not have a `persist` method). EVs also don't have a `processId` (as they do not contribute to vector clocks) but still need to implement `log` from which they consume events.  

```scala
class MyEV(val log: ActorRef) extends EventsourcedView {
  def onCommand: Receive = {
    case cmd => // ...
  }

  def onEvent: Receive = {
    case evt => // ...
  }
}
```

From a CQRS perspective, 

- EVs should be used to implement the query side (Q) of CQRS (and maintain a read model) whereas
- EAs should be used to implement the command side (C) of CQRS (and maintain a write model)


Vector timestamps
-----------------

The vector timestamp of an event can be obtained with `lastVectorTimestamp` during event processing. Vector timestamps, for example, can be attached as version vectors to current state and compared to timestamps of new events in order to determine whether previous state updates happened-before or are concurrent to the current event.

```scala
class MyEA(val log: ActorRef, val processId: String) extends EventsourcedActor {
  var updateTimestamp: VectorTime = VectorTime()

  def updateState(event: String): Unit = {
    // ...
  }

  def onCommand: Receive = {
    // ...
  }

  def onEvent: Receive = {
    case evt: String if updateTimestamp < lastVectorTimestamp =>
      // all previous state updates happened-before current event
      updateState(evt)
      updateTimestamp = lastVectorTimestamp
    case evt: String if updateTimestamp conc lastVectorTimestamp =>
      // one or more previous state updates are concurrent to current event
      // TODO: track concurrent versions ... 
  }
}
```

Vector timestamps can be compared with operators `<`, `<=`, `>`, `>=`, `conc` and `equiv` where
 
- `t1 < t2` returns `true` if `t1` happened-before `t2`
- `t1 conc t2` returns `true` if `t1` is concurrent to `t2`
- `t1 equiv t2` returns `true` if `t1` is equivalent to `t2`

Causal ordering
---------------

The replicated event log preserves the happens-before relationship (= potential causal ordering) of events: if `e1 -> e2`, where `->` is the happens-before relationship, then all EAs that consume `e1` and `e2` will consume `e1` before `e2`. The main benefit of causal ordering is that non-commutative state update operations (such as appending to a list) can be used.   

Concurrent versions
-------------------

The ordering of concurrent events, however, is not defined: if `e1` is concurrent to `e2` then some EA instances may consume `e1` before `e2`, others `e2` before `e1`. With commutative state update operations (see CmRDTs, for example), this is not an issue. With non-commutative state update operations, applications may want to track state updates from concurrent events as concurrent (or conflicting) versions of application state. This can be done with `ConcurrentVersions[S, A]` where `S` is the type of application state for which concurrent versions shall be tracked in case of concurrent events of type `A`.

```scala
class MyEA(val log: ActorRef, val processId: String) extends EventsourcedActor {
  var versionedState: ConcurrentVersions[Vector[String], String] = ConcurrentVersions(Vector(), (s, a) => s :+ a)

  def updateState(event: String): Unit = {
    versionedState = versionedState.update(event, lastVectorTimestamp, lastProcessId)
    if (versionedState.conflict) {
      // TODO: either automated conflict resolution immediately or interactive conflict resolution later ...
    }
  }

  def onCommand: Receive = {
    // ...
  }

  def onEvent: Receive = {
    case evt: String => updateState(evt)
  }
}
```

In the example above, application state is of type `Vector[String]` and update events are of type `String`. State updates (appending to a `Vector`) are not commutative and we track concurrent versions of application state as `ConcurrentVersions[Vector[String], String]`. The state update function (= event projection function) `(s, a) => s :+ a` is called whenever `versionedState.update` is called (with event, event timestamp and the `processId` of the event emitter as arguments). In case of a concurrent update, `versionedState.conflict` returns `true` and the application should [resolve](#conflict-resolution) the conflict.

Please note that concurrent events are not necessarily conflicting events. Concurrent updates to different domain objects may be acceptable to an application whereas concurrent updates to the same domain object may be considered as conflict. In this case, concurrent versions for individual domain objects should be tracked. For example, an order management application could declare application state as follows:

```scala
var versionedState: Map[String, ConcurrentVersions[Order, OrderEvent]] 
```

The current implementation of `ConcurrentVersions` still needs to be optimized to de-allocate versions that are older than a certain threshold (as they can be re-created any time from the event log, if needed). This will be supported in future project releases.      

Conflict resolution
-------------------

Concurrent versions of application state represent update conflicts from concurrent events. Conflict resolution selects one of the concurrent versions as the winner which can be done in an automated or interactive way. The following is an example of automated conflict resolution: 

```scala
class MyEA(val log: ActorRef, val processId: String) extends EventsourcedActor {
  var versionedState: ConcurrentVersions[Vector[String], String] = ConcurrentVersions(Vector(), (s, a) => s :+ a)

  def updateState(event: String): Unit = {
    versionedState = versionedState.update(event, lastVectorTimestamp, lastProcessId)
    if (versionedState.conflict) {
      // conflicting versions, sorted by processIds of EAs that emitted the concurrent update events
      val conflictingVersions: Seq[Versioned[Vector[String]]] = versionedState.all.sortBy(_.processId)
      // example conflict resolution function: lower process id wins
      val winnerVersion: VectorTime = conflictingVersions.head.version
      // resolve conflict by specifying the winner version
      versionedState = versionedState.resolve(winnerVersion)
    }
  }

  def onCommand: Receive = {
    // ...
  }

  def onEvent: Receive = {
    case evt: String => updateState(evt)
  }
}
```

In the example above, the version created from the event with the lower emitter `processId` is selected to be the winner. The selected version is identified by its version vector (`winnerVersion`) and passed as argument to `versionedState.update` during conflict resolution. Using emitter `processId`s for choosing a winner is just one example how conflicts can be resolved. Others could compare conflicting versions directly to make a decision. It is only important that the conflict resolution result does not depend on the order of conflicting events so that all application state replicas converge to the same value. 

Interactive conflict resolution does not resolve conflicts immediately but requests the user to select a winner version. In this case, the selected version must be stored as `Resolved` event in the event log (see example application code for details). Furthermore, interactive conflict resolution requires global agreement to avoid concurrent conflict resolutions. This can be achieved by static rules (for example, only the initial creator of a domain object may resolve conflicts for that object) or by global coordination, if needed. Global coordination can work well if conflicts do not occur frequently.

Commutative replicated data types
---------------------------------

Eventuate also provides an implementation of commutative replicated data types (CmRDTs or operation-based CRDTs) as described in the article [Implementing operation-based CRDTs in Scala](https://krasserm.github.io/2015/02/17/Implementing-operation-based-CRDTs/).  

Sending messages
----------------

For sending messages to other non-event-sourced actors (representing external services, for example), EAs have several options:

- during command processing with at-most-once delivery semantics

    ```scala
    trait MyEA extends EventsourcedActor {
      def onCommand: Receive = {
        case cmd => persist("my-event") {
          case Success(evt) => sender() ! "command accepted"
          case Failure(err) => sender() ! "command failed"
        }
      }
    
      def onEvent: Receive = {
        case evt => // ...
      }
    }
    ```

- during event processing with at-least-once delivery semantics (see also [at-least-once-delivery](http://doc.akka.io/docs/akka/2.3.8/scala/persistence.html#at-least-once-delivery) in akka-persistence)

    ```scala
    trait MyEA extends EventsourcedActor with Delivery {
      val destination: ActorPath
    
      def onCommand: Receive = {
        case cmd => // ...
      }
    
      def onEvent: Receive = {
        case "my-deliver-event" => deliver("my-delivery-id", "my-message", destination)
        case "my-confirm-event" => confirm("my-delivery-id")
      }
    }
    ```

- during event processing with at-most-once delivery semantics 

    ```scala
    trait MyEA extends EventsourcedActor {
      val destination: ActorRef

      def onCommand: Receive = {
        case cmd => // ...
      }

      def onEvent: Receive = {
        case "my-event" if !recovering => destination ! "my-message"
      }
    }
    ```
    
Conditional commands
--------------------

Conditional commands are commands that have a vector timestamp attached as condition and can be processed by EAs and EVs.

```scala
case class ConditionalCommand(condition: VectorTime, cmd: Any)
```

Their processing is delayed until an EA's or EV's `lastVectorTimestamp` is greater than or equal (`>=`) the specified `condition`. Hence, conditional commands can help to achieve read-your-write consistency across EAs and EVs (and even across locations), for example.   

Example application
-------------------

The example application is an over-simplified order management application that allows users to add and remove items from orders via a command-line interface. 

### Domain

The `Order` domain object is defined as follows:
 
```scala
 case class Order(id: String, items: List[String] = Nil, cancelled: Boolean = false) {
   def addItem(item: String): Order = ...
   def removeItem(item: String): Order = ...
   def cancel: Order = ...
 }
```

Order creation and updates are tracked as events in the replicated event log and replicated order state is maintained by `OrderManager` actors at all locations.

### Replication

As already mentioned, order events are replicated across locations A - F (all [running](#running) on `localhost`).

    A        E
     \      /    
      C -- D
     /      \
    B        F

Each location can create and update update orders, even under presence of network partitions. In the example application, network partitions can be created by shutting down locations. For example, when shutting down location `C`, partitions `A`, `B` and `E-D-F` are created. 

Concurrent updates to orders with different `id`s do not conflict, concurrent updates to the same order are considered as conflict and must be resolved by the user before further updates to that order are possible.    

### Interface

The example application has a simple command-line interface to create and update orders:

- `create <order-id>` creates an order with specified `order-id`.
- `add <order-id> <item>` adds an item to an order's `items` list.
- `remove <order-id> <item>` removes an item from an order's `items` list.
- `cancel <order-id>` cancels an order.

If a conflict from a concurrent update occurred, the conflict must be resolved by selecting one of the conflicting versions:

- `resolve <order-id> <index>` resolves a conflict by selecting a version `index`. Only the location that initially created the order object can resolve the conflict (static rule for distributed agreement).  

Other commands are:

- `count <order-id>` prints the number of updates to an order (incl. creation). Update counts are maintained by a separate view that subscribes to order events. 
- `state` prints the current state (= all orders) on at a location (which may differ at other locations during network partitions).
- `exit` stops a location and the replication links to and from that location.

Update results from user commands and replicated events are written to `stdout`. Update results from replayed events are not shown.

### Running

Before you continue, install [sbt](http://www.scala-sbt.org/) and run `sbt test` from the project's root (needs to be done only once). Then, open 6 terminal windows (representing locations A - F) and run

- `example-site A` for starting location A
- `example-site B` for starting location B
- `example-site C` for starting location C
- `example-site D` for starting location D
- `example-site E` for starting location E
- `example-site F` for starting location F

Alternatively, run `example` which opens these terminal windows automatically (works only on Mac OS X at the moment). For running the Java version of the example application, run `example-site` or `example` with `java` as additional argument. 

Create and update some orders and see how changes are replicated across locations. To make concurrent updates to an order, for example, enter `exit` at location `C`, and add different items to that order at locations `A` and `F`. When starting location `C` again, both updates propagate to all other locations which are then displayed as conflict. Resolve the conflict with the `resolve` command.  
