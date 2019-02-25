
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

When we consider that each microservice component in a microservices architecture is designed to focus on a specific domain, this principle gives us a great deal of **flexibility** in determining how we want to decouple a monolithic application into smaller components. 

![do-one-thing-well](/images/do_one_thing_well.svg)

As an example, consider a scenario where a codebase consists of many God Classes. Your ability to decouple application into services that focus on a specific domains is severely limited because the basic building blocks of these services (the clasess themselves), are large, complex entities with many unrelated responsibilities. In such cases, you'll have to refactor those large classes into more modularized components before you can develop microservice components with the correct level of granularity.

## Encapsulation

Individual microservice components are designed as black boxes, that is, they hide the details of their complexity from other components. Any communication between services happens via welldefined APIs to prevent implicit and hidden dependencies.

## Message Passing
 
 

## Abstraction


## Coupling & Cohesion
Different components in a microservices architecture can be changed, upgraded, or replaced independently without affecting the functioning of other components. Similarly, the teams responsible for different microservices are enabled to act independently from each
other. 
