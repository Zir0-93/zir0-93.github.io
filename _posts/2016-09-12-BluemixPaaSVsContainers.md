---
title:  "Comparing The Performance of Bluemix PaaS Instances and IBM Containers"
date:   2016-09-12 15:04:23
image: /images/bluemix.png
tags: [bluemix, docker, PaaS, containers, IBM]
description: One of the most important decisions regarding the development of any software application revolves around the choice of technology used to host and run the application in production. Moreover, when we talk about cloud applications, the number of vendors and solutions that be can used for hosting drastically increases. This article will compare the **performance** of Platform-As-A-Service [PaaS](https://www.ibm.com/cloud-computing/ca/en/paas.html) and [Container](https://www.ibm.com/cloud-computing/bluemix/containers/) based deployment solutions provided by the IBM Cloud.

excerpt_separator: <!--more-->
---
One of the most important decisions regarding the development of any software application revolves around the 
choice of technology used to host and run the application in production. Moreover, when we talk about cloud applications, the number
of vendors and solutions that be can used for hosting drastically increases. This article will compare the **performance** of 
Platform-As-A-Service [PaaS](https://www.ibm.com/cloud-computing/ca/en/paas.html) and [Container](https://www.ibm.com/cloud-computing/bluemix/containers/) based deployment solutions provided by the IBM Cloud.
<!--more-->

## Deployment Solution A: IBM Bluemix PaaS


![ibmbluemix](/images/bluemix_banner.png)

PaaS offerings such as IBM's Bluemix (built on top of Pivotal's [CloudFoundry](https://github.com/cloudfoundry/cf-release)) allow developers to run their web applications in the cloud without having to worry about any infrastructure. As a result, developers can develop their applications in their local IDEs of choice, run and test their applications on local servers and simply push the applications to the cloud. Bluemix also allows developers to bind a variety cloud services (Database, analytics, etc..) to their cloud app dynamically. One downside of PaaS environments however is the lack of freedom given to customers in configuring/tuning their deployment environment.

## Deployment Solution B: IBM Containers

![ibmcontainer](/images/containers.png)

 Containers wrap-up an application in a self-contained filesystem which includes all the dependencies the app requires to run: binaries, runtime libraries, system tools, system packages, etc. This model allows for portability and allows applications to be deployed in a very consistent and predictive way. Unlike PaaS environments, containers allow developers to configure the environments running their applications.

## Experimental Setup
The [Clarpse](http://mfadhel.com/2016/clarpse/) Source Code Analysis project was packaged as a web application with a single REST end point that will:

1. Accept a JSON list of source files
2. Parse these files
3. Return a JSON object representing the parsed output

The basic test case simply routes a request (JSON list of 100 java source files) to this endpoint (either deployed on Bluemix PaaS or IBM containers) and calculates the amount of time it takes to recieve the response. To set up the deployment environments, the web application was exported as a WAR file and pushed
 to the the Bluemix Paas Environment (1 x 256MB instance). The same WAR file was also installed within a default Tomcat instance in an IBM container (1 x 256MB instance). 

## Simple Parse Tests
This test consisted of sending 20 requests, one at a time, to each of the two deployment solutions, the results are presented below.

![simplecontainertest](/images/simplecontainertestz.png)


![simplePaaSTest](/images/simplePaasTestz.png)


## Burst Tests
This test consisted of sending five asynchronous requests, 20 times, to each of the two deployment solutions. The results
are presented below:

![containerbursttest](/images/singlecontainerbursttestz.png)


![PaaSBurstTest](/images/PaasBurstTestz.png)


The Bluemix PaaS instance has nine results rather than 20 because it ran out of memory and crashed. This
experiment originally involved sending 20 asynchronous requests instead of five, but the Bluemix PaaS 
instance kept crashing before a single response was delivered. Nine results are good enough to keep
moving forward at this point!

## Concluding Remarks
<br>

![testtable](/images/testtable.PNG)
<br>
<br>
Overall the containers seem to outperform Bluemix PaaS instances by a large margin. This is surprising
because containers are what power PaaS environments so in theory there should not be a significant difference in the performance. 
What is even more surprising however, is the cost of Bluemix PaaS instances compared to that of IBM Containers. At present, 4 1-GB Bluemix
PaaS instances cost $175.35 and 4 1-GB IBM Containers $72.43. 
