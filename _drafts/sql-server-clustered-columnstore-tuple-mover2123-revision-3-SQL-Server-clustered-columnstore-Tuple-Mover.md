---
id: 2126
title: SQL Server clustered columnstore Tuple Mover
date: 2013-12-02T02:39:21+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/12/02/2123-revision-3/
permalink: /2013/12/02/2123-revision-3/
---
The [updateable clustered columnstore indexes introduced with SQL Server 2014](http://rusanu.com/2013/06/11/sql-server-clustered-columnstore-indexes-at-teched-2013/) rely on a background task called the <tt>Tuple Mover</tt> to periodically compress deltastores into the more efficient columnar format. Deltastores store data in the traditional row-mode (they are B-Trees) and as such are significantly more expensive to query that the compressed columnar segments. How more expensive? They are equivalent to storing the data in an uncompressed Heap and, due to small size (max 1048576 rows per deltastore rowset), they get little traction from parallelism and from read-aheads. It is important for your upload, initial seed and day-to-day ETL activities to achieve a healthy columnstore index, meaning no deltastores or only a few deltastores. To achieve this desired state of a healthy columnstore it is of paramount importance to understand how deltastores are created and removed.

<!--more-->

## How are Deltastores created

Deltastores appear through one of the following events:

  * Trickle <tt>INSERT</tt>: ordinary INSERT statements that do not use the BULK INSERT API. That includes all <tt>INSERT</tt> statements except <tt>INSERT ... SELECT ...</tt>.
  * <tt>UPDATE</tt>, <tt>MERGE</tt> statements: all updates are implemented in clustered columnstores as a delete of old data and insert of new (modified) data. The insert of new data as result of data updates will always be a trickle insert.
  * **Undersized** BULK INSERT operations: insufficient number of rows of data inserted using the BULK INSERT API (that is using one of [<tt>IRowsetFastLoad</tt>](http://technet.microsoft.com/en-us/library/ms131708.aspx), use the [SQL Native client ODBC Bulk operation extensions](http://msdn.microsoft.com/en-us/library/ms130792.aspx) or the .Net managed <tt>SqlBulkCopy</tt> class) and <tt>INSERT ... SELECT ...</tt> statements</li> 
    
    . These BULK INSERT APIs are often better known by the various tools that expose them, like <tt>bcp.exe</tt> or the &#8216;fast-load&#8217; SSIS OleDB destination. _Insufficient_ rows means that the number of rows submitted **in a batch** is too small to create a compressed segment **in a partition**. The exact number is, I think, around 100000 rows. Bellow this margin, even when BULK INSERT API is used, the rows will be inserted as a deltastore. Above 100000 a compressed segment is created, but do note that a clustered columnstore consisting of 100k rows segments will be sub-optimal. The ideal batch size is 1 million rows, and your ETL process should thrive to achieve this batch size **per partition per thread** (ie each inserting thread should insert 1 million rows for each partition it inserts into). </li> </ul>