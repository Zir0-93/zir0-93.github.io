--- 
title:  "How to Setup a Continuous Delivery Environment for Your AWS Lambda Functions on Bitbucket"
image: /images/rocket_cd.png
date:  2019-01-05 15:04:23
tags: [python, AWS, lambda, continuous delivery, bitbucket]
description: *Continuous delivery* is an approach where teams produce **production ready** products frequently and predictably from source code in an automated fashion. In this tutorial, I'll demonstrate how to execute a continuous delivery strategy for you Lambda functions deployed on AWS. We'll assume we're working in a team environment with many developers, where any delivered code must pass through testing in several environments (test, staging, etc..) before being deployed to production using a manual trigger.
excerpt_separator: <!--more-->
---
*Continuous delivery* is an approach where teams produce **production ready** products frequently and predictably from source code
in an automated fashion. In this post, I'll demonstrate how to execute a continuous delivery strategy for you Lambda 
functions deployed on AWS. We'll assume we're working in a team environment with many developers, where any delivered code must 
pass through testing in several environments (test, staging, etc..) before being deployed to production using a manual trigger.
<!--more-->

![cd_img](/images/Continuous-Delivery-and-Deployment.jpg)

# Multi-Environment Serverless Architecture
In general, multi-environments setups help teams gain **confidence** in the ability of developed software to perform as expected before it is delivered into production. We'll use a multi-stack approach to setup a `staging` and `prod` environment for our lambda functions, API Gateways, and databases, the architecture of which is depicted below. The idea is that any new code deliveries will sit in the staging environment for a few days. If it performs as expected, only then will it be incorporated into the production environment.

![staging_prod_architecture](/images/staging_prod.png)

# Branching Model


tes
