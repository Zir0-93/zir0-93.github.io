---
title:  "Gain Clarity Series - Extensibility in Junit5"
date:   2016-09-08 15:04:23
categories: [gain clarity series]
excerpt_separator: <!--more-->
tags: [tech, code, clarity views, junit5, static analysis, uml]
---
The Gain Clarity Series aims to uncover architectural designs in some of the most popular developer frameworks and libraries.
Today we will use [Clarity Views](http://clarityviews.io) to analyze major architectural changes that have come about in [Junit5](https://github.com/junit-team/junit5)
[![Clarity Views Label](http://clarityviews.io/badge)](http://clarityviews.io/github/junit-team/junit5). One of the main drawbacks that existed in Junit-4 was the lack of separation of concerns between the various core mechanisms
that existed in Junit. Other test engines, extensions and build tools built on top of Junit needed to reach
deep into Junit-4's internals to implement much needed features. As a result, while Junit-4 was extremely successful as a platform,
its maintainers could not enhance Junit as a tool as much as they would have liked without breaking all the external tools that were built on top of it.
<!--more-->
 
## The Junit-5 Platform Launcher
The Junit Platform Launcher component was introduced to provide a **uniform** and much more powerful set of API's for external tools and IDE's to interact with test
execution (launching tests, viewing test results, etc..). Upon further inspection of the DefaultLauncher class, we can see that it is able to
execute tests and register TestExecutionListeners which will are responsible for getting feedback about the progress and results of the test execution.
![launcher](/images/launcher.svg)


## The Junit-5 Test Engine
One further responsibility of the Launcher, as documented in the Launcher interface, is to decide
what Test Engine's to delegate the execution of tests to at runtime. A Test Engine instance essentially has two main functionalities.

1. Execute tests
2. Discover what tests it is actually capable of executing
   
A Launcher instance is able to decide what Test Engine to delegate tests to by passing a LauncherDiscoveryRequest instance to each
registered Test Engine (found dynamically via Java's [ServiceLoader](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html)
mechanism). Each Test Engine then returns a Test Plan indicating what tests it can discover and later execute.
As you can see below, there are multiple Test Engine implementations within
the Junit5 code base itself.

```
![TestEngine](http://clarityviews.io/embed/junit-team/junit5/master/diagram/junit5-master/junit-platform-engine/src/main/java/org/junit/platform/engine/TestEngine.java)
```

![TestEngineDiagram](http://clarityviews.io/embed/junit-team/junit5/master/diagram/junit5-master/junit-platform-engine/src/main/java/org/junit/platform/engine/TestEngine.java)


## The Jupiter Test Engine
If you've come this far, you've probably realized why Junit-5's architecture makes developing extensions for Junit much easier.
Junit-5 now supports a model where multiple Test Engines (not just Junit-5), each complete with their own implementations of
executing tests, exist under a common Launcher. Because they all dynamically plug into the same Launcher infrastructure, IDE's and build tools that support the Junit Platform will be able to run these Test Engines through a single, uniform API. In fact, new Test Engine 
implementations can be supported by such tools without requiring any changes on the tooling side!

Junit-5 introdued its own Test Engine implementation (diagrammed below)
which as you would expect  is responsible
for discovering and executing tests written using Junit-5's [brand new APIs](http://junit.org/junit5/docs/current/user-guide/#writing-tests-dynamic-tests). The Vintage Test Engine was another Test Engine implementation
created in Junit-5 to executing tests written in Junit-4 and Junit-3 by the Junit Platform. 

```
![Clarity Views Diagram](http://clarityviews.io/embed/junit-team/junit5/master/diagram/junit5-master/junit-jupiter-engine/src/main/java/org/junit/jupiter/engine/JupiterTestEngine.java)
```

![JupiterDiagram](http://clarityviews.io/embed/junit-team/junit5/master/diagram/junit5-master/junit-jupiter-engine/src/main/java/org/junit/jupiter/engine/JupiterTestEngine.java)

We hoped you enjoyed this article and as always, all diagrams were generated using [Clarity Views](http://clarityviews.io)!
If you would like, feel free to explore Junit-5 for yourself! [![Clarity Views Label](http://clarityviews.io/badge)](http://clarityviews.io/github/junit-team/junit5)
