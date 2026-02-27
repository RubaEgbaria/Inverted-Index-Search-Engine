# Inverted-Index-Search-Engine
Big Data - Spark and Information Retrieval

HW-1: Big Data Analysis							

Spark and Information Retrieval
The goal of this assignment is to build a small-scale spark-based search engine which searches in a list of documents to find those answering a user’s query. 
Assume having a list of N documents: doc1, doc2, doc3, …, docN, each of which contains a sequence of tokes (words). This list of files is stored in a single folder.


For this, you need to do the following:

## Inverted Index Construction: 

You have to read the content of the files into a single RDD, and then to do the required mapping and/or other operations so that you end up creating a new file (wholeInvertedIndex.txt) having the following format:


Word, Count(Word), Document_list:

Word: one single word from this collection of files

Count(Word): the number of documents containing Word

Doument_list: a list of documents containing (Word)



### For example, some random rows from the generated inverted index file can be

meet, 2, doc3, doc14

play, 4, doc3, doc6, doc120, doc133

soccer, 3, doc4, doc120, doc133

.
.


### Note the following:

•	The number of rows in this file equals to the number of unique words in the collection. 

•	this step is done once.

•	The words are sorted alphabetically

•	The document list associated with each token is sorted in an ascending order as well.


Query Processing: 
In this step, we need to read wholeInvertedIndex.txt into a new RDD. Then, the application should search this RDD to answer a user’s query. For example, if user searches for “play”, then the system should print: doc3, doc6, doc120, doc133. If user searches for “play soccer”, then the system prints: doc120, doc133.


https://www.youtube.com/watch?v=b2Kcd8ttuq8
