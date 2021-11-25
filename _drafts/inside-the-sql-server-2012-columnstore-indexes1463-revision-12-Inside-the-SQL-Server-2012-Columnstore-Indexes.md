---
id: 1478
title: Inside the SQL Server 2012 Columnstore Indexes
date: 2012-05-16T06:57:19+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/16/1463-revision-12/
permalink: /2012/05/16/1463-revision-12/
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

> The fact that BI workloads use a pattern of queries that references only few columns but many rows, combined with the ability of the columnar storage to read from disk only the columns actually referenced by the query is the first and foremost advantage of the column store technology.

An OLTP workload by contrast tends to read entire rows (all columns) and only one or very few rows at a time. Columnar storage looses its appeal in OLTP workloads and in fact the columnar storage can be significantly slower than row oriented storage for OLTP workloads.

## Compression

Because column oriented storage groups values from the same column together there is a new beneficial side effect: data becomes more compressible. Compression can be deployed on row-oriented storage too, see <a href="http://msdn.microsoft.com/en-us/library/cc280464.aspx" target="_blank">Page Compression Implementation</a>, but with column oriented storage the data is more homogenous, as it contains only values from a single column. Since compression ratio is subject to the data entropy (how homogenous the data is) it follows that columnar storage format is more compressible than the same data represented in row oriented storage format. Furthermore, since the columnstore format is targeted explicitly at BI workloads and large datasets, it is justifiable to invest significantly more development effort into enhancing compression benefits: deploy several alternative compression algorithms for the engine to choose from, leverage when possible operations directly in the compressed format _w/o_ decompressing the data first. Row oriented storage always have to strike a balance between compression benefits and its runtime cost, because it has to consider the typical OLTP workload.

> Columnstores embrace compression fully and go to great lengths into achieving high compression ratio because in the expected BI workload the savings in IO from better compression more than offset the runtime CPU loss.

## Batch Mode Processing</p>