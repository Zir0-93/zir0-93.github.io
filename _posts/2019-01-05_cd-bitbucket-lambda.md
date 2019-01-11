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

# Overview
## Multi-Environment Serverless Architecture
In general, multi-environments setups help teams gain **confidence** in the ability of newly developed software to perform as expected before it is delivered into production. We'll use a multi-stack approach to setup a `staging` and `prod` environment for our lambda functions, API Gateways, and databases as illustrated in the diagram below. Note that in this setup, seperate API Gateways and Datastore instances are created for each environment. And unlike our Lambda functions, they do not need to be updated every time new code is delivered. In the proposed system, any new code deliveries will be pushed to our staging environment where it will run for a few days. If it performs as expected, our contious delivery pipelines will push the changes into our production environment via a manual trigger.

![staging_prod_architecture](/images/staging_prod.png)

### Multi-Stack vs Single Stack Architecture
The single stack approach shares its API Gateway and Lambda functions across all environments, and uses stages, environment variables and Lambda aliases to differentiate between environments. The multi-stack appraoch demonstrated in this post uses completely separate instances of each service instead and not API stages or Lambda aliases to seperate environment. The main reason for this decision is to **minimize risk**. In the event that something goes wrong in a single stack approach, there is a much higher likelihood that your production systems are affected. This likelyhood is greatly minimized in a multi-stack approach where entirely sepearate instances are deployed for each environment.

### Process Flow

![staging_prod_architecture](/images/process.png)

