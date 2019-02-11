
---
title:  "Why Good Serverless Functions Must Follow Object Oriented Principles"
date:   2019-02-25 15:04:23
icon: /images/badges_csharp_objects_stage01.png
tags: [oop, messaging, microservices]
description: 
excerpt_separator: <!--more-->
---
Many people will start their cloud adoption journey by breaking down their existing application into smaller components hoping to migrat. them to Lambda functions. This can result in a lot of frustration, and time wasted in running doing Serverless when it was supposed to save you time and money. 

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
• Polyglot – Microservices architectures don’t follow a “one size fits all”
approach. Teams have the freedom to choose the best tool for their
specific problems. As a consequence, microservices architectures take a
heterogeneous approach to operating systems, programming languages,
data stores, and tools. This approach is called polyglot persistence and
programming.
Amazon Web Services – Microservices on AWS
Page 2
• Black box – Individual microservice components are designed as black
boxes, that is, they hide the details of their complexity from other
components. Any communication between services happens via welldefined APIs to prevent implicit and hidden dependencies.
• You build it; you run it – Typically, the team responsible for building
a service is also responsible for operating and maintaining it in
production. This principle is also known as DevOps.
3 DevOps also helps
bring developers into close contact with the actual users of their software
and improves their understanding of the customers’ needs and
expectations. The fact that DevOps is a key organizational principle for
microservices shouldn’t be underestimated because according to
Conway’s law, system design is largely influenced by the organizational
structure of the teams that build the system.4
