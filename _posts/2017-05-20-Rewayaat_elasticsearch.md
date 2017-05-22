---
title:  "Thoughts on Powering A Hadith Search Engine using Elastic Search"
date:   2017-05-20 15:04:23
categories: [Islam, hadith, narrations, rewayaat, elastic search]
tags: [Islam, hadith, narrations, rewayaat, elastic search]
excerpt_separator: <!--more-->
---
[Elasticsearch](https://www.elastic.co/products/elasticsearch) is an open-source, broadly-distributable, readily-scalable, enterprise-grade search engine. Accessible through an extensive
and elaborate API, Elasticsearch can power extremely fast searches that support your data discovery applications. 
At [Rewayaat.info](http://rewayaat.info/) I worked on implementing a [Hadith](https://en.wikipedia.org/wiki/Hadith)
Search Engine using Elastic Search. I found that for certain use cases, rolling Elastic Search for your back end offers significant advantages over conventional SQL database systems.
<!--more--> 

 ![esyudothis.jpg](/images/initializeshards.png)
 
## Dealing With Human Languages (Fuzzy, Arabic)

Full-text search is a battle between precision — returning as few irrelevant documents as possible, 
and recall—returning as many relevant documents as possible. This battle is made more challenging when searching 
hadith since as documents were originally recorded in Arabic and later translated for English speakers. 
This poses a problem for recall when there are multiple translations of the same word -
a quick and dirty example can be seen in the word ```مسلم```. Valid English translations of this word
include  ```a Muslim``` and ```one who submits```; However, given that a translation must use **one** of several 
valid meanings for a word, a user searching for one translation of the word (eg:  ) will completely miss out on 
relevant documents containing the other valid translations!

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

![stem](/images/stem2.svg)

While inflection aids expressivity, it interferes with retrievability, as a single root word sense (or meaning)
may be represented by many different sequences of letters. 
Stemming attempts to help improve recall by removing the differences between inflected forms of a word, in order to 
reduce each word to its root form. 


#### Fuzzy Searching
Fuzzy matching treats two words that are “fuzzily” similar as if they were the same word. 
Here, fuzziness describes the number of single-character edits required to transform one
word into the other. Not only does this allow us to account for minor variations in the spelling of 
translated words, but fuzzy searching allows us to return relevant results for misspeleed words and for words
that contain typos as well!


 ## Simplified User Search Experience
 
 Elasticsearch is a document oriented database and does not require you to specify a schema upfront. 
 Throw a JSON-document at it, and it will do some educated guessing to infer its type. This saves a great deal amount 
 of time in data retrieval, and Elastic Search will also take care of storing your documents such that
there optimized for search and retrieval. You can now take advantage of many different queries Elastic Search
provides out of the box for analyzing and querying your data. Because I was designing a search engine open to the public,
I wanted to ensure that both technical oriented and non-technical oriented users could get the most out of the search.
The Query String Query is the perfect query provided by Elastic Search to accomplish. Its allows users to search the database
with a simple list of space separated terms:

```
Strawberry Cuppy Cakes

// This would return all documents containing one or more of the terms "Strawberry", "Cuppy" or "Cakes".
```

Or, more advanced users can take advantage of [Lucene's Query Parser Syntax](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html)
in order to gain deeper insights from the data:

```
(title:"foo bar" AND body:"quick fox") OR title:fox

// Search for either the phrase "foo bar" in the title field AND the phrase "quick fox" in the body field,
or the word "fox" in the title field.
 
  
  ## Summary
  
  Elastic Search provides many great tools right out of the box for creating a great full-text search engine. Other
  notable features I did not mention above include document highlighting, aggregations, and multi-lingual support for 
  over 20 languages provided right out of the box.
