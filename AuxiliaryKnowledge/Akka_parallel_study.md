* 什么是Akka？
* Akka解决了什么问题？
* 什么是Actor模型及什么是actor？
* Akka并发编程的理解，结合Flink里面的通讯机制

# 什么是Akka？

Akka is a toolkit and runtime for building highly concurrent, distributed, and fault-tolerant event-driven applications on the JVM

Akka can be used with both Java and Scala

# 什么是Actor模型及什么是Actor？

Actors are the unit of execution in Akka

The Actor model is an abstraction that makes it easier to write correct concurrent, parallel and distributed systems

# 什么是Akka ActorSystem

The akka.actor.ActorSystem factory is, to some extent, similar to Spring’s BeanFactory. 

It acts as a container for Actors and manages their life-cycles. 

The actorOf factory method creates Actors and takes two parameters, a configuration object called Props and a name.

# Akka解决了什么问题或者说为什么会设计Akka

参考：

[Why modern systems need a new programming model](https://doc.akka.io/docs/akka/current/guide/actors-motivation.html)

[How the Actor Model Meets the Needs of Modern, Distributed Systems](https://doc.akka.io/docs/akka/current/guide/actors-intro.html)


what happens when an actor receives a message:

1. The actor adds the message to the end of a queue.
2. If the actor was not scheduled for execution, it is marked as ready to execute.
3. A (hidden) scheduler entity takes the actor and starts executing it.
4. Actor picks the message from the front of the queue.
5. Actor modifies internal state, sends messages to other actors.
6. The actor is unscheduled.

To accomplish this behavior, actors have:

1. A mailbox (the queue where messages end up).
2. A behavior (the state of the actor, internal variables etc. 
3. Messages (pieces of data representing a signal, similar to method calls and their parameters).
4. An execution environment (the machinery that takes actors that have messages to react to and invokes their message handling code).
5. An address (more on this later).


# Akka并发编程的理解


***Challenges*** 

* How to build and design high-performance, concurrent applications.
* How to handle errors in a multi-threaded environment.
* How to protect my project from the pitfalls of concurrency.

lib

* Akka-actor
* Remote
* Cluster
* Cluster Sharding

**The Akka actor hierarchy**

![](..\images\ea9c680d.png)

Why do we need this hierarchy? What is it used for?

An important role of the hierarchy is to safely manage actor lifecycles






















