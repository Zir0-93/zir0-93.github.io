--- 
title:  "Learning Exceptional Code Using Neural Networks"
image: /images/ml.jpg
date:   2018-02-10 15:04:23
categories: [machine-learning]
tags: [machine learning, skipgram, graphs, tensorflow]
excerpt_separator: <!--more-->
---
I'm afraid this article won't be about using machine learning to generate unsually good, outstanding code to solve the world's problems.
I think we can all agree there simply isn't enough training data in existence for that. Rather, I'll try to use machine learning on graphs
in order to detect common exception handling anti-patterns [Exception/Error handling](https://en.wikipedia.org/wiki/Exception_handling) in source code.
<!--more-->
![Exception_img](/images/exceptions.jpg)

One of the main reasons as to why exception handling can be tricky to do properly is a result of the almost "hidden" control-flow
possibilities it introduces at essentially every line of code.
Such a hidden control transfer possibility is all too easy for programmers to overlook, resulting in exceptions that can 
cause a program to become corrupt and/or difficult to predict. Of 88 000 [review comments](https://developer.github.com/v3/pulls/comments/) mined from the most starred Java repositories on
[GitHub](https://github.com), approximately 13 000 (15%) were related to exception handling! In this experiment, I will use machine
learning methods to try and automatically predict whether a given snippet of code exhibits proper exception handling strategies or not.
All the code is available as a [python notebook](), and all experimental results were executed a linux machine with 32 GB RAM and a Tesla K40c GPU.  

    ![sample GitHub review comment](/images/review_comment.png)

    Sample GitHub review comment**

# Step 1 - Build a Review Comment Classifier 

# Step 2 - Convert Code Snippets to AST's

# Step 3 - Generate AST Embeddings Using Skip-Gram

# Step 4 - Train Neural Network

# Results and Conclusion
