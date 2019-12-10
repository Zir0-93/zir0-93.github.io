--- 
title:  "A Practical Guide to Inc"
icon: /images/rocket_cd.png
date:  2019-01-05 15:04:23
og_image: /images/microservices.png
tags: [microservices, rest, messaging]
description: Remember the last time you designed a distributed system - did you consider using something other than RESTfull calls as the method of communication between components in this system?
excerpt_separator: <!--more-->
---

Remember the last time you designed a distributed system - did you consider using something other than RESTfull calls as the method of communication between components in this system?
<!--more-->

In the world of microservices, the discussion of inter-service communication has given rise to two main camps. On one end, are those who believe in using [REST](https://www.programmableweb.com/news/why-messaging-queues-suck/analysis/2017/02/13) for this purpose. While on the other end, exist those who believe in using [message queues](https://dev.to/matteojoliveau/microservices-communications-why-you-should-switch-to-message-queues--48ia) for such a purpose.

As is typically the case when making any such design decisions, it comes down to gaining a deep understanding of your requirements and the tradeoffs involved in either approach. In this post, we'll try to improve our understanding of these trade offs by analyzing some widely held misconceptions found on either side of the debate.


![cd_img](/images/1536697037-consul-dynamic-infrastructure.svg)

* TOC
{:toc}


# Overview
Ideally, services are self contained in a distributed system and don't need to rely on each other to do their job. This is one of the many parallels between services in a MicroServices architecture and [objects](http://mfadhel.com/lost-oop/#objects-are-intelligent-and-self-contained) in an object oriented system. However, as is commonly found in both domains, such a thing is not possible 100% of the time. To introduce this discussion of communication patterns in Microservices, lets define some key concepts:
* **Microservices**: both a software development method and architectural style for that focusses on building sophsiticated software systems as a collection of loosely coupled and highly maintainable single-function modules with well-defined interfaces and operations. 
* **REST**: An architectural design patten for APIs that revolves around the concept of resources. For our discussion, it is sufficient to understand that communication in a REST architecture is typically implemented via HTTP protocols.
* **Message Queue**: A form of asynchornous service-to-service communication commonly used in serverless and microservices architectures. Rather than directly invoking recievers, as is done in HTTP calls, senders place messages in the queue untill they are processed and deleted.

For the last few years, microservices have witnessed a tremendous growth in popularity as organizations are increasingly transforming their monolithic applications into MicroServices. However, it is interesting to see that most introductory articles, tutorials, and guides on the subject of microservices implement inter-service communication using synchronous RESTfull communication patterns. 

This is not surprising, as this communication mechanism is  easiest to adopt by clients, is already familiar to most developers and has extensive libraries to support it in every programming language. Not to mention, RESTful systems can be documented in [intutive ways](http://mfadhel.com/API_Tables/#api-tables) due to it's reliance on the concept of resources.  However, as Martin Thompson  puts it, "Synchronous communication is the crystal meth of distributed software," and is generally overused in most microservices implementations and data pipelines.

Although less formal support is natively provided for security, caching, and documentation in message queues, they have a number of advantages over synchronous HTTP calls. Not only do message queues decouple senders from recievers, but they also make systems much more fault tolerant. That is, messages can be persisted in the queue even when individual services go down. These services can then resume reading messages from the queue when they are back online. 

At the end of the day, there are usually good reasons to use both mechanisms in a distributed system. In this post however, I want to focus on some of the wrong reasons commonly given to use one communication mechanism over the other.

# 1 - Message Queues Introduce a Single point of failure (SPOF)

Rationale: Message queues introduce an entirely new piece of infrastructure to your architecture. As a result of being solely responsible for enabling communication between services, such a component introduces a massive SPOF.

Message queues do represent a SPOF, but they certainly do not introduce them to any existing MicroServices implementation based on non-messaging communication patterns. Mechanisms like load balancers and service discovery are SPOFS that are considered integral to any MicroServices architecture. 

Additionally, any service that recieveves communications directly from a sender represents a SPOF. If the recieving service becomes unavailable for any reason, the system will lose any transmitted data and essentially fail in the sense of being unable to move forward with the process.

The advantage of message queues here lies in its ability to handle downtime for individual services. That is to say, the queue can still keep the messages transmitted to it by a sender service until the recieving service is available again and able to consume them. Moreover, we would only have to worry about making our message queue highly available as opposed to every single service within our system. 

# 2 - Message Queues Increase Cost
Rationale: The cost of the message queue infrastructure to persist messages introduces a significant expenses to the system

It should first be noted that in the absence of a message queue, HTTP communication would typically be routed through a load balancer which would incurr expenses as well, albeit not as significant.

Secondly, the requirement of fault tolerance and failure needs to be clear for this statement to be true. If you're system has higher tolerance levels for failed transmissions, than using message queues is more costly. However, if your system has a low tolerance for message deliveries and failed transmissions, then you have two options:

Develop and manage logic to handle failed message transmissions within the services themselves.
Use a message queue

Option 1 would typically involve the use of additional infrastructure components such as a database/caching system.  Additionally, the level of complexity and sophisication involved in developing resillient and robust inter-services data transmission mechanisms would far outweigh the cost of using a ready made tool designed to solve exactly that problem. In this scenario, more cost would be incurred to an organization that chooses not to use a message queue, and so this statement is simply not true.

Additionally, as 3 and 4  will demonstrate, AMQP is a much lighter protocol than HTTP, which will put less load on your system, thereby making it cheaper.

# 3 - Unlike HTTP Calls, Messaging is Asynchronous

Rationale:  Unlike AMQP, HTTP is a synchronous protocol which prevents services that use it from communicating with each other in an asynchronous way.

HTTP is certainly a synchronous protocol, but the terms synchronous and asynchronous have different connotations depending on which domain it is used in, making misunderstandings such as these all too common. Lets analyze it at three levels:

**Input/Output Level**:  At this level, asynchronous means that requests made to other services do not block the main executing thread untill the service has responded. This allows the thread to complete other tasks in the meantime, and increases the efficiency of your CPU by allowing you to serve more requests. This level of asynchronization can be implemented in both RESTful calls and messaging queues. For example I can use the `CompletableFuture` mechanism introduced in Java 8 to make an asynchronous HTTP call to an API with following code. It asynchronous because the request completes in a separate thread and notifies the main thread upon completion.
```
public CompletableFuture<String> get(String uri) {
    HttpClient client = HttpClient.newHttpClient();
    HttpRequest request = HttpRequest.newBuilder()
          .uri(URI.create(uri))
          .build();

    return client.sendAsync(request, BodyHandlers.ofString())
          .thenApply(HttpResponse::body);
}
```
**Protocol Level**: As mentioned previously, HTTP is a Synchronous protocol - The client issues a request and waitress for a response. Messaging queues on the other hand, are typically based on asynchronous protocols. RabbitMQ for example, is based on [Advanced Message Queuing Protocol](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol) (AMQP) which is a much lighter weight protocol than HTTP and can operate in a "fire and forget" manner. Using the nucleon.amqp library, we can write code to publish an event to an AMQP based message queue in the following way. 
from nucleon.amqp import Connection
```
conn = Connection('amqp://localhost/')

with conn.channel() as channel:
    channel.queue_declare(queue='test')
    # This operation will return immediately
    channel.basic_publish(
        exchange='',
        routing_key='test',
        body='Hello world!'
    )
 ```
**Service Integration Level**:  Asynchronous communication at this level is concerned with designing micoservices so that they do not need to communicate with other services during their request/response cycle.  Why? Because at the end of day, the goal for a service is to be available to the end-user even if other services that are part of the whole system are offline or unhealthy.
If a service needs to trigger some action in another service, it should be done outside of the request/response cycle. Additionally, if a service relies on data that is located in another service, replicate that data into your own service’s data store, using eventual consistency. This also has the advantage that you can translate that data into the language of your own Bounded Context. 
The point here is that asynchronous service integration is an architectural design decision that is independant of the specific communication mechanism used. That is to say, message queues can actually be used to design a synchronous service integration process where services are designed to deliver and wait for messages in a stateful way. This would be very poor architectural design but the point is clear - message queues are not more "asynchronous" than traditional RESTful HTTP calls at the service integration level.
