---
title:  "On Comparing The Performance Between Bluemix PaaS Instances and IBM Containers"
date:   2016-09-12 15:04:23
categories: [tech, bluemix, docker, PaaS, containers, IBM]
tags: [tech, bluemix, docker, PaaS, containers, IBM]
---
One of the most important decisions regarding the development of any software application revolves around the 
choice of technology used to host and run the application in production. Moreover, when we talk about cloud applications, the number
of vendors and solutions that can used for hosting drastically increases. This article will compare the **performance** 
[PaaS](https://www.ibm.com/cloud-computing/ca/en/paas.html) and [Container](https://www.ibm.com/cloud-computing/bluemix/containers/) based deployment solutions provided by the IBM Cloud. Please not that performance is not the only indicator of whether you should use PaaS vs Containers and the results presented below were collected using the IBM Cloud (do keep in mind however that Bluemix's PaaS is based of [Pivotal's Cloud Foundry[(https://pivotal.io/platform)).

## Deployment Solution A: IBM Bluemix PaaS

PaaS offerings such as IBM's Bluemix allow developers to run their applications in the cloud without having to worry about any infrastructure. To push applications to the cloud, developers only push their (web) applications, all the remaining infrastructure is taken
case of by the platform. Furthermore, Bluemix provides scalability of applications and auto-recovery, logging dashboards and more administration functionality right out of the box. Lastly, developers can develop their applications in their local IDEs of choice, run and test thier applications on local servers and simply push the applications to the cloud. While developers have a great deal of functionality out of the box, they are not typically able to configure the underlying system running their apps in a PaaS environment.

## Deployment Solution B: IBM Containers

Docker is a container technology to package full application stacks so that these containers can easily be run in different environments. This portability is achieved by packaging not only the core applications (your own code) but also the complete underlying stack you need to run applications including application servers, Java runtimes, configuration and other dependencies. Docker uses Linux as operating system and leverages concepts like Linux namespaces to run multiple applications in sandboxes. Containers are more portable since all you need are Linux Docker environments while for Cloud Foundry packaged applications you have more prerequisites, the Cloud Foundry runtimes and services. Unlike PaaS environments, containers allow developers more flexibility to configure the infrastructure running their applications
and therefore require additional resources to setup.

## Experimental Setup
The [Clarpse](http://mfadhel.com/2016/clarpse/) Source Code Analysis project was packaged as a web application with a single end point that
1. Accepts a JSON list of source files
2. Parses these files
3. Returns a JSON object representing the parsed output

The basic test case simply routes a request to this endpoint (either deployed on Bluemix PaaS or IBM containers) and calculates the amount of time it takes to recieve the response. To set up the deployment environments, the web application was exported as a WAR file and pushed
 to the the Bluemix Paas Environment (1 x 256MB instance). The same WAR file was also installed within a default Tomcat instance in an IBM container (1 x 256MB instance). 

## Simple Parse Tests
This test consisted of sending twenty requests, one at a time, to each of the two deployment solutions, the results are presented below.


