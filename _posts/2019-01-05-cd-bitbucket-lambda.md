--- 
title:  "Best Practices for Multi-Environment Serverless Architectures"
image: /images/rocket_cd.png
date:  2019-01-05 15:04:23
tags: [python, AWS, lambda, continuous delivery, bitbucket]
description: *Serverless* is the new Buzz word in town being promoted by cloud service providers designed to allow organizations to focus on **developing applications**, and not on **managing infrastructure**. However, the need for minimizing the risk in the software development cycle for these applications is not such a new concept. In most organizations, code delivered by developers is typically required to flow through multiple environments (test, staging, etc..) to ensure it works as expected before it is deployed to production. That said, Servless Architectures introduce a unique set of challenges that need to be considered when running a multi-environment setup. In this article, I'll go over a few best practices for managing multi-environment serverless architectures.
excerpt_separator: <!--more-->
---
*Serverless* is the new Buzz word in town being promoted by cloud service providers designed to allow organizations to focus on **developing applications**, and not on **managing infrastructure**. However, the need for minimizing the risk in the software development cycle for these applications is not such a new concept. In most organizations, code delivered by developers is typically required to flow through multiple environments (test, staging, etc..) to ensure it works as expected before it is deployed to production. That said, Servless Architectures introduce a unique set of challenges that need to be considered when running a multi-environment setup. In this article, I'll go over a few best practices for managing multi-environment serverless architectures.
<!--more-->

![cd_img](/images/Continuous-Delivery-and-Deployment.jpg)

# Multi-Environment Approach - Multi-Stack or Single Stack?
In order to minimize the risk introduced by any code integrations, organizations typically setup multiple environments (test, staging, etc..) designed to test the robustness of any delivered code before it is pushed into production. When configuring these environments in a serverless environment however, an important decision must be made as to whether a single stack or multi-stack strategy is most appropriate.

![staging_prod_architecture](/images/staging_prod.png)

The single stack approach shares its API Gateway and Lambda functions across all environments, and uses stages, environment variables and Lambda aliases to differentiate between environments. In contrast, a multi-stack approach uses a completely separate instance of each service for every environment and refrains from utilizing API stages or Lambda aliases to so. The main differences between the two approaches is that the multi-stack approach **minimizes risk**, while the single stack approach minimizes **configuration/management effort**.

![staging_prod_single_architecture](/images/staging_prod_single_stack.png)

For example, in the event in which something goes wrong in a single stack approach, there is a much higher likelihood that your production systems are negatively impacted as well. This is due to this approach's reliance on environment variables and lambda aliases for indirection. In the multi-stack approach, this probability is greatly reduced as a result of the use of separate resources for each environment. 

# Adopt Continuous Delivery
The main idea behind Continuous delivery is to produce **production ready** artifacts from your code base frequently in an **automated fashion**. It  ensures that code can be rapdily and safely deployed to production by delivering every change to a production-like environment and ensuring business applications and services function as expected through rigorous automated testing. The most important point however, is that code must be tested and deployed (to environments other than production) using complete automation.

Using their front end web application, Cloud providers such as Amazon Web Services make it easy to spin off a new lambda function for testing purposes or to update a function's application code. However, all aspects of your continous delivery strategy should be automated - the only manual step should be the push of the `deploy to production` button. This is vital for two reasons:

1. **Minimizing Risk** - Even with proper resource access controls in place, which is seldom the case, the risk of accidentally causing bad things to happen are higher than you think. You definitely don't want be the reason for a disruption to your service.
2. **Traceability** - When your continuous delivery strategy is an automated, often using an automated pipeline of some sort, the use of  deployments, builds and other components are better documented, making it easier to resolve issues and manage your multi-environment setup in general.


<script src="https://gist.github.com/Zir0-93/85b1d1c7433049c8d41de082366c362c.js"></script>


Additionally, 
# Store App Config In the Environment


# Use A Serverless Framework




Now that the general architecture of our multi-environment setup has been outlined, I'll now describe what steps take place in our system from the time a developer begins working on a new feature to the time that feature is merged into production.


![staging_prod_architecture](/images/process.png)

