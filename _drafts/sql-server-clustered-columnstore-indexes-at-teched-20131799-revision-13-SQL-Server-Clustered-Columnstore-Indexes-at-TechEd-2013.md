---
id: 1813
title: SQL Server Clustered Columnstore Indexes at TechEd 2013
date: 2013-06-11T03:24:54+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/06/11/1799-revision-13/
permalink: /2013/06/11/1799-revision-13/
---
Now that the TechEd 2013 presentations are [up online on Channel9](http://channel9.msdn.com/Events/TechEd/NorthAmerica/2013) you can checkout for yourself Brian Mitchell&#8217;s presentation [What&#8217;s New for Columnstore Indexes and Batch Mode Processing](http://channel9.msdn.com/Events/TechEd/NorthAmerica/2013/DBI-B322). Brian does a great job at presenting the the new updatable clustered columnstore indexes and the enhancements done to the vectorized query execution (aka. batch mode).

I would also like to talk a little bit about how the new updatable clustered columnstore index work. The best resource for your education on the topic is the SIGMOD 2013 paper [Enhancements to SQL Server Column Stores](http://research.microsoft.com/pubs/193599/Apollo3%20-%20Sigmod%202013%20-%20final.pdf). This paper cites several of the improvements that are available in clustered columnstores, besides the obvious updatability: 

  * Improved Index Build
  * Sampling Support
  * Bookmark Support
  * Schema modification support
  * Support for short strings
  * Mixed Execution Mode
  * Hash Join support
  * Improvements in Bitmap filters
  * Archival support

It should be no surprise to anyone studying columnar storage that the updatable clustered columnstores coming with the next version of SQL Server are based on deltastores. I talked before about the [SQL Server 2012 Columnstore internals](http://rusanu.com/2012/05/29/inside-the-sql-server-2012-columnstore-indexes/) and I explained why the highly compressed format that makes columnar storage so fast is also make it basically impossible to update in-place. The technique of having a &#8216;deltastore&#8217; which stores updates and merge it just-in-time with the columnar data during scans is not new and is employed by several of the columnar storage solution vendors.

<!-- more -->

# Deltastores

<p class="callout float-right">
  Delstastores are ordinary B-Trees that store uncompressed row groups of the clustered columnstore
</p>

Columnstores introduce a new unit of organization, a row-group. A row-group is a logical unit that groups up to 2<sup>20</sup> rows (about 1 million rows). In SQL Server 2012 the row-groups where implicit and there was catalog view to show them. As you can see in Brian&#8217;s presentation SQL Server 14 adds a new catalog view: <tt>sys.column_store_row_groups</tt>. This catalog view show the state of each row group for all columnstores (including non-clustred ones). Updatable clustred columnstores can show the row groups in COMPRESSED state or in OPEN/CLOSED state. The OPEN/CLOSED row groups are deltastores (yes, there could be multiple deltastores per columnstore, see the above mentioned SIGMOD 2013 paper). OPEN row groups are ready to accept more inserts while CLOSED row groups have filled up and are awaiting compression.