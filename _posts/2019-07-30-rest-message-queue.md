--- 
title:  "3 Common Misunderstandings of Inter-Service Communication in MicroServices"
date:  2019-12-05 15:04:23
og_image: /images/microservices.png
redirect_to: "https://www.hadii.ca/insights/microservice-communication"
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
