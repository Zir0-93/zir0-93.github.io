---
title:  "Thoughts on Powering A Hadith Search Engine using Elastic Search"
date:   2017-05-20 15:04:23
categories: [Islam, hadith, narrations, rewayaat, elastic search]
tags: [Islam, hadith, narrations, rewayaat, elastic search]
excerpt_separator: <!--more-->
---
[Elasticsearch](https://www.elastic.co/products/elasticsearch) is an open-source, broadly-distributable, readily-scalable, enterprise-grade search engine. Accessible through an extensive
and elaborate API, Elasticsearch can power extremely fast searches that support your data discovery applications. At [Rewayaat.info](http://rewayaat.info/) I worked on implementing a [Hadith](https://en.wikipedia.org/wiki/Hadith) Search Engine using Elastic Search. I found that for certain use cases, rolling Elastic Search for your back end offers significant advantages over conventional SQL database systems.
<!--more-->

 ![esyudothis.jpg](/images/CGsJWhyVAAEoqFz.jpg)
 
## Dealing With Human Languages (Fuzzy, Stem, Arabic)

Full-text search is a battle between precision — returning as few irrelevant documents as possible, and recall—returning as many relevant documents as possible. This battle is made more challenging when searching hadith since as documents were originally recorded in Arabic and later translated for English speakers. This poses a problem for both precision and recall when there are multiple translations of the same word - a quick and dirty example can be seen in the word ```مسلم```. Valid English translations of this word include  ```a Muslim``` and ```one who submits```; However, given that a translation must use **one** of several valid meanings for a word, a user searching for alternative meanings of the word will completely miss out on relevant documents!

![precisionrecall](/images/precisionrecall.png)

Spelling is another crucial area, The name Muhammad for example, may take over eight almost identical English spellings across all the books of hadith. How can we easily return all the relevant documents based on a search for one of these valid spellings? We also need to consider the use of titles in Islamic History. An influential figure like Ali Ibn Abi Talib for example, may also be referred to as Abu Turab (Father of Soil) , Amir Al-Momineen (Commander of the Faithful) amongst many other titles. As you can see, dealing with language in a full text search engine is a challenging task. 

### 




 ## Simplified User Search Experience
 
 
 
 
 
 
 

  
  
  
  
  
  
  ## Highlighting
