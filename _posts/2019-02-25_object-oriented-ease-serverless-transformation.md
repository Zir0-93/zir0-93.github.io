
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
Classes that take on many responsibilities, also known as God Classes, are very difficult to read, maintain, and extend due to their enormous size. For this reason, classes in an Object Oriented design should **focus** on doing one thing well. If the complexity of a class exceeds a reasonable level, it should be refactored into two or more separate ones.

When we consider that each service in a microservices architecture is designed to focus on a specific domain, this principle gives us a great deal of **flexibility** in determining how we want to decouple a monolithic application into such services. 

![do-one-thing-well](/images/do_one_thing_well.svg)

As an example, consider a scenario where a codebase consists of many God Classes. Your ability to decouple application into services that focus on a specific domains is severely limited because the basic building blocks of these services (i.e the clasess themselves), are overly complex entities with many unrelated responsibilities. In such scenarios, you'll have to refactor those large classes into modularized components before you can develop microservice components with a decreased level of granularity.

## Information Hiding

Information Hiding refers to the process of designing objects in a way that keeps their details relating to core design decisions, especially those that are expected to change, **hidden** from other objects. As a result, callers of an object are effectively decoupled from it's internal workings, which makes it possible for the 'hidden' parts to change without needing to change the way the object is called.

This principle is also a desirable property for components in micoservices/serverless based architecture. Microservices for example, should be designed as a black box in which the service's internal complexity and details are hidden from other services in the system. Because of this, communication between services will take place via welldefined APIs, which is a desirable trait of microservice architectures.

It should be easy to see now why the process of decoupling a monolith that practices Information Hiding into microservices that behave as black boxes is a straight forward task. It's because each class would behave as a mini black box itself, which would make the process of composing them into larger microservices that behave as black boxes a trivial exercise. However, if Information Hiding is not properly observed in the original code, you are at risk of decoupling the monolith into microservices that depend on the internal workings of other microservices to provide their service. Obviously, this is not a desirable characteristic of microservices. 

## Message Passing
 
 

## Abstraction


## Coupling & Cohesion
Different components in a microservices architecture can be changed, upgraded, or replaced independently without affecting the functioning of other components. Similarly, the teams responsible for different microservices are enabled to act independently from each
other. 
