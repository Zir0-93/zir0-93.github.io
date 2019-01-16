--- 
title:  "Best Practices for Multi-Environment Serverless Architectures"
image: /images/rocket_cd.png
date:  2019-01-05 15:04:23
tags: [python, AWS, lambda, continuous delivery, bitbucket]
description: *Serverless* is the new Buzz word in town being promoted by cloud service providers designed to allow organizations to focus on **developing applications**, and not on **managing infrastructure**. However, the need for minimizing the risk in the software development cycle for these applications is not such a new concept. In most organizations, code delivered by developers is typically required to flow through multiple environments (test, staging, etc..) to ensure it works as expected before it is deployed to production. Moreover, Servless Architectures introduce a unique set of challenges that need to be considered when running a multi-environment setup. In this article, I'll go over a few best practices for managing multi-environment serverless architectures.
excerpt_separator: <!--more-->
---
*Serverless* is the new Buzz word in town being promoted by cloud service providers designed to allow organizations to focus on **developing applications**, and not on **managing infrastructure**. However, the need for minimizing the risk in the software development cycle for these applications is not such a new concept. In most organizations, code delivered by developers is typically required to flow through multiple environments (test, staging, etc..) to ensure it works as expected before it is deployed to production. Moreover, Servless Architectures introduce a unique set of challenges that need to be considered when running a multi-environment setup. In this article, I'll go over a few best practices for managing multi-environment serverless architectures.
<!--more-->

![cd_img](/images/Continuous-Delivery-and-Deployment.jpg)

# Multi-Stack Vs Single Stack Serverless Architecture Approach
In order to minimize the risk introduced by any code integrations, organizations typically setup multiple environments (test, staging, etc..) designed to test the robustness of any delivered code before it is pushed into production. When configuring these environments using a serverless architecture however, an important decision must be made as to whether a single stack or multi-stack strategy should be used.

![staging_prod_architecture](/images/staging_prod.png)

The single stack approach shares its API Gateway and Lambda functions across all environments, and uses stages, environment variables and Lambda aliases to differentiate between environments. In contrast, a multi-stack approach uses a completely separate instance of each service for every environment and refrains from utilizing API stages or Lambda aliases to so. The main differences between the two approaches is that the multi-stack approach **minimizes risk**, while the single stack approach minimizes **configuration/management effort**.


For example, in the event in which something goes wrong in a single stack approach, there is a much higher likelihood that your production systems are negatively impacted as well. This is due to this approach's reliance on environment variables and lambda aliases for indirection. In the multi-stack approach, this probability is greatly reduced as a result of the use of separate resources for each environment. 

## Store App Config In the Environment


# Use A Serverless Framework

Now that the general architecture of our multi-environment setup has been outlined, I'll now describe what steps take place in our system from the time a developer begins working on a new feature to the time that feature is merged into production.


![staging_prod_architecture](/images/process.png)

