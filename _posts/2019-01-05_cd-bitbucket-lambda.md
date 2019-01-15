--- 
title:  "How to Setup Continuous Delivery for Serverless Architectures on AWS"
image: /images/rocket_cd.png
date:  2019-01-05 15:04:23
tags: [python, AWS, lambda, continuous delivery, bitbucket]
description: The quality of a code integration process is is measured by it's ability to integrate multiple code deliveries quickly while minimizing the risk to the end product as a result of those deliveries. A Continuous Delivery pipeline is a type of code integration system in which a production a ready version of source code is developed frequently and predictably from source code in an automated manner. In this system, efficiency is achieved through an automated pipeline which runs automated tests, static analysis tools, pushes code to environments, and much more. Concurrently, automated tests, along with multiple testing environments are incorporated to help minimize the risk of any delivered code. This article outlines fundamental considerations and strategies for setting up a continuous delivery pipeline for a serverless architecture on AWS Lambda using Bitbucket Pipelines.
excerpt_separator: <!--more-->
---
The quality of a code integration process is is measured by it's ability to integrate multiple code deliveries quickly while minimizing the risk to the end product as a result of those deliveries. A Continuous Delivery pipeline is a type of code integration system in which a production a ready version of source code is developed frequently and predictably from source code in an automated manner. In this system, efficiency is achieved through an automated pipeline which runs automated tests, static analysis tools, pushes code to environments, and much more. Concurrently, automated tests, along with multiple testing environments are incorporated to help minimize the risk of any delivered code. This article outlines fundamental considerations and strategies for setting up a continuous delivery pipeline for a serverless architecture on AWS Lambda using Bitbucket Pipelines.
<!--more-->

![cd_img](/images/Continuous-Delivery-and-Deployment.jpg)

# Serverless Architecture Overview
In order to minimize risk of any delivered, organizations typically require any code deliveries to flow through multiple environments to ensure it works properly before it is deployed to production. When configuring these environments using a serverless architecture however, an important decision must be made as to whether a single stack or multi-stack strategy should be used.

The single stack approach shares its API Gateway and Lambda functions across all environments, and uses stages, environment variables and Lambda aliases to differentiate between environments. This is the approach assumed in this article. In contrast, the multi-stack approach uses a completely separate instance of each service for every environment and refrains from utilizing API stages or Lambda aliases to so. The main differences between the two approaches is that the multi-stack approach **minimizes risk**, while the single stack approach minimizes **configuration/management effort**.

![staging_prod_architecture](/images/staging_prod.png)

For example, in the event in which something goes wrong in a single stack approach, there is a much higher likelihood that your production systems are negatively impacted as well. This is due to this approach's reliance on environment variables and lambda aliases for indirection. In the multi-stack approach, this probability is greatly reduced as a result of the use of separate resources for each environment. However, the single stack approach is easier to setup and manage, and we'll use this approach for the purposes of demonstrating our continuous delivery pipeline in this article.


# Process Flow
Now that the general architecture of our multi-environment setup has been outlined, I'll now describe what steps take place in our system from the time a developer begins working on a new feature to the time that feature is merged into production.


![staging_prod_architecture](/images/process.png)

