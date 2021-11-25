---
id: 2153
title: How to analyse SQL Server performance
date: 2014-01-07T03:26:05+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/01/07/2144-revision-9/
permalink: /2014/01/07/2144-revision-9/
---
So you have this SQL Server database that your application uses and it somehow seems to be slow. How do you troubleshoot this problem? Where do you look? What do you measure? I will try to give redux of my know-how on this topic. I hope this article will answer enough questions to get you started so that you can identify the bottlenecks yourself, or know what to search for to further extend your arsenal and knowledge.

# How does SQL Server work

To be able to troubleshoot performance problems you need to have an understanding of how SQL Server works. The grossly simplified explanation is that SQL Server executes your queries as follows:

  * The application sends a <tt>request</tt> to the server, containing a stored procedure name or some T-SQL statements.
  * The request is placed in a queue inside SQL Server memory.
  * A free thread from SQL Server&#8217;s own thread pool picks up the request, compiles it and executes it.
  * The request is executed statement by statement, sequentially. A statement in a request must finish before the next starts, always. Stored procedures executes the same way, statement by statement.
  * Statements can read or modify data. All data is read from in-memory cache of the database (the <tt>buffer pool</tt>). If data is not in this cache, it must be read from disk into the cache. All updates are written into the database log and into the in-memory cache (into the buffer pool), according to the [Write-Ahead Logging protocol](http://technet.microsoft.com/en-us/library/ms186259(v=sql.105).aspx).
  * Data locking ensures correctness under concurrency.
  * When all statements in the request have executed, the thread is free to pick up another request and execute it.

for a much more detailed explanation read [Understanding how SQL Server executes a query](http://rusanu.com/2013/08/01/understanding-how-sql-server-executes-a-query/).

# What could cause SQL Server to be slow

<p class="callout float-right">
  Once started a request is either executing or is suspended.
</p>

This may sound trivial, but understanding that a request (and hence, statements executed as part of the request) is either executing or is suspended is the key to troubleshooting SQL Server performance.