
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

**Why?** Because whether you are implementing a microservices or serverless based architecture, the most time consuming part of the migration process is the **decoupling** of the monolith base into smaller, more refined components.

And this decoupling process will be much more frustrating and tedious than neccessary if the code does not have certain **characteristics** that make it easy *to split up into smaller pieces*. 

In this article I'll outline four principles of object oriented code and how each of them **faciltate** the process ofdecoupling of a monolithic application into smaller components intended to run in the cloud.

## Single Responsibility Principle

Each microservice component is designed for a set of capabilities and focuses on a specific domain. If developers contribute so much code to a particular component of a service that the component reaches a certain level of complexity, then the service could be split into two or more services.


## Encapsulation

Individual microservice components are designed as black boxes, that is, they hide the details of their complexity from other components. Any communication between services happens via welldefined APIs to prevent implicit and hidden dependencies.

## Message Passing
 
 

## Abstraction


## Coupling & Cohesion
Different components in a microservices architecture can be changed, upgraded, or replaced independently without affecting the functioning of other components. Similarly, the teams responsible for different microservices are enabled to act independently from each
other. 
