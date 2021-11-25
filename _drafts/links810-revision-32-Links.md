---
id: 853
title: Links
date: 2010-07-11T20:12:39+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/07/11/810-revision-32/
permalink: /2010/07/11/810-revision-32/
---
 

## Future of RDBMS

  * [The End of an Architectural Era (It’s Time for a Complete Rewrite)](http://vldb.org/conf/2007/papers/industrial/p1150-stonebraker.pdf)
  * [Life beyond Distributed Transactions: an Apostate’s Opinion](http://www-db.cs.wisc.edu/cidr/cidr2007/papers/cidr07p15.pdf)
  * [HStore: A HighPerformance, Distributed Main Memory Transaction Processing System](http://cs-www.cs.yale.edu/homes/dna/papers/hstore-demo.pdf)
  * [Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)</ul> 

## SQL Server Links

  * [To BLOB or not to BLOB](http://research.microsoft.com/pubs/64525/tr-2006-45.pdf)
  * [Previously committed rows might be missed if NOLOCK hint is used](http://blogs.msdn.com/b/sqlcat/archive/2007/02/01/previously-committed-rows-might-be-missed-if-nolock-hint-is-used.aspx)
  * [Developing Large Scale Web Applications and Services](http://mschnlnine.vo.llnwd.net/d1/pdc08/WMV-HQ/BB07.wmv)
  * [MySpace DB Overview](http://www.sdsqlug.org/presentations/May2009/MySpace_DB_Overview.pptx)
  * [Real Time Analytics with SQL Server 2008 R2 StreamInsight](http://channel9.msdn.com/learn/courses/SQL2008R2TrainingKit/SQL10R2UPD00/SQL10R2UPD00_REC_03/)
  * [March Madness on Demand](http://blogs.msdn.com/rdoherty/archive/2009/03/13/march-madness-on-demand.aspx)
## RDBMS Historic Links

<a name="SystemR" href="http://www.seas.upenn.edu/~zives/cis650/papers/System-R.PDF">System R: Relational Approach to Database Management</a>
:   System R is the grand father of all relational databases today. This is the original paper that introduced System R to the world, back in 1976. All of today&#8217;s RDBMSs inherit from Sytem R the most fundamental traits:</p> 
    
      * Disk oriented storage
      * Indexing structures
      * Multithreading to amortize latency
      * Lock based concurency
      * Log based recovery

Write-Ahead Logging
:   Write-Ahead Logging is the cornerstone of recoverability in most RDBMS sytems. If you ever wanted to understand how a database log works, this is the paper to read: <a name="Aries" href="http://www.cs.berkeley.edu/~brewer/cs262/Aries.pdf">ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging</a>

The Five Minute Rule
:   Ever wondered where does the 300 seconds bufer pool page life expectancy come for? The original research was done back in 1986 and simple economics were involved: given the cost of memory vs. the cost of mass storage in 1986 it was best to keep in memory a page if it was being accessed again in less than 5 minutes. The paper was revisited again in 1996 and 2006 and the 5 minute point was still the point of equilibrium:</p> 
    
      * [The Five-Minute Rule for Trading Memory for Disk Accesses](http://www.hpl.hp.com/techreports/tandem/TR-86.1.pdf)
      * [The Five-Minute Rule Ten Years Later](ftp://ftp.research.microsoft.com/pub/tr/tr-97-33.pdf)
      * [The Five-Minute Rule 20 Years Later](http://cacm.acm.org/magazines/2009/7/32091-the-five-minute-rule-20-years-later/fulltext)

[The 1995 SQL Reunion: People, Projects, and Politics](http://www.mcjones.org/System_R/SQL_Reunion_95/sqlr95.html)
:   This paper is actually a group interview with most of the people involve din the original release of the very first RDBMS.

[AlphaSort: A Cache-Sensitive Parallel External Sort](http://research.microsoft.com/en-us/um/people/gray/alphasort.doc)
: