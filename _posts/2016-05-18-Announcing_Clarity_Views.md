---
title:  "Clarity Views - Public Offering Release"
date:   2016-05-18 15:04:23
categories: [tech, code, static analysis, uml]
tags: [tech, code, static analysis, uml]
---

With [Clarity Views](bttp://clarityviews.com) developers can analyze their open source repositories for both architectural designs and implementation level patterns right from the browser, making any piece of code a breeze to enhance, maintain and test. Simply visit the [Clarity Views](http://clarityviews.com) web page to get started and open the repository of your choice.

<span class="fa fa-question-circle-o"></span>

## You write great code, but...
We get it, you're dev team is filled with rockstar developers. You're all about using the right design patterns, writing effective unit tests, developing maintainable code, writing useful documentation - your GitHub activity says it all. However, despite your efforts to maintain best practices, potential contributors to your project and new team members are often overwhelmed with a parking garage of code when they visit your GitHub repository page.  


![overwhelmed](/images/overwhelmed.gif)


## So we asked...  
How do we reduce the amount of time it takes for someone to understand how a given component of a system works? 

-   How do I develop tests for this component?
-   What design patterns, classes and methods of the system can be re-used/extended?
-   What are the coding styles and patterns of the current codebase?
-   What is the design rationale behinds certain parts of the system?


##  Code Visualizations ![code](/images/diagramviewbtn.PNG)

Tradional UML diagrams recovered from code are too complicated and fail to abstract away trivial implementation details. We believe visualizations of code should simplify its understanding, so we've developed our own algorithm to generate concise, meaningfull diagrams that can be interacted with so you can figure the all-important why behind code. **Look for the ![pluses](/images/diagramviewbtnsmall.png) button** to switch to the diagram view.

![diagram](/images/diagram.PNG) 

## Code Intelligence ![code](/images/codeviewbtn.PNG)
Clarity does not only understand your code, it does more with it than your code host. Users can click on any type within the Code View to *jump to definitions, view type information, and view example code snippets for a given type*. We also prioritizes displaying test related code snippets to quickly convey intended usage scenarios for a given type. **Look for the ![codess](/images/codeviewbtnsmall.png) button** to switch to the Code View.

![diagram](/images/codeview.PNG)

## Code Search
Save time and effort by searching across functions, classes, and other code elements. Simply enter the name of the function of type into the search bar at the top of your Clarity Views environment to start searching.

![diagram](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/search.PNG)

## Create GitHub Issues ![plus](/images/newissuebtn.PNG)
Starting discussions and creating issues in line with your code and diagrams keeps everyone on the same page. Simply *highlight a piece of code* and **look for the ![pluss](/images/newissuebtnsmall.png) button** to create a new contextual issue where developers can ask questions, suggest improvements, and explain design decision. You can embed diagrams into a new GitHub issue from the Diagram View  ![pluses](/images/diagramviewbtnsmall.png) as well.

![diagram](/images/embed.PNG)

## Embed Your Diagrams ![embed](/images/embedbtn.PNG)
Clarity Views also helps help teams write useful, maintainable documentation. Users can *embed diagrams into any HTML or markdown environment*, simply **look for the  ![embeds](/images/embedbtnsmall.png) button** to get the embed code. Furthermore, as your codebase evolves, so will the diagrams!

```
![Clarity Views Diagram](http://clarityviews.com/embed/junit-team/junit5/master/diagram/junit5-master/junit-platform-engine/src/main/java/org/junit/platform/engine/TestDescriptor.java)
````
![Clarity Views Diagram](http://clarityviews.com/embed/junit-team/junit5/master/diagram/junit5-master/junit-platform-engine/src/main/java/org/junit/platform/engine/TestDescriptor.java)
<p class="lead">Best of all, Clarity Views is <b>completely secure</b> - all code is requested on the fly using GitHub API's and never kept in any storage of any kind. Clarity Views is also <b>completely free</b> for open source repositories, get <a href="http://clarityviews.com">started</a> today! </p>

## Display Your Badge!
Click on the Clarity Views badge ![badge](http://clarityviews.com/badge) within any repository page to get your badge embed code!
