---
id: 1469
title: Inside the SQL Server 2012 Columnstore Indexes
date: 2012-05-16T04:53:44+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/16/1463-revision-4/
permalink: /2012/05/16/1463-revision-4/
---
Columnar storage has established itself as the de-facto option for Business Intelligence (BI) storage. The traditional row-oriented storage of RDBMS was designed for fast single-row oriented OLTP workloads and it has problems handling the large volume range oriented analytical processing that characterizes BI workloads. But what _is_ columnar storage and, more specifically, how does SQL Server 2012 implement columnar storage with the new <tt>COLUMNSTORE</tt> indexes?

<!--more-->

The defining characteristic of columnar storage is the ability to read the values of a particular column of a table without having to read the values of all the other columns. In row-oriented storage this is impossible because the individual column values are physically stored grouped in rows on the pages and reading a page in order to read a column value _must_ fetch the entire page in memory, thus automatically reading all the other columns in the row:

[<img src="http://rusanu.com/wp-content/uploads/2012/05/row-oriented-reads.png" alt="" title="row-oriented-reads" width="600" class="aligncenter size-full wp-image-1465" />](http://rusanu.com/wp-content/uploads/2012/05/row-oriented-reads.png)

This image shows how in row oriented storage a query that only needs to read the values of column <tt>A</tt> needs to pay the penalty of reading the entire pages, including the unnecessary columns B, C, D and E, simply because the physical format of the record and page. If you&#8217;re not familiar with the row format I strongly recommend Paul Randal&#8217;s article<a href="http://www.sqlskills.com/blogs/paul/post/Inside-the-Storage-Engine-Anatomy-of-a-record.aspx" target="_blank">Inside the Storage Engine: Anatomy of a Record</a>.

By contrast the column oriented storage stores the data in a format that groups columns together on disk, so reading values from a single column incurs no penalties in additional IO:

[<img src="http://rusanu.com/wp-content/uploads/2012/05/column-oriented-reads.png" alt="" title="column-oriented-reads" width="447" height="454" class="aligncenter size-full wp-image-1467" />](http://rusanu.com/wp-content/uploads/2012/05/column-oriented-reads.png)