--- 
title:  "How to Setup a Continuous Delivery Environment for Your Lambda Functions on Bitbucket"
image: /images/rocket_cd.png
date:  2019-01-05 15:04:23
tags: [python, AWS, lambda, continuous delivery, bitbucket]
description: Continuous delivery is an approach where teams produce production **ready** products frequently and predictably from source code in an automated fashion. In this tutorial, I'll demonstrate how to execute a continuous delivery strategy for you Lambda functions deployed on AWS. We'll assume we're working in a team environment with many developers, where any delivered code must pass through testing in several environments (test, staging, etc..) before being deployed to production using a manual trigger.
excerpt_separator: <!--more-->
---
Continuous delivery is an approach where teams produce production **ready** products frequently and predictably from source code
in an automated fashion. In this tutorial, I'll demonstrate how to execute a continuous delivery strategy for you Lambda 
functions deployed on AWS. We'll assume we're working in a team environment with many developers, where any delivered code must 
pass through testing in several environments (test, staging, etc..) before being deployed to production using a manual trigger.

![cd_img](/images/Continuous-Delivery-and-Deployment.jpg)

<!--more-->
