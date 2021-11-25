---
id: 1805
title: SQL Server Clustered Columnstore Indexes at TechEd 2013
date: 2013-06-10T07:30:45+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/06/10/1799-revision-6/
permalink: /2013/06/10/1799-revision-6/
---
Now that the TechEd 2013 presentations are [up online on Channel9](http://channel9.msdn.com/Events/TechEd/NorthAmerica/2013) you can checkout for yourself Brian Mitchell&#8217;s presentation [What&#8217;s New for Columnstore Indexes and Batch Mode Processing](http://channel9.msdn.com/Events/TechEd/NorthAmerica/2013/DBI-B322). Brian does a great job at presenting the the new updatable clustered columnstore indexes and the enhancements done to the vectorized query execution (aka. batch mode).

I would also like to talk a little bit about how the new updatable clustered columnstore index work. The best resource for your education on the topic is the SIGMOD 2013 paper [Enhancements to SQL Server Column Stores](http://research.microsoft.com/pubs/193599/Apollo3%20-%20Sigmod%202013%20-%20final.pdf). It should be no surprise to anyone studying columnar storage that the updatable clustered columnstores coming with the next version of SQL Server are based on deltastores. I talked before about the [SQL Server 2012 Columnstore internals](http://rusanu.com/2012/05/29/inside-the-sql-server-2012-columnstore-indexes/) and I explained why the highly compressed format that makes columnar storage so fast is also make it basically impossible to update in-place. The technique of having a &#8216;deltastore&#8217; which stores updates and merge it just-in-time with the columnar data during scans is not new and is employed by several of the columnar storage solution vendors.

<!-- more -->