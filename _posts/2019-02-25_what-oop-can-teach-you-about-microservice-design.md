
---
title:  "How to Write Better Serverless Functions Using Object Oriented Principles"
date:   2019-02-25 15:04:23
icon: /images/badges_csharp_objects_stage01.png
tags: [oop, serverless]
description: 
excerpt_separator: <!--more-->
---
Most people *underestimate* the complexity of writing code for a serverless environment. 

At a high level, the required code changes seem like a matter of refactoring components of software from a larger code base. This, along with migrating any data storage and authentication  them to Lambda functions. This can result in a lot of frustration, and time wasted in running doing Serverless when it was supposed to save you time and money. 

Writing code for Serverless applications requires a much different approach than writing code for typical monolithic applications.




• Decentralized – Microservices architectures are distributed systems
with decentralized data management. They don’t rely on a unifying
schema in a central database. Each microservice has its own view on
data models. Microservices are also decentralized in the way they are
developed, deployed, managed, and operated.

• Independent – Different components in a microservices architecture
can be changed, upgraded, or replaced independently without affecting
the functioning of other components. Similarly, the teams responsible
for different microservices are enabled to act independently from each
other.

• Do one thing well – Each microservice component is designed for a
set of capabilities and focuses on a specific domain. If developers
contribute so much code to a particular component of a service that the
component reaches a certain level of complexity, then the service could
be split into two or more services.

• Black box – Individual microservice components are designed as black
boxes, that is, they hide the details of their complexity from other
components. Any communication between services happens via welldefined APIs to prevent implicit and hidden dependencies.
