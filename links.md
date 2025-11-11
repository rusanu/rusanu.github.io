---
id: 810
title: Links
date: 2010-05-15T21:17:10+00:00
author: remus
layout: page
guid: /?page_id=810
---
## SQL Server Links

[To BLOB or not to BLOB](http://research.microsoft.com/pubs/64525/tr-2006-45.pdf)
:   One of the most frequently asked question on discussion forums is wether to store media files in the database as BLOB or to store them on disk and keep just the name of the file in the database. This paper covers the performance characteristics of NTFS vs. storing a BLOB in the database and concludes, perhaps surprisingly that _&#8220;if objects are larger than one megabyte on average, NTFS has a clear advantage over SQL Server. If the objects are under 256 kilobytes, the database has a clear advantage&#8221;_

[Previously committed rows might be missed if NOLOCK hint is used](http://blogs.msdn.com/b/sqlcat/archive/2007/02/01/previously-committed-rows-might-be-missed-if-nolock-hint-is-used.aspx)
:   For all the NOLOCK hint aficionados out there: this is why you never do dirty reads.

[SQL Server I/O Basics](http://technet.microsoft.com/en-us/library/cc966500.aspx)
:   This old SQL Server 2000 article still applies to SQL Server 2005, 2008 and 2008 R2. It explains the inner workings of SQL Server I/O at the most detail level. There is a complementary SQL Server 2005 document [SQL Server I/O Basics, Chapter 2](http://technet.microsoft.com/en-us/library/cc917726.aspx) which goes over SQL Server 2005 specifics. You need to read the SQL Server 2000 article first, as the 2005 one does not go over the same concepts again and assumes you had read the first document.

Troubleshooting SQL Server connectivity
:     * <a href="http://blogs.msdn.com/b/sql_protocols/archive/2008/04/30/steps-to-troubleshoot-connectivity-issues.aspx" rel="nofollow">Steps to troubleshoot SQL connectivity issues</a>
      * <a href="http://blogs.msdn.com/b/sql_protocols/archive/2005/10/22/sql-server-2005-connectivity-issue-troubleshoot-part-i.aspx" rel="nofollow">SQL Server 2005 Connectivity Issue Troubleshoot &#8211; Part I</a> (SNAC)
      * <a href="http://blogs.msdn.com/b/sql_protocols/archive/2005/10/29/486861.aspx?Redirected=true" rel="nofollow">Troubleshoot Connectivity Issue in SQL Server 2005 &#8211; Part II</a> (MDAC)
      * <a href="http://blogs.msdn.com/b/sql_protocols/archive/2005/12/22/506607.aspx?Redirected=true" rel="nofollow">Troubleshoot Connectivity Issue in SQL Server 2005 &#8211; Part III</a> (SqlClient)

Service Broker in the real world
:     * [Developing Large Scale Web Applications and Services](http://mschnlnine.vo.llnwd.net/d1/pdc08/WMV-HQ/BB07.wmv)
      * [MySpace DB Overview](http://www.slideshare.net/markginnebaugh/myspace-sql-server-service-broker-oct-2009)
      * [Real Time Analytics with SQL Server 2008 R2 StreamInsight](http://channel9.msdn.com/learn/courses/SQL2008R2TrainingKit/SQL10R2UPD00/SQL10R2UPD00_REC_03/)
      * [March Madness on Demand](http://blogs.msdn.com/rdoherty/archive/2009/03/13/march-madness-on-demand.aspx)
## SQL vs. NoSQL

  * [The End of an Architectural Era (It’s Time for a Complete Rewrite)](http://vldb.org/conf/2007/papers/industrial/p1150-stonebraker.pdf)
  * [Life beyond Distributed Transactions: an Apostate’s Opinion](http://www-db.cs.wisc.edu/cidr/cidr2007/papers/cidr07p15.pdf)
  * [HStore: A HighPerformance, Distributed Main Memory Transaction Processing System](http://cs-www.cs.yale.edu/homes/dna/papers/hstore-demo.pdf)
  * [Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)</ul> 

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

[Volcano-An Extensible and Parallel Query Evaluation System](http://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf)
:   This is the typical row-mode pull driven query execution model, where operators ask their children for the next tuple.

The Five Minute Rule
:   Ever wondered where does the 300 seconds bufer pool page life expectancy come for? The original research was done back in 1986 and simple economics were involved: given the cost of memory vs. the cost of mass storage in 1986 it was best to keep in memory a page if it was being accessed again in less than 5 minutes. The paper was revisited again in 1996 and 2006 and found that the changes in technologies and cost were canceling each other and kept the equilibrium point still at 5 minutes:</p> 
    
      * [The Five-Minute Rule for Trading Memory for Disk Accesses](http://www.hpl.hp.com/techreports/tandem/TR-86.1.pdf)
      * [The Five-Minute Rule Ten Years Later](ftp://ftp.research.microsoft.com/pub/tr/tr-97-33.pdf)
      * [The Five-Minute Rule 20 Years Later](http://cacm.acm.org/magazines/2009/7/32091-the-five-minute-rule-20-years-later/fulltext)

[The 1995 SQL Reunion: People, Projects, and Politics](http://www.mcjones.org/System_R/SQL_Reunion_95/sqlr95.html)
:   This paper is actually a group interview with most of the people involve din the original release of the very first RDBMS.

[AlphaSort: A Cache-Sensitive Parallel External Sort](http://research.microsoft.com/en-us/um/people/gray/alphasort.doc)
:   Although this is not a RDBMS paper, I choose to post it here none the less. This paper shows the importance of writing cache-concious algorithms in modern processors. Is importance is even greater today, as multi-core systems become prevalent and parallel tasks are writen by everybody in day to day programming.

[Failure Trends in a Large Disk Population](http://static.googleusercontent.com/external_content/untrusted_dlcp/labs.google.com/en/us/papers/disk_failures.pdf)
:   That moment you fear when your database disk will all of the sudden stop to respond may be _way_ closer than you think. Did you test your Disaster Recovery plans and procedures? Oh, and perhaps you should read this also: [Disk Failures in the Real World: What does a MTBF of 100,000 hours mean to you](http://www.cs.cmu.edu/~bianca/fast07.pdf).

## Reference Links

  * <a href="http://msdn.microsoft.com/en-us/library/ms803998.aspx" target="_blank">Windows Performance Counters by Object</a>