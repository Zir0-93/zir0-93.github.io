---
title:  "Clarity Views - How Developers Understand Code"
date:   2016-05-18 15:04:23
categories: [tech, code, static analysis, uml]
tags: [tech, code, static analysis, uml]
---

<p class="lead"><a href="http://clarityviews.com">Clarity Views</a> is web based software comprehension toolbox built for GitHub <img style="display: inline-block; margin:0;" src="http://clarityviews.com/badge">. It allows developers to rapidly comprehend the inner workings of a software system so they can start contributing faster. Simply visit the <a href="http://clarityviews.com">Clarity Views</a> web page to get started, then display your repository's Clarity Views badge in a visible place for your team to use. </p>

<span class="fa fa-question-circle-o"></span>

## You write great code, but...
We get it, you're dev team is filled with rockstar developers. You're all about using the right design patterns, writing effective unit tests, developing maintainable code, writing useful documentation - your GitHub activity says it all. However, despite your efforts to maintain best practices, potential contributors to your project and new team members are often overwhelmed with a parking garage of code when they visit your GitHub repository page.  


![overwhelmed](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/overwhelmed.gif)

Yet at the same time,  developer rotation throughout a project is great for code quality and promote readability and maintainability of the software. Many open source projects are testament to this, and the folks at [teamed.io](http://www.teamed.io/) are proving that using a distributed network of programmers can help deliver high quality software solutions.

## So we asked... 
How do we reduce the amount of time it takes for someone to understand how a given component of a system works? 

-   What do the types in the system currently do?
-   What design patterns, classes and methods of the system can be re-used/modfied?
-   What are the coding styles and patterns of the current codebase?
-   What is the design rationale behinds certain parts of the system?


##  Code Visualizations
We are big believers in the power of interactive visualizations of source code models in communicating software design. Clarity Views **reverse engineers your code into stunning class diagram representations to allow the user to interact with the various parts of their code in a visual manner**. Of course the "stunning" aspect of these diagrams depend on your OOP skills; however, you're a rockstar developer so no need to worry. Users can interact with these diagrams to jump to definitions, view type information and sample code snippets used within the codebase for a given type.

![diagram](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/diagram.PNG)

## Code Intelligence
Clarity does not only understand your code, it does more with it than your code host. Your code becomes much easier to understand when users can **jump to definitions, view type information, and view example code snippets for a given type - all in your browser**. Clarity also prioritizes displaying test related code snippets to quickly convey intended usage scenarios for a given type.

![diagram](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/code.PNG)

## Code Search
Save time and effort by searching across functions, classes, and other code elements. Simply enter the name of the function of type into the search bar at the top of your Clarity Views environment to start searching.

![diagram](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/search.PNG)

## Code Discussions
Starting discussions and creating issues in line with your code and diagrams keeps everyone on the same page. Simply highlight a piece of code and look for the "plus" sign to create a new contextual issue where developers can ask questions, suggest improvements, and explain design decision. 

![diagram](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/create new issue.PNG)

## Useful Documentation
Clarity Views also helps help teams write useful, maintainable documentation. Users can **embed diagrams into any HTML or markdown environment**, simply look for the "attachment" symbol to get the embed code. Furthermore, as your codebase evolves, so will the diagrams!

```
![Clarity Views Diagram](http://clarityviews.com/github/clarity-team/clarpse/master/diagram/clarpse-master/clarpse/src/main/java/com/clarity/parser/AntlrParser.java)
````
![Clarity Views Diagram](http://clarityviews.com/embed/clarity-team/clarpse/master/diagram/clarpse-master/clarpse/src/main/java/com/clarity/parser/AntlrParser.java)


<p class="lead">Best of all, Clarity Views is <b>completely secure</b> - all code is requested on the fly using GitHub API's and never kept in any storage of any kind. Clarity Views is also <b>completely free</b> for open source repositories, get <a href="http://clarityviews.com">started</a> today! </p>

## Display Your Badge!
Simply click open the Clarity Views environment for your repository and click on the badge to get the embed code.
![diagram](https://raw.githubusercontent.com/clarity-team/jekyll-uno/master/images/Badge.PNG)
