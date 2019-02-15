
---
title:  "Why Object Oriented Code Can Streamline Your Serverless Transformation Journey"
date:   2019-02-25 15:04:23
icon: /images/badges_csharp_objects_stage01.png
tags: [oop, serverless]
description: 
excerpt_separator: <!--more-->
---
Most people *underestimate* the complexity of writing code for a serverless environment. 

When Lambda functions were first introduced, they were mostly used as simple event based mechanisms for processing and transforming data: Analyzing logs, transforming records, backing up data, and so on. Today, the level of sophistication and number of supported business use cases enabled by serverless technology has increased drastically. Using Lambda functions to implement web applications and microservice architectures has become common practice.

However, writing a Lambda function to serve as a component of your microservice architecture is usually much more difficult than writing one to just insert records into a database. Thats' becasue such an endeavour requires you to spend more time thinking about authentication, performance, communication, exposed interfaces and more. Most importantly of all, it requires you to write more code - a lot more code.

And with more code comes increased complexity. 




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

 Is there over-reliance between services? (Message Passing)
 
 
• Do one thing well – Each microservice component is designed for a
set of capabilities and focuses on a specific domain. If developers
contribute so much code to a particular component of a service that the
component reaches a certain level of complexity, then the service could
be split into two or more services.

• Black box – Individual microservice components are designed as black
boxes, that is, they hide the details of their complexity from other
components. Any communication between services happens via welldefined APIs to prevent implicit and hidden dependencies.
