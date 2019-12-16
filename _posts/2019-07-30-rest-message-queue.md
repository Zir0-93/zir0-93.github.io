--- 
title:  "3 Common Misunderstandings of Inter-Service Communication in MicroServices"
date:  2019-12-05 15:04:23
og_image: /images/microservices.png
tags: [microservices, rest, messaging]
description: Think back to the last time you worked in a distributed system, did you consider using something other than RESTful HTTP calls as the method of communication between components in this system?
excerpt_separator: <!--more-->
---

Think back to the last time you worked in a distributed system, did you consider using something **other** than RESTful HTTP calls as the method of communication *between components in this system*?
<!--more-->

In the world of microservices, the problem of inter-service communication has given rise to two main solutions. The first solution is based on the use of
[RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer#Applied_to_Web_services){:target="_blank"} HTTP calls, while the other solution revolves around the use of
[message queues](https://en.wikipedia.org/wiki/Message_queue){:target="_blank"}.

As is typically the case when making such design decisions, the right decision is made based on a firm understanding of your **requirements** and the **trade-offs** involved in either approach. In this post, we'll try to improve our understanding of these trade-offs by analyzing some widely held misconceptions that are often given in justifying one approach over the other.

![cd_img](/images/1536697037-consul-dynamic-infrastructure.svg)

* TOC
{:toc}


## Overview
Ideally, services are self contained in a distributed system and don't need to rely on each other to do their job. This is one of the many parallels between services in a MicroServices architecture and [objects](http://mfadhel.com/lost-oop/#objects-are-intelligent-and-self-contained) in an object oriented system. However, as is commonly found in both domains, such a thing is not possible 100% of the time. As a result, services need a way to communicate with each other.

To introduce this discussion of inter-service communication approaches, let's first define some key concepts:

* **Microservices**: both a software development method and architectural style for that focuses on building sophisticated software systems as a collection of loosely coupled and highly maintainable single-function modules with well-defined interfaces and operations. Our discussion revolves around the communication between services in such systems. 
* **REST**: An architectural design patten for APIs that revolves around the concept of resources. For our discussion, it is sufficient to understand that communication in a REST architecture is typically implemented via HTTP protocols where senders **directly** invoke consumers.
* **Message Queue**: An asynchronous, **decoupled** form of inter-service communication in which senders of data place messages in a message queue until they are processed and deleted by receiving services. 

For the last few years, microservices have witnessed a tremendous growth in popularity as organizations increasingly transform
their monolithic applications into microservices. However, it is interesting to see that most introductory articles, tutorials, and
guides on the subject of microservices implement inter-service communication using synchronous RESTful communication patterns. 

{% include image_with_caption.html url="/images/interest_micro.png" description="Google trends chart for interest over time for the search term 'microservices'." %}

This is not surprising, as this communication mechanism is easiest to adopt by clients due to its simplicity and extensive support in the form of libraries in every programming language. Not to mention, RESTful systems can be documented in some pretty
[intuitive ways](http://mfadhel.com/API_Tables/#api-tables) due to its reliance on the concept of resources. However, as Martin Thompson
puts it, "Synchronous communication is the crystal meth of distributed software," and is generally overused in most microservices
implementations and data pipelines.

Although less formal support is natively provided for security, caching, and documentation in message queues, they have their own set of
advantages over RESTful HTTP calls. Not only do message queues decouple senders from receivers, but they also make systems much more
fault tolerant. That is, messages can be persisted in the queue even when individual services go down. These services can then resume
reading messages from the queue when they are back online. 

At the end of the day, there are usually good reasons to use both mechanisms in a distributed system.
In this post however, I want to **focus on some of the wrong reasons** commonly given to use one communication mechanism over the other.

## Message Queues Add Additional Cost
> "The cost of the message queue infrastructure to persist messages introduces significantly increased costs to the system."

Firstly, HTTP communication would typically be routed through
a load balancer in the absence of a message queue, which would incur some cost anyways although it may not be as significant.

Secondly, message queues typically operate over much *lighter weight* protocols than HTTP, which increases the effectiveness of your
infrastructure and lowers costs incurred as a result. When two services are experiencing 
a large amount of traffic between each other, the overhead of creating the TCP connection and negotiating SSL/TLS for all the 
requests sent between these two services is extremely large since HTTP cannot keep a connection open.

On the other hand, protocols like [Advaned Message Queuing Protocol](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)
(AMQP) over which many message queue technologies are based on create a connection at the start
and persist it. They keep messages flowing between service on top of the TCP connection, and so the TCP and SSL/TLS cost is only paid
once. The protocol minimizes the number of bytes that flow over the wire, and is ideally suited for systems that need to exchange
a high volume of small messages. 

{% include image_with_caption.html url="/images/camqp.png" description="High level overview of the AMQP protocol." %}

Lastly, the requirement of fault tolerance and failure in your systems needs to be clarified for this statement regarding cost to be true.
If your system has higher tolerance levels for failed transmissions, than using message queues is more costly. 
However, if your system has a low tolerance for message deliveries and failed transmissions, then you must select one
of the following two options:

1. Develop and manage logic to handle failed communications between services **inside** the services themselves.
2. Use a message queue

Option `1` would typically involve the use of additional infrastructure components such as a database/caching system.
Additionally, the level of complexity and sophistication involved in developing resilient and robust inter-services data 
transmission mechanisms would far outweigh the cost of using a ready-made tool designed to solve exactly that problem.
In this scenario, more cost would be incurred to an organization that chooses not to use a message queue, and so this statement 
would not be true in this case.

## Message Queues Introduce a Single point of failure (SPOF)

> "Message queues introduce an entirely new piece of infrastructure to your architecture. As a result of being solely responsible for enabling communication between services, such a component introduces a massive SPOF."

Minus any extra architectural concerns to make them Highly Available (HA), message queues are a SPOF. However, they certainly
**do not introduce** a SPOF to any existing microservices implementation.
Mechanisms like load balancers and API Gateways are SPOFs that are considered integral to any microservices architecture. They too,
like message queues, must be designed to be HA any microservices implementation.

{% include image_with_caption.html url="/images/microservices_diag.png" description="Like a message queue, an API gateway needs to be HA to avoid being a SPOF." %}

One disadvantage of RESTful HTTP calls however is that services must receive communications directly from senders, which represents a
SPOF. If the receiving service becomes 
unavailable for any reason, the system will lose any transmitted data and essentially fail in the sense of being unable to move forward
with the process.

The advantage of message queues here lies in its ability to handle downtime for individual services. That is to say, the queue can
still keep the messages transmitted to it by a sender service until the receiving service is available again and able to consume them. 
Moreover, we would only have to worry about making our message queue highly available as opposed to every single service within our 
system. 

## Unlike HTTP Calls, Messaging is Asynchronous

> "Unlike AMQP, HTTP is a synchronous protocol which prevents services that use it from communicating in an asynchronous way"

HTTP is certainly a synchronous protocol, but the terms synchronous and asynchronous have different connotations depending on which domain it is used in, making over oversimplifications such as these all too common. Lets analyze it at three levels:

**1 - Input/Output Level**:  At this level, asynchronous means that requests made to other services do not block the main executing
thread until the service has responded. This allows the thread to complete other tasks in the meantime, and increases the efficiency
of your CPU by allowing you to serve more requests. 

{% include image_with_caption.html url="/images/ansychronous-synchronous-requests-android-preview-opt.png" description="RESTFul HTTP calls can be implemented in both a synchronous and asynchronous fashion at an IO level." %}

This level of asynchronization can be implemented in both RESTful calls and 
message queues, which makes the statement above false. I've illustrated an example below using the `CompletableFuture` mechanism introduced in Java 8 to make an asynchronous HTTP call 
to an API. It's asynchronous because the request completes in a separate thread and does not block the main thread.
```java
public CompletableFuture<String> get(String uri) {
    HttpClient client = HttpClient.newHttpClient();
    HttpRequest request = HttpRequest.newBuilder()
          .uri(URI.create(uri))
          .build();

    return client.sendAsync(request, BodyHandlers.ofString())
          .thenApply(HttpResponse::body);
}
```
**2 - Protocol Level**: As mentioned previously, HTTP is a synchronous protocol because the client issues a request and waits
for a response. Message queues on the other hand, are typically based on asynchronous protocols. RabbitMQ for example, 
is based on AMQP which is a much lighter weight protocol than HTTP and can operate in a "fire and forget" manner.
Using the `nucleon.amqp` python library, we can write code to publish an event to an AMQP based message queue in the following way. 
```python
from nucleon.amqp import Connection

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
Unlike an HTTP call, no response is returned from publishing a message to a queue, and the `basic_publish` method returns immidiately. For this reason, the above statement regarding asynchronicity is true. 
**3 - Service Integration Level**: Asynchronous communication at this level is concerned with designing micoservices so that they do
not need to communicate with other services during their request/response cycle.  Why? Because at the end of day, the goal for a 
service is to be available to the end-user even if other services that are part of the whole system are offline or unhealthy.

If a service needs to trigger some action in another service, it should be done outside of the request/response cycle.
Additionally, if a service relies on data that is located in another service, data should be replicated across the services using
eventual consistency. This also has the advantage of being able to translate that data into the language of your own Bounded Context.

![eventual_consistency](/images/indepentent-microservice-databases.png)

The point here is that asynchronous service integration is an architectural design decision that is **independent** of the specific
communication mechanism used. In theory, message queues could be used to *synchronously* integrate services where they 
are designed to deliver and wait for messages from a message queue in a stateful way. Ofcourse, that would be a really poor design. 

## Do you Agree?
I hope this blog post has shed some light on some common misunderstandings when it comes to the RESTful HTTP Calls Vs Message Queue
debate. I strongly feel that important architectural decisions such as these cannot be made with a "surface level" understanding of the
technologies involved, which is why I've tried to improve my understanding in this area. 

If you disagree with some of the points I've mentioned, or think I've missed some good points in my list, let me know in the comments below or on twitter!
