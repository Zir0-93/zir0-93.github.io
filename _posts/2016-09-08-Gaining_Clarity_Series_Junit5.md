---
title:  "Gain Clarity Series - Extensibility in Junit5"
date:   2016-09-08 15:04:23
categories: [gain clarity series]
excerpt_separator: <!--more-->
tags: [tech, code, clarity views, junit5, static analysis, uml]
---
The Gain Clarity Series aims to uncover architectural desigs in some of the most popular developer frameworks and libraries.
Today we will use [Clarity Views](http://clarityviews.com) to analyze major architectural changes that have come about in [Junit5](https://github.com/junit-team/junit5)
[![Clarity Views Label](http://clarityviews.com/badge)](http://clarityviews.com/github/junit-team/junit5). One of the main drawbacks that existed in Junit4 was the lack of separation of concerns between the various core mecahnisms 
that existed in Junit. For this reason, other test engines, extensions and buld tools built on top of Junit needed to reach
deep into Junit 4's internals to implement much needed features. As a result, while Junit4 was extremely successfull as a platform,
its maintainers could not enhance Junit the tool as much as they would have liked without breaking all the third party tools that depended on it. 
 <!--more-->
 
## The Junit5 Platform Launcher
The Junit Platfrom Launcher component was introduced to provide a **uniform** and much more powerful set of API's for external tools and IDE's to interact with test
execution (launching tests, viewing test results, etc..). Upon further inspection of the DefaultLauncher class, we can see that it is able to
execute tests and register the all important TestExecutionListeners which will will get feedback about the progress and results of test execution. 
![launcher](/images/launcher.svg)


## The Junit5 Test Engine
One further responsibility of the Launcher, as documented in the Launcher interface, is to determine 
what Test Engine's to delegate the execution of tests to at runtime. A TestEngine essentially has two main functionalities.

1. Execute tests 
2. Discover what tests it is actually capable of executing
    
You can see now how a Launcher instance is able to determine what Test Engine to delegate tests to. It passes a LauncherDiscoveryRequest instance to each
registered Test Engine (found dynamically via Java's [ServiceLoader](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html)
mechanism) and lets the engine return a Test Plan indicating what tests it can discover and execute. 
As you can see below, their are multiple TestEngine implementations within
the Junit5 code base itself.

```
![TestEngine](http://clarityviews.com/embed/junit-team/junit5/master/diagram/junit5-master/junit-platform-engine/src/main/java/org/junit/platform/engine/TestEngine.java)
```

![TestEngineDiagram](http://clarityviews.com/embed/junit-team/junit5/master/diagram/junit5-master/junit-platform-engine/src/main/java/org/junit/platform/engine/TestEngine.java)

## A Concrete Example, The Jupiter Test Engine
If you've come this far, you've probably realized why Junit5's architecture makes developing extensions for Junit much easier.
Junit5 now supports a model where multiple Test Engines (not just Junit5), each with thier own implementations of
executing tests exist under a common Laucher. We mentioned earlier that Junit Platform Launcher is responsible for 
providing  the API's external tools will use. Therefore, all the Test Engine implementations found at runtime discovered by the
Junit Platform Launcher ca 
be accessed using a common set of public APIs, great stuff. As in the diagram below, Junit5 provides its own TestEngine implementation, called Jupiter,
which as you would expect  is responsible 
for discovering and executing tests written using Junit 's brand new APIs. The Vintage Test Engine was another Test Engine implementation
created in Junit5 to executing tests written in Junit 4 and Junit 3 by the Junit Platform.

```
![Clarity Views Diagram](http://clarityviews.com/embed/junit-team/junit5/master/diagram/junit5-master/junit-jupiter-engine/src/main/java/org/junit/jupiter/engine/JupiterTestEngine.java)
```

![JupiterDiagram](http://clarityviews.com/embed/junit-team/junit5/master/diagram/junit5-master/junit-jupiter-engine/src/main/java/org/junit/jupiter/engine/JupiterTestEngine.java)

We hoped you enjoyed this article and as always, all diagrams have been generated using [Clarity Views](http://clarityviews.com)!
And if you would like, feel free to explore Junit5 for yourself! [![Clarity Views Label](http://clarityviews.com/badge)](http://clarityviews.com/github/junit-team/junit5)
