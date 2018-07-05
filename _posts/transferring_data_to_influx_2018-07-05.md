--- 
title:  "Analyzing Code Review Comments on GitHub - A Text Classification Experiment"
image: /images/code-reviews.png
date:   2018-05-04 15:04:23
tags: [machine learning, GitHub, code reviews, NLP, n-grams, pyhon, scikit]
excerpt_separator: <!--more-->
---
A code review is a form of code inspection where a developer assesses code for style, defects, and other standards prior to integration into a code base. As part of the code review process on GitHub, developers may leave comments on portions of the unified diff of a GitHub pull request. These comments are extremely valuable in factilitating technical discussion amongst developers, and in allowing developers to get feedback on their code submissions. In an effort to better understand code reivewing habbits, we'er going to create an SVM classifier to classify over 30 000 GitHub review comments based on the main topic
addressed by each comment (e.g. naming, readability, etc.).
<!--more-->

![sample_comment](/images/review_example.png)
