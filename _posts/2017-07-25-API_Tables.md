--- 
title:  "API Documentation Using Tabular Expressions"
date:   2017-07-20 15:04:23
categories: [api, REST, tabular, documentation]
tags: [api, REST, tabular, documentation]
excerpt_separator: <!--more-->
---
It is wellknown that APIs need developer-friendly docs in order
to gain widespread adoption. However, little if any improvement in the way REST APIs are presented to developers have been made over the 
past few years. Nowadays, most API docs and developer portals adopt a [Swagger UI](http://petstore.swagger.io/) type of format where all the endpoints in the specification
are simply listed, one after another. This format does a good job of communicating the low level details of each endpoint; However, 
as the complexity of APIs grows, developers need a way to get a high level understanding of the API **before** diving down into those
low level details.
<!--more--> 

![swagger example](/images/petstorev2.png)

The image above depicts Swagger UI documentation for a sample pet resource. As an API specification grows in complexity,
I find it increasingly difficult to answer the following questions using the aformentioned format:

**1. What are all the resources that this API exposes?**

**2. What operations are available on key resources?**

Its easy to see why these problems exist. Even with tags, an API specification with many endpoints will take a long time to
navigate. As a result, determining basic information about an API such as what resources are available requires
developers to sift through a longer than necessary lists of API endpoints.

### Introducing API Tables

API Tables form the perfect solution for providing a quick overview of a group of related endpoints in a specification. The image below represents
an API Table for working with Pull Requests in the [GitHub API](https://developer.github.com/v3/). The table is
read left to right, where each cell builds on the cells to its left to represent a certain endpoint of the API
spec. Each cell also contains buttons representing operations available at that particular endpoint (GET, POST, etc..) 
that users can click on to view more detailed documentation. Download the interactive version of this API Table in
pdf format [here](https://github.com/Zir0-93/zir0-93.github.io/raw/master/images/tabular_github_apiv3.pdf)

![tabexpr](/images/tabexprv7.svg)

A quick glance at the table above conveys what resources are made available by this API along with
what operations can be executed on those resources.
Believe it or not, the API Table above compactly depicts over 30 different API operations whose documentation is spread across six different
and lengthy pages 
on GitHub. And this is precisely why API tables shine, **they provide a high level representation of an API spec that allows
developers to find and navigate to more detailed documentation in a natural and intuitive way**. Additionally, these
tables are scalable and can be used to represent larger and more sophisticated API specs without sacrificing readability.
The sample table depicted above can easily accomodate up to 65 different operations!

I am not sure about you

#### Its Time For Change
