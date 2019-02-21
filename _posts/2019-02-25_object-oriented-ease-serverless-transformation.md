
---
title:  "How to Avoid a Serverless Transformation Nightmare Through Object Oriented Code"
date:   2019-02-25 15:04:23
icon: /images/badges_csharp_objects_stage01.png
tags: [oop, serverless]
description: 
excerpt_separator: 
---
Even with the best solution architects and resources available, your application's serverless transformation journey will be a nightmare if your code is **not** *object oriented*.
<!--more-->

![serverless_meme](/images/2u3jzm.jpg)

When Lambda functions were first introduced, they were mostly used as simple event based mechanisms for processing and transforming data. Common use cases included analyzing logs, transforming records, and storing data. Today, the level of sophistication and variety of solutions enabled by serverless technology has increased drastically. Powering web applications and microservice architectures using Lambda functions has become common practice.

Wanting to benefit from scalability and cost-efficiency of serverless technology, organizations are increasingly migrating their monolothic applications to serverless architectures. This migration can be a very difficult process.

**Why?** Because the most significant part of the migration process will involve deconstructing the application into components and services. If your code is procedural in nature, you're in a for a long and painful process. However, if you're code is clean, object oriented, and organized, than it can easily be adapted for a serverless environment.

In this article I'll outline **four design principles** for Serverless functions and demonstrate how an object oriented code base faciltates the transformation of a monolithic application into functions that can satisfy these principles.

## Decentralized

Serverless architectures are distributed systems with decentralized data management. One consequence of this principle is that they don’t rely on a unifying schema in a central database. Each microservice has its own view on data models. Microservices are also decentralized in the way they are developed, deployed, managed, and operated. 


## Independent 

Different components in a microservices architecture can be changed, upgraded, or replaced independently without affecting the functioning of other components. Similarly, the teams responsible for different microservices are enabled to act independently from each
other. 


## Message Passing
 
 
## Do one thing well
Each microservice component is designed for a set of capabilities and focuses on a specific domain. If developers contribute so much code to a particular component of a service that the component reaches a certain level of complexity, then the service could be split into two or more services.

## Black box

Individual microservice components are designed as black boxes, that is, they hide the details of their complexity from other components. Any communication between services happens via welldefined APIs to prevent implicit and hidden dependencies.
