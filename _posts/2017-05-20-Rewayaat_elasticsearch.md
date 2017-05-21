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

 ![esyudothis.jpg](/images/initializeshards.png)
 
## Dealing With Human Languages (Fuzzy, Arabic)

Full-text search is a battle between precision — returning as few irrelevant documents as possible, 
and recall—returning as many relevant documents as possible. This battle is made more challenging when searching 
hadith since as documents were originally recorded in Arabic and later translated for English speakers. 
This poses a problem for recall when there are multiple translations of the same word -
a quick and dirty example can be seen in the word ```مسلم```. Valid English translations of this word
include  ```a Muslim``` and ```one who submits```; However, given that a translation must use **one** of several 
valid meanings for a word, a user searching for alternative meanings of the word will completely miss out on 
relevant documents!

![precisionrecall](/images/precisionrecall.png)

Spelling is another crucial area that affects recall, The name Muhammad for example, may take over eight almost identical English 
spellings across all the books of hadith. How can we easily return all the relevant documents based on a search 
for one of these valid spellings? We also need to consider the use of titles in Islamic History. An influential 
figure like Ali Ibn Abi Talib for example, may also be referred to as Abu Turab (Father of Soil) ,
Amir Al-Momineen (Commander of the Faithful) amongst many other titles. As you can see, dealing with language 
in a full text search engine is a challenging task. 

#### Synonym Filters

Synonyms help to broaden the scope of the user's search by relating concepts and ideas. As mentioned earlier,there maybe
multiple English translations of a given Arabic word, which poses a serious problem for returning as many relevant
documents as possible. We can use Elastic Search synonym filters to make a word more generic, allowing us to account
for the variation that exists in translated Arabic words. For example, a user searching for ```Lord```, represented by the
arabic word ```رب``` will automatically get results for 
Documents containing its other valid translations: ```Sustainer, Cherisher, Master, Nourisher and God```! Instructions
for setting up synonyms can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-tokenfilter.html).

#### Word Stemming

Most languages of the world are inflected, meaning that words can change their form to express differences in number,
tense, gender, person, cause and mood.

![stem](/images/stem.svg)

While inflection aids expressivity, it interferes with retrievability, as a single root word sense (or meaning)
may be represented by many different sequences of letters. 
Stemming attempts to help improve recall by removing the differences between inflected forms of a word, in order to 
reduce each word to its root 
form. 


#### Fuzzy Searching
Fuzzy matching treats two words that are “fuzzily” similar as if they were the same word. 
First, we need to define what we mean by fuzziness. In 1965, Vladimir Levenshtein developed the
Levenshtein distance, which measures the number of single-character edits required to transform one
word into the other. Elasticsearch supports a maximum edit distance, specified with the fuzziness parameter, of 2.


 ## Simplified User Search Experience
 
 
 
 
 
 
 

  
  
  
  
  
  
  ## Highlighting
