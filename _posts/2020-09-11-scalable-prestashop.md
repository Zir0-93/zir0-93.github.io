--- 
title:  "Deploy A Production-Ready E-Commerce Solution on AWS with CloudFormation"
date:  2020-09-11 15:04:23
og_image: https://raw.githubusercontent.com/hadii-tech/production-ready-prestashop/master/resources/scalable_presta.png
redirect_to: "https://github.com/hadii-tech/production-ready-prestashop/issues"
tags: [aws, ecommerce, cloudformation, scalability]
description: "Shopify works until it doesn't — vendor lock-in and transaction fees compound at scale, and the platform gives you limited control over infrastructure when it matters. This post walks through a CloudFormation-based reference architecture for running PrestaShop on AWS with the operational properties of a production system: auto-scaling application tier, managed RDS database, ElastiCache for session and object caching, and a CDN layer for static assets. Every resource is defined as code, so the entire stack can be reproduced across environments with a single command. The architecture was developed following a real client engagement where the cost and control tradeoffs of hosted e-commerce platforms became untenable at scale."
excerpt_separator: <!--more-->
---

E-commerce platforms like Shopify have dominated the e-commerce industry for many years due to their low upfront setup costs and ease of use. 
However, after a recent engagement with a large online merchant, our team found two significant reasons as to why a commercial solution such as Shopify may not always make sense for an organisation: 
<!--more-->
