
---
title:  "The Harmony between Cloud and Object Oriented Code"
date:   2019-02-25 15:04:23
icon: /images/2udfzo.png
tags: [oop, serverless]
description: 
excerpt_separator: 
---
Even with the best solution architects and resources available, your application's cloud transformation journey will be a nightmare if your code is **not** *object oriented*.
<!--more-->

![serverless_meme](/images/2udfzo.jpg)

Wanting to benefit from scalability and cost-efficiency of cloud technologies, organizations are increasingly migrating their monolothic applications to microservice and serverless based architectures. This migration is usally a long and difficult process.

**Why?** Because whether you are implementing a microservices or serverless based architecture, the most time consuming part of the migration process involves **decoupling** the monolith base into smaller, more refined components.

And this decoupling process will be much more frustrating and tedious than neccessary if the code does not have certain **characteristics** that make it easy *to split up into smaller pieces*. 

In this article I'll outline four principles of object oriented code and how each of them **faciltate** the process of decoupling of a monolithic application into smaller components intended to run in the cloud.


## Do One Thing Well

Classes that take on many responsibilities, also known as God Classes, are very difficult to read, maintain, and extend due to their enormous size. For this reason, classes in an Object Oriented design should **focus** on doing one thing well, and behave as a logically single, atomic unit. If the complexity of an object exceeds a reasonable level, it should be refactored into two or more separate entities.

When we consider that each service in a microservices architecture should focus on a specific domain, this method of designing classes gives us a great deal of **flexibility** in determining how we want to decouple a monolithic application into such services. 

![do-one-thing-well](/images/do_one_thing_well.svg)

As an example, consider a scenario where a codebase consists of many God Classes. Your ability to decouple application into services that focus on a specific domains is severely limited because the basic building blocks of these services (i.e the objects themselves), are extremely broad and unfocussed. In such scenarios, you'll have to refactor those large classes into components that have fewer responsibilities before you can compose them into microservice components with the desired level granularity.

## Information Hiding

Information Hiding refers to the process of writing modules in a way that keeps their details relating to core design decisions, especially those that are expected to change, **hidden** from other objects. As a result, callers of an object are effectively decoupled from it's internal workings, which makes it possible for the 'hidden' parts to change without needing to change the way the object is called.

This principle is also a desirable property for components in micoservices/serverless based architecture. Microservices for example, should be designed as a black box in which the service's internal complexity and details are hidden from other services in the system. Because of this, communication between services will take place via welldefined APIs, which is a desirable trait of microservice architectures.

It should be easy to see now why the process of decoupling a monolith that practices Information Hiding into microservices that behave as black boxes is a straight forward task. Simply put, each class that practices Information Hiding already behaves as a mini black box, which makes composing them into larger microservices that behave as black boxes a trivial exercise. However, if Information Hiding is not properly observed in the original code, you are at risk of decoupling the monolith into microservices that depend on the internal workings of other microservices to provide their service. Obviously, this will hurt the maintainability and evolution of the overall architecture.

## Message Passing

Perhaps the most misunderstood aspect of Object Oriented systems is inter-object communication. In fact, **a method call was not the way modules in an Object Oriented system were intended to communicate**, rather it was through [messaging](http://mfadhel.com/lost-oop/#inter-object-communication).
<br/>
<blockquote class="twitter-tweet tw-align-center"><p lang="en" dir="ltr">Communication in OOP should be thought as message passing, not method calls. Message passing results in smart objects that accomplish tasks together via content negotiation. Method calls result in many unintelligent objects that are constantly told how to do their job. <a href="https://twitter.com/hashtag/oop?src=hash&amp;ref_src=twsrc%5Etfw">#oop</a></p>&mdash; Muntazir Fadhel (@FadhelMuntazir) <a href="https://twitter.com/FadhelMuntazir/status/1103057880520052736?ref_src=twsrc%5Etfw">March 5, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<br/>
The difference between them can be seen in the way we perceive method calls versus messages. Calling a method essentially puts the person making the method call in control of running the process. The caller gets the callee to do something and prevents it from doing something that the caller does not want. Message passing on the other hand revolves around negotiation, and this is the key in building object oriented systems.

A major challenge in converting a monolith into microservices based architecture lies in re-designing it's communication mechanism. Microservices are ideally integrated using asynchronous communication in order to enforce microservice autonomy and to develop an architecture that is resillient when microservices fail or underperform. The problem with all of this is that code is typically written in a procedural and synchronous manner. Breaking such a monolith into microservices will force architects to use a synchornous communication scheme.

On the other hand, code that practices message passing facilitates the decopling of a monolith into microservices that communicate using asynchronous patterns to a great degree. **How?** Because a module that practices message passing properly does not require any response from the client in other for it to continue doing it's job. It does not control it's clients, and assumes that modules recieving messagings are smart enough to do whatever needs to be done next. The end result of composing such objects into microservices is an architecture that can be adapted to use asynchronous communication mechanisms with little effort. 

## Coupling

Coupling represents the degree to which a module or object is independent from others. A highly Coupled module relies on many other modules to do it's job, whereas a slightly Coupled module relies on a few modules to do it's job. 

Object Oriented systems generally avoid highly Coupled modules. As more components are introduced into a system, they become increasingly more difficult to maintain and test. Unsurprisingly, distributed cloud architectures also favour the design of services and components that do not rely on many other components to do it's job. In this case however, the motivating goal is to support **component evolution**.

That is to say, components in a distributed architecture should have the ability to be changed, upgraded, or replaced independently without affecting the functioning of other components. This requires microservices to be designed in a way that makes them primarily responsible for *running their own show*, and gives the teams responsible for different services the autonomy to make decisions and act independently from each other.

Needless to say, a monolithic application **cannot** be broken up into more refined services if the code is highly coupled without re-implementing significant parts of the code base. On the other hand, *slightly* coupled code will help architects decouple monoliths into service based architectures in numberous ways:

1. **Self-forming Dependancy Networks**: 
2. **Microservice Granularities**:
3. 






































