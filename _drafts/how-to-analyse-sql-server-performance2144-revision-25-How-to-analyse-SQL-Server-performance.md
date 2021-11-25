---
id: 2172
title: How to analyse SQL Server performance
date: 2014-01-10T05:40:18+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/01/10/2144-revision-25/
permalink: /2014/01/10/2144-revision-25/
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

For a much more detailed explanation read [Understanding how SQL Server executes a query](http://rusanu.com/2013/08/01/understanding-how-sql-server-executes-a-query/).

<p class="callout float-right">
  A request is either executing or is waiting.
</p>

This may sound trivial, but understanding that the request is either executing or is waiting for something (is suspended) is the key to troubleshooting SQL Server performance. If requests sent to SQL Server take a long time to return results then they either take a long time _executing_ or they take a long time _waiting_. Knowing whether is one case or the other is crucial to figure out the performance bottlenecks. Additionally, if the requests take a long time waiting, we can dig further and find out _what_ are they waiting for and for how long.

# Understanding request Waits

<p class="callout float-right">
  Waiting requests have Wait Info data
</p>

Whenever a request is suspended, for _any_ reason, SQL Server will collect information about _why_ it was suspended and for how long. In the internal SQL Server code there is simply no way to call the function that suspends a task without providing the required wait info data. And this data is then collected and made available for you in many ways. This wait info data is crucial in determining performance problems:

  * A requests currently executing that is suspended _right now_ you can see what is it waiting on and how long has been waiting for.
  * A requests currently executing that is executing_right now_ you can see what is the last thing it waited on.
  * You can understand when requests are waiting on other requests.
  * You can get aggregates for what are the most waited for (busy) resources in the entire server.
  * You can understand which physical (hardware) resources are saturated and are causing performance problems.

# Wait Info for currently executing requests

For every request executing on SQL Server there is a row in [<tt>sys.dm_exec_requests</tt>](http://technet.microsoft.com/en-us/library/ms177648.aspx). Querying this DMV, at any moment, gives you a quick view of everything executing **right then**. The <tt>wait_type</tt>, <tt>wait_time</tt> and <tt>last_wait_type</tt> columns will give you an immediate feel of what is &#8216;runnig&#8217; vs. what is &#8216;waiting&#8217; and what is being waited for:

    
    select session_id, 
    	status, 
    	command,
    	blocking_session_id,
    	wait_type, 
    	wait_time,
    	last_wait_type,
    	wait_resource
    from sys.dm_exec_requests
    where session_id >= 50
    and session_id <> @@spid;
    

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_exec_requests_wait.png" alt="" title="dm_exec_requests_wait" width="600" class="alignleft size-full wp-image-2164" />](http://rusanu.com/wp-content/uploads/2014/01/dm_exec_requests_wait.png)

  * Request on session 53 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on a lock.
  * Request on session 54 is an <tt>INSERT</tt> and is currently **waiting**. It is waiting on a page latch.
  * Request on session 55 is an <tt>INSERT</tt> and is currently **running**. The last time it waited it had to wait on a page latch.
  * Request on session 56 is an <tt>INSERT</tt> and is currently **waiting**. It is waiting for the database log to flush (is committing a transaction).
  * Request on session 57 is an <tt>INSERT</tt> and is currently **waiting**. It is waiting for the database log to flush (is committing a transaction).
Note that the request for session 54 has a status of <tt>running</tt> but it is actually waiting. This is because latch waits are expected to be short. The <tt>blocking_session_id</tt> also tells us, for some of the waiting tasks, what other request is holding the resource currently being waited. Session 53 is waiting on session 56 for a KEY lock resource, which means the <tt>SELECT</tt> is trying to read a row locked by the <tt>INSERT</tt>. The <tt>SELECT</tt> will not resume until the <tt>INSERT</tt> commits. Session 54 is waiting on session 55 for a page latch, which means that the session 55 <tt>INSERT</tt> is modifying that data page _right now_ and the data on the page is unstable, not allowed to be read. Sessions 56 and 57 are waiting but there is no other session blocking them. They are waiting for the log to &#8216;flush&#8217;, meaning they must ensure the commit log record for their transaction had been durably written to disk. They will not resume until the disk controller acknowledges that the log was written.

Here is another example:

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_exec_requests_wait_2.png" alt="" title="dm_exec_requests_wait_2" width="600" class="alignleft size-full wp-image-2166" />](http://rusanu.com/wp-content/uploads/2014/01/dm_exec_requests_wait_2.png)

  * Request on session 53 is a <tt>COMMIT</tt> and is currently **waiting**. It is waiting for the database log to flush.
  * Request on session 54 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
  * Request on session 55 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
  * Request on session 56 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
  * Request on session 57 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
Note that in this example every request is actually suspended. At this moment the server is basically doing **nothing**. We have 4 sessions (54, 55, 56 and 57) waiting on the same lock (the row with the key lock resource <tt>KEY: 5:72057594039369728 (ac11a2bc89a9)</tt>). Session 54 is waiting on session 55 and session 55 is waiting on session 53 which means really session 54 is waiting on session 53. So everybody waits on session 53, which is waiting for the disk controller to write a log record on disk. We can also see, from the <tt>wait_time</tt> column, how long has each session waited for: about 160 milliseconds. Notice how in the previous example we had only one request actually running out of 5. I&#8217;m running these queries on a workstation with 8 cores, plenty of RAM and a decent disk, so there is plenty of hardware to carry these requests, but instead they are most of the times _waiting_ instead of executing.

Is this situation unusual, having most (or even all) requests waiting, rather than executing? Not at all, this is the norm! Any time you will look at a SQL Server executing an even moderate load, you&#8217;ll see that most requests are waiting and only a few are executing.

<p class="callout float-left">
  To make your requests faster you need to have them be executing rather than waiting.
</p>