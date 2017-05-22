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
 
## Dealing With Human Languages

Full-text search is a battle between precision — returning as few irrelevant documents as possible, 
and recall—returning as many relevant documents as possible. When dealing with documents that contain
searcheable text that in both Arabic and English, this battle becomes even more difficult to deal with.
Outlined below are some of the challenges I faced while developing a search engine and how Elastic Search
helps to deal with them.

![precisionrecall](/images/precisionrecall.png)
 

### Synonym Filters

A user searching for "intelligence" for example, would expect not only to find documents containing
the word "intelligence", but documents containing words that have an almost identical meaning to "intelligence"
as well. These results should have lower higher priority, but nonetheless they should show up. Furthermore, its
not as simple as finding synonyms of a word and searching for those as well, sometimes we need to relate terms that
are not seemingly related. For example, a user searching for "United States" might expect to see results for "USA",
"America" and the "United States of America" as well! This is where Elastic Search synonym filters come into play.


Synonyms help to broaden the scope of the user's search by relating concepts and ideas. We can use Elastic Search synonym filters to make a word more generic, allowing us to account
for the multiple closely related meanings a word might have, in both Arabic and English. For example, I can define
a simple rule like the following:

```
intelligent -> wise, knowledgeable, smart
```

Now, a user searching for ```intelligent``` will also see
Documents containing  "wise", "knowledgeable" and  "smart" as well! Instructions
for setting up synonyms can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-tokenfilter.html).

### Word Stemming

Most languages of the world are inflected, meaning that words can change their form to express differences in number,
tense, gender, person, cause and mood.

![stem](/images/stem2.svg)

While inflection aids expressivity, it interferes with retrievability, as a single root word sense (or meaning)
may be represented by many different sequences of letters. For example, a user searching for "run" would miss
out on relevant documents containing "runner" or "running". Stemming attempts to help improve recall by reducing
each word in your documents to their root form before carrying out the user's queries. This is not always a good thing,
as a user may be soley interested in documents containing the word "run"; therefore, using a [Bool Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)
to combine the results of the user's boosted main query without stemming with the user's query with stemming gives
the best of both worlds. Multiple [Alogirithmic](https://www.elastic.co/guide/en/elasticsearch/guide/current/algorithmic-stemmers.html) and [Dictionary](https://www.elastic.co/guide/en/elasticsearch/guide/current/dictionary-stemmers.html) Stemmers are provided by Elastic Search to use for
queries right out of the box.

### Fuzzy Searching

One major difficulty with dealing with documents containing English translations of Arabic has to do with the various
spellings a single Arabic word or name might take in English. The Arabic name "جعفر" might take English forms of
"Jaffar", "Jafar", "Ja'ffar" and "Ja'far" across the entire search space. How can we easily return all the relevant documents based on a search 
for one of these valid spellings?  Or how do we account for the fact
that users of the engine might use both the American and British spelling of the same word (color vs colour)? We certainly don't want to have to define synonyms in this case, rather we can take advantage of Elastic Search [Fuzzy Queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html).

Fuzzy matching treats two words that are “fuzzily” similar as if they were the same word. 
Here, fuzziness describes the number of single-character edits required to transform one
word into the other. Not only does this type of query allow us to account for minor variations in the spelling of 
translated words, but fuzzy searching allows us to return relevant results for misspeled words and for words
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
```
This would return all documents containing one or more of the terms "Strawberry", "Cuppy" or "Cakes".

Or, more advanced users can take advantage of [Lucene's Query Parser Syntax](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html)
in order to gain deeper insights from the data:

```
(title:"foo bar" AND body:"quick fox") OR title:fox
```
This would search for either the phrase "foo bar" in the title field AND the phrase "quick fox" in the body field,
or the word "fox" in the title field.

## Summary
  
  Elastic Search provides many great tools right out of the box for creating a great full-text search engine. 
  notable features I did not mention above include document highlighting, aggregations, and multi-lingual support for 
  over 20 languages provided right out of the box. 
