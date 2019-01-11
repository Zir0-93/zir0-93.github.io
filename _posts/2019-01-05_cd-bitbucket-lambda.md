--- 
title:  "How to Setup Continuous Delivery for Serverless Architectures on AWS"
image: /images/rocket_cd.png
date:  2019-01-05 15:04:23
tags: [python, AWS, lambda, continuous delivery, bitbucket]
description: The quality of a code integration system is is measured by it's ability to integrate multiple code deliveries quickly while minimizing the risk to the end product as a result of those deliveries. A Continuous Delivery pipeline is a type of code integration system in which a production a ready version of source code is developed frequently and predictably from source code in an automated manner. In this system, efficiency is achieved through an automated pipeline which runs automated tests, static analysis tools, pushes code to environments, and much more. Concurrently, the incorporation of automated tests along with the use of multiple testing environments help minimize the risk of any code deliveries into the system. This article outlines fundamental considerations and strategies for setting up a continuous delivery pipeline for a serverless architecture on AWS Lambda.
excerpt_separator: <!--more-->
---
The quality of a code integration system is is measured by its ability to support frequent code deliveries while minimizing the risk to the end product as a result of those deliveries. A Continuous Delivery pipeline is a type of code integration system in which a production a ready version of source code is developed frequently and predictably from source code in an automated manner. In this system, efficiency is achieved through an automated pipeline which handles the execution of static analysis tools and automated tests, as well as the packaging and deployment of the software. Concurrently, the incorporation of automated tests along with the use of multiple testing environments help minimize the risk of any code deliveries into the system. This article outlines fundamental considerations and strategies for setting up a continuous delivery pipeline for a serverless architecture on AWS Lambda.
<!--more-->

![cd_img](/images/Continuous-Delivery-and-Deployment.jpg)

# Serverless Architecture Overview
In order to minimize risk of any code integrations, organizations typically require any code deliveries to flow through multiple environments to ensure it works properly before it is deployed to production. When configuring multiple environments using a serverless architecture however, an important decision must be made as to whether a single stack or multi-stack strategy will be used.

The single stack approach shares its API Gateway and Lambda functions across all environments, and uses stages, environment variables and Lambda aliases to differentiate between environments. In contrast, the multi-stack approach which is demonstrated in this article, uses a completely separate instance of each service for every environment and refrains from utilizing API stages or Lambda aliases to so. The main reason for this decision is because we have chosen to **minimize risk**. In the event that something goes wrong in a single stack approach, there is a much higher likelihood that your production systems are negatively impacted as well. This probability greatly decreases in a multi-stack approach in which entirely separate resources are deployed for each environment. This approach definitely requires more work up-front, and organizations should weigh the pros and cons of each method to decide what setup is best for their particular use case.

![staging_prod_architecture](/images/staging_prod.png)

Our multi-stack approach maintains separate lambda functions, api gateways and data stores for each environment.


# Process Flow
Now that the general architecture of our multi-environment setup has been outlined, I'll now describe what steps take place in our system from the time a developer begins working on a new feature to the time that feature is merged into production.


![staging_prod_architecture](/images/process.png)

