---
id: 1499
title: Inside the SQL Server 2012 Columnstore Indexes
date: 2012-05-17T03:16:26+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/17/1463-revision-33/
permalink: /2012/05/17/1463-revision-33/
---
Columnar storage has established itself as the de-facto option for Business Intelligence (BI) storage. The traditional row-oriented storage of RDBMS was designed for fast single-row oriented OLTP workloads and it has problems handling the large volume range oriented analytical processing that characterizes BI workloads. But what _is_ columnar storage and, more specifically, how does SQL Server 2012 implement columnar storage with the new <tt>COLUMNSTORE</tt> indexes?

<!--more-->

## Column Oriented Storage

The defining characteristic of columnar storage is the ability to read the values of a particular column of a table without having to read the values of all the other columns. In row-oriented storage this is impossible because the individual column values are physically stored grouped in rows on the pages and reading a page in order to read a column value _must_ fetch the entire page in memory, thus automatically reading all the other columns in the row:

[<img src="http://rusanu.com/wp-content/uploads/2012/05/row-oriented-reads.png" alt="" title="row-oriented-reads" width="600" class="aligncenter size-full wp-image-1465" />](http://rusanu.com/wp-content/uploads/2012/05/row-oriented-reads.png)

This image shows how in row oriented storage a query that only needs to read the values of column <tt>A</tt> needs to pay the penalty of reading the entire pages, including the unnecessary columns B, C, D and E, simply because the physical format of the record and page. If you&#8217;re not familiar with the row format I strongly recommend Paul Randal&#8217;s article<a href="http://www.sqlskills.com/blogs/paul/post/Inside-the-Storage-Engine-Anatomy-of-a-record.aspx" target="_blank">Inside the Storage Engine: Anatomy of a Record</a>.

By contrast the column oriented storage stores the data in a format that groups columns together on disk, so reading values from a single column incurs only the minimal IO needed to fetch in the column required by the query:

[<img src="http://rusanu.com/wp-content/uploads/2012/05/column-oriented-reads.png" alt="" title="column-oriented-reads" width="447" height="454" class="aligncenter size-full wp-image-1467" />](http://rusanu.com/wp-content/uploads/2012/05/column-oriented-reads.png)

By physically separating the values from individual columns into their own pages the engine can read only the columns needed. This reduces the IO for queries that:

  * read only a small subset of columns from all the columns of the table (no <tt>SELECT *</tt>)
  * read many rows (scans and range scans)

Not surprisingly this type of queries is the typical BI query pattern. Note that a column qualifies as &#8216;read&#8217; not only if is projected in the result set, but also if is referenced in the <tt>WHERE</tt> clause, or in a join, or anywhere else in the query. Typical BI queries aggregates fact values over large ranges defined by some dimension subsets (eg. &#8220;sum of sales in the NW region in Sep.- Nov. period&#8221;) and thus they tend to create exactly the queries that the columnar storage loves.

<p class="callout float-right">
  Columnstore read from disk only the columns referenced by the query
</p>

The fact that BI workloads use a pattern of queries that references only few columns but many rows, combined with the ability of the columnar storage to read from disk only the columns actually referenced by the query is the first and foremost advantage of the column store technology.

An OLTP workload by contrast tends to read entire rows (all columns) and only one or very few rows at a time. Columnar storage looses its appeal in OLTP workloads and in fact the columnar storage can be significantly slower than row oriented storage for OLTP workloads.

## Compression

Because column oriented storage groups values from the same column together there is a new beneficial side effect: data becomes more compressible. Compression can be deployed on row-oriented storage too, see <a href="http://msdn.microsoft.com/en-us/library/cc280464.aspx" target="_blank">Page Compression Implementation</a>, but with column oriented storage the data is more homogenous, as it contains only values from a single column. Since compression ratio is subject to the data entropy (how homogenous the data is) it follows that columnar storage format is more compressible than the same data represented in row oriented storage format. Furthermore, since the columnstore format is targeted explicitly at BI workloads and large datasets, it is justifiable to invest significantly more development effort into enhancing compression benefits: deploy several alternative compression algorithms for the engine to choose from, leverage when possible operations directly in the compressed format _w/o_ decompressing the data first. Row oriented storage always have to strike a balance between compression benefits and its runtime cost, because it has to consider the typical OLTP workload.

<p class="callout float-left">
  Columnstores target large read-only datasets for which compression yields most benefits
</p>

Columnstores embrace compression fully and go to great lengths into achieving high compression ratio because in the expected BI workload the savings in IO from better compression more than offset the runtime CPU loss. The individual compression techniques vary between various columnar storage implementation, but your can make a safe bet that some common techniques are used by most implementations:

  * <a href="http://en.wikipedia.org/wiki/Dictionary_coder" target="_blank">Dictionary Encoding</a>
  * <a href="http://en.wikipedia.org/wiki/Huffman_coding" target="_blank">Huffman Encoding</a>
  * <a href="http://en.wikipedia.org/wiki/Run-length_encoding" target="_blank">Run Length Encoding</a>
  * <a href="http://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch" target="_blank">Lempel-Ziv-Welch</a>

In addition domain specific compression is used, leveraging the knowledge of the types and values being stored. For example <a href="http://en.wikipedia.org/wiki/Column-oriented_DBMS#Compression" target="_blank">Wikipedia</a> mentions the benefits of sorting the data in order to yield better compression:

> To improve compression, sorting rows can also help. For example, using bitmap indexes, sorting can improve compression by an order of magnitude.[6] To maximize the compression benefits of the lexicographical order with respect to run-length encoding, it is best to use low-cardinality columns as the first sort keys.

Note that I am not saying to apply the advice from above mentioned Wikipedia article directly to SQL Server 2012. Specifically columnstore indexes do not require to &#8216;use low-cardinality keys first&#8217;, the order in which you specify keys in SQL Server 2012 columnstore indexes is irrelevant.

## Batch Mode Processing

Reading only the data for the columns referenced by the query and extensive use of compression are techniques used by all columnar storage engines. But SQL Server 2012 also brings something new: a completely new query execution engine optimized for BI queries. To understand why this was necessary and why it has a big performance impact one needs to understand first how query processing works on SQL Server. Paul White has a good series of introductory articles into query processing starting with <a href="http://sqlblog.com/blogs/paul_white/archive/2012/04/28/query-optimizer-deep-dive-part-1.aspx" target="_blank">Query Optimizer Deep Dive &#8211; Part 1</a> and I recommend these articles if you want a deeper introduction into SQL Server query processing. For our purposes it suffices to understand that a query execution consists of iterating over a tree of operators. Query execution is basically a loop that takes the query execution tree top operator and calls a method called <tt>GetNextRow</tt> until the methods returns end-of-file. This top operator in turn will have child operators on which it calls <tt>GetNextRow</tt> until to produce the data, and these child operators have further child operators and so on and so forth, all the way to the bottom of the tree operators that implement actual physical access to the data. Whether is a simple scan, a seek, a nested loop join, a hash join, a sort, a where filter, _any_ operator behaves the same. Which also implies that this <tt>GetNextRow</tt> call is a <a href="http://en.wikipedia.org/wiki/Virtual_method_table" traget="_blank">virtual function call</a>. Sometimes these execution trees can be hundreds of operators deep, which means that there will be hundreds of virtual function calls in order to produce one row, followed by again hundreds of calls to produce the next row, then again for the next row and so on. For the analytical workload of OLTP this is not a big deal, since queries are supposed to produce only a few rows and iterate only over small range of the data (a direct seek or a small range scan). But analytical BI queries, even if they produce a small result set containing aggregates, need to scan extremely large sets of data and the cost of traversing deep execution sub-trees to retrieve rows one by one quickly adds up, specially when the calls are all going through a v-table indirection.

<p class="callout float-right">
  Batch mode execution amortizes the cost of calling the operators by requesting many rows in a single call
</p>

Enter the new batch mode operators. In batch mode instead of calling a function that returns one row at a time, the query operators call a function that returns many rows at a time: a batch. The benefits of doing so are not obvious for non-programmers, but believe you me, this can result in speed improvements of 2 and even 3 orders of magnitude.

But batch mode execution requires everybody to play the same tune: all operators have to implement the batch mode execution. Row mode operators cannot interact with batch mode operators nor the other way around. For best performance a query plan has to run in batch mode from top to bottom. Since not all operations were implemented in batch mode by the time SQL Server 2012 RTM shipped, sometimes there will be restrictions that prevent batch mode execution. The Query Optimizer can resort to a conversion operator from batch mode to row mode and it can create plans that have sub trees in batch mode even if the upper part of the tree is in row mode. This mixed mode can still yield significant performance improvements, provided that most data processing still occurs in batch mode (eg. aggregation occurs in batch mode and the result of aggregation is consumed in row mode). Eric Hanson has some very useful tricks for how to achieve batch mode processing in some edgier conditions at <a href="http://social.technet.microsoft.com/wiki/contents/articles/4995.sql-server-columnstore-performance-tuning.aspx#Ensuring_use_of_the_Fast_Batch_Mode_of_Query_Execution" target="_blank">Ensuring Use of the Fast Batch Mode of Query Execution</a>.

## SQL Server 2012 Columnar Storage</p>