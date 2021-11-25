---
id: 2205
title: How to analyse SQL Server performance
date: 2014-01-13T01:59:29+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/01/13/2144-revision-53/
permalink: /2014/01/13/2144-revision-53/
---
So you have this SQL Server database that your application uses and it somehow seems to be slow. How do you troubleshoot this problem? Where do you look? What do you measure? I will try to give redux of my know-how on this topic. I hope this article will answer enough questions to get you started so that you can identify the bottlenecks yourself, or know what to search for to further extend your arsenal and knowledge.

<!-- more -->

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
  A request is either executing or is waiting
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
  * Session 54 is an <tt>INSERT</tt> and is currently **waiting**. It is waiting on a page latch.
  * Session 55 is an <tt>INSERT</tt> and is currently **running**. Previous wait was a page latch.
  * Session 56 is an <tt>INSERT</tt> and is currently **waiting**. It is waiting for the database log to flush (is committing a transaction).
  * Session 57 is an <tt>INSERT</tt> and is currently **waiting**. It is waiting for the database log to flush.
Note that the request for session 54 has a status of <tt>running</tt> but it is actually waiting. This is because latch waits are expected to be short. The <tt>blocking_session_id</tt> also tells us, for some of the waiting tasks, what other request is holding the resource currently being waited. Session 53 is waiting on session 56 for a KEY lock resource, which means the <tt>SELECT</tt> is trying to read a row locked by the <tt>INSERT</tt>. The <tt>SELECT</tt> will not resume until the <tt>INSERT</tt> commits. Session 54 is waiting on session 55 for a page latch, which means that the session 55 <tt>INSERT</tt> is modifying that data page _right now_ and the data on the page is unstable, not allowed to be read. Sessions 56 and 57 are waiting but there is no other session blocking them. They are waiting for the log to &#8216;flush&#8217;, meaning they must ensure the commit log record for their transaction had been durably written to disk. They will not resume until the disk controller acknowledges that the log was written.

Here is another example:

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_exec_requests_wait_2.png" alt="" title="dm_exec_requests_wait_2" width="600" class="alignleft size-full wp-image-2166" />](http://rusanu.com/wp-content/uploads/2014/01/dm_exec_requests_wait_2.png)

  * Request on session 53 is a <tt>COMMIT</tt> and is currently **waiting**. It is waiting for the database log to flush.
  * Session 54 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
  * Session 55 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
  * Session 56 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
  * Session 57 is a <tt>SELECT</tt> and is currently **waiting**. It is waiting on lock.
Note that in this example every request is actually suspended. At this moment the server is basically doing **nothing**. We have 4 sessions (54, 55, 56 and 57) waiting on the same lock (the row with the key lock resource <tt>KEY: 5:72057594039369728 (ac11a2bc89a9)</tt>). Session 54 is waiting on session 55 and session 55 is waiting on session 53 which means really session 54 is waiting on session 53. So everybody waits on session 53, which is waiting for the disk controller to write a log record on disk. We can also see, from the <tt>wait_time</tt> column, how long has each session waited for: about 160 milliseconds. Notice how in the previous example we had only one request actually running out of 5. I&#8217;m running these queries on a workstation with 8 cores, plenty of RAM and a decent disk, so there is plenty of hardware to carry these requests, but instead they are most of the times _waiting_ instead of executing.

<p class="callout float-left">
  To make your requests faster you need to have them be executing rather than waiting
</p>

Is having most (or even all) requests waiting, rather than executing, something unusual? Not at all, this is the norm! Any time you will look at a SQL Server executing an even moderate load, you&#8217;ll see that most requests are waiting and only a few are executing. What you need to watch out for a **long** waits or short but **repeated** waits that add up. Long waits indicate some resource that is held for long times, and typically this occurs with locks. Repeated short waits indicate a resource that is being saturated, possibly a hot spot for performance.

Before I go further I just want to show [sys.dm\_os\_waiting_tasks](http://technet.microsoft.com/en-us/library/ms188743.aspx), which is a SQL Server DMV specifically designed to show currently waiting tasks:

    
    select * from sys.dm_os_waiting_tasks
    where session_id >= 50
    and session_id <> @@spid;
    

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_waiting_tasks.png" alt="" title="dm_os_waiting_tasks" width="600" class="alignleft size-full wp-image-2173" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_waiting_tasks.png)

  * Session 53 is waiting the log to flush
  * Session 57 is waiting for 40 ms for a lock on a row and is blocked by session 53 (which therefore must be owning the lock)
  * Session 54 is waiting for 37 ms for a lock on a row and is blocked by session 57 (but session 57 is in turn blocked by session 53)

The situation we have here is much similar to the previous one, where we had 4 <tt>SELECT</tt> sessions blocked by an <tt>INSERT</tt>. He can see here two requests blocked by attempting to read a row (so they&#8217;re probably <tt>SELECT</tt>) and they blocking session is waiting for its transaction to durably commit. The information in this DMV is very similar to the one in <tt>sys.dm_exec_requests</tt> and the later one has more information, but there is one important case where the information in <tt>sys.dm_exec_requests</tt> is misleading: parallel queries.

<p class="callout float-right">
  To understand <tt>CXPACKET</tt> wait types you need to look into child parallel tasks
</p>

When a statement can benefit from parallel execution the engine will create multiple tasks for the request, each processing a subset of the data. Each one of these tasks can execute on a separate CPU/core. The request communicate with these tasks using basically a producer-consumer queue. The query operator implementing this queue is called an [Exchange operator](http://technet.microsoft.com/en-us/library/ms178065(v=sql.105).aspx) (I&#8217;m really simplifying here, read [The Parallelism Operator (aka Exchange)](http://blogs.msdn.com/b/craigfr/archive/2006/10/25/the-parallelism-operator-aka-exchange.aspx) for a more accurate description). If this producer-consumer queue is empty (meaning the producers did not push any data into it) the consumer must suspend and wait and the corresponding wait type is the <tt>CXPACKET</tt> wait type. Requests that show this wait type are really showing that the tasks which should had produced data to consume are not producing any (or enough) data. These producer tasks in turn may be suspended, waiting on some other wait type and that is what is blocking your request, not the exchange operator. Here is an example:

    
    select r.session_id, 
    	status, 
    	command,
    	r.blocking_session_id,
    	r.wait_type as [request_wait_type], 
    	r.wait_time as [request_wait_time],
    	t.wait_type as [task_wait_type],
    	t.wait_duration_ms as [task_wait_time],
    	t.blocking_session_id,
    	t.resource_description
    from sys.dm_exec_requests r
    left join sys.dm_os_waiting_tasks t
    	on r.session_id = t.session_id
    where r.session_id >= 50
    and r.session_id <> @@spid;
    

[<img src="http://rusanu.com/wp-content/uploads/2014/01/cxpacket_wait.png" alt="" title="cxpacket_wait" width="600" class="alignleft size-full wp-image-2179" />](http://rusanu.com/wp-content/uploads/2014/01/cxpacket_wait.png)

Here we can see that request on session 54 has 3 tasks waiting, and while the request <tt>wait_type</tt> shows <tt>CXPACKET</tt>, one of the parallel child tasks is actually waiting on a page lock. 

## Aggregated wait stats: sys.dm\_os\_wait_stats

SQL Server aggregates statistics about all wait types and exposes them in a [<tt>sys.dm_os_wait_stats</tt>](http://msdn.microsoft.com/en-us/library/ms179984.aspx). If looking at the currently executing requests and the waiting tasks showed us what was being waited at any moment, the aggregate stats give a cumulative status since server start up. Querying this DMV is straightforward, but interpreting the results is a bit more tricky:

    
    select * 
    from sys.dm_os_wait_stats
    order by wait_time_ms desc;
    

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats.png" alt="" title="dm_os_wait_stats" width="600" class="alignleft size-full wp-image-2182" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats.png)

First, what&#8217;s the deal with all those wait types that take the lion&#8217;s share at the top of the result? <tt>DIRTY_PAGE_POOL</tt>, <tt>REQUEST_FOR_DEADLOCK_SEARCH</tt>, <tt>LAZYWRITER_SLEEP</tt>? We did not see any request ever waiting for these! Here is why: by adding the <tt>WHERE session_id >= 50</tt> we filtered out background tasks, threads internal for SQL Server that are tasked with various maintenance jobs. Many of these background tasks follow a pattern like &#8220;wait for an event, when the event is signaled do some work, then wait again&#8221;, while other are patterned in a sleep cycle pattern (&#8220;sleep for 5 seconds, wake up and do something, go back to sleep 5 seconds&#8221;). Since no task can suspend itself without providing a wait type, there are wait types to capture when these background tasks suspend themselves. As these patterns of execution result in the task being actually suspended almost all time, the aggregated wait time for these wait types often trumps every other wait type. The SQL Server community came to name these wait types &#8216;benign wait types&#8217; and most experts have a list of wait types they simply filter out, for example see [Filtering out benign waits](http://sqlmag.com/blog/filtering-out-benign-waits):

    
    select * 
    from sys.dm_os_wait_stats
    WHERE [wait_type] NOT IN (
            N'CLR_SEMAPHORE',    N'LAZYWRITER_SLEEP',
            N'RESOURCE_QUEUE',   N'SQLTRACE_BUFFER_FLUSH',
            N'SLEEP_TASK',       N'SLEEP_SYSTEMTASK',
            N'WAITFOR',          N'HADR_FILESTREAM_IOMGR_IOCOMPLETION',
            N'CHECKPOINT_QUEUE', N'REQUEST_FOR_DEADLOCK_SEARCH',
            N'XE_TIMER_EVENT',   N'XE_DISPATCHER_JOIN',
            N'LOGMGR_QUEUE',     N'FT_IFTS_SCHEDULER_IDLE_WAIT',
            N'BROKER_TASK_STOP', N'CLR_MANUAL_EVENT',
            N'CLR_AUTO_EVENT',   N'DISPATCHER_QUEUE_SEMAPHORE',
            N'TRACEWRITE',       N'XE_DISPATCHER_WAIT',
            N'BROKER_TO_FLUSH',  N'BROKER_EVENTHANDLER',
            N'FT_IFTSHC_MUTEX',  N'SQLTRACE_INCREMENTAL_FLUSH_SLEEP',
            N'DIRTY_PAGE_POLL',  N'SP_SERVER_DIAGNOSTICS_SLEEP')
    order by wait_time_ms desc;
    

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats_filtered.png" alt="" title="dm_os_wait_stats_filtered" width="600" class="alignleft size-full wp-image-2183" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats_filtered.png)

Now we have a more coherent picture of the waits. the picture tells us what wait types are most prevalent, on aggregate, on this SQL Server instance. This can be an important step toward identifying a bottleneck cause. Furthermore, interpreting the data is data is subjective. Is the aggregate <tt>WRITELOG</tt> value of 4636805 milliseconds good or bad? I don&#8217;t know! Is an aggregate, accumulated since this process is running, so will obviously contiguously increase. Perhaps it had accumulated values from periods when the server was running smooth and from periods when it was running poorly. However, I now know that, from all &#8216;non bening&#8217; wait types, <tt>WRITELOG</tt> is the one with the highest total wait time. I also know that <tt>max_wait_time</tt> for these wait types, so I know that there was at least one time when a task had to wait 1.4 seconds for the log to flush. The <tt>wait_task_count</tt> tells me how many time a task waited on a particular wait type, and dividing <tt>wait_time_ms/wait_task_count</tt> will tell me the average time a particular wait type has been waited on. And I see several lock related wait types in top: <tt>LCK_M_S</tt>, <tt>LCK_M_IS</tt> and <tt>LCK_M_IX</tt>. I can already tell some things about the overall behavior of the server, from a performance point of view: flushing the log is most waited resource on the server, and lock contention seems to be the next major issue. <tt>CXPACKET</tt> wait types are right there on top, but we know that most times <tt>CXPACKET</tt> wait types are just surrogate for other wait types, as discussed above. How about those other with types in the list, like <tt>SOS_SCHEDULER_YIELD</tt>, <tt>PAGELATCH_EX/SH</tt> or <tt>PAGEIOLATCH_EX/SH/UP</tt>?

## Understanding Wait Types names

In order to investigate wait types and wait times, either aggregate or for an individual query, one needs to understand what the wait type names mean. Many of the names are self-describing, some are cryptic in their name and some are down right deceitfully named. I think the best resource to read a detailed description of the names is the [Waits and Queues](http://technet.microsoft.com/en-us/library/cc966413.aspx) white paper. It has an alphabetical list of wait types, description and what physical resource is correlated with. The white paper is a bit old (it covers SQL Server 2005) but still highly relevant. The wait type names for important resources has not changed since SQL server 2005, what you&#8217;ll be missing will be newer wait type names related to post SQL Server 2005 features. The wait types are briefly documented also in the [sys.dm\_os\_wait_types](http://msdn.microsoft.com/en-us/library/ms179984.aspx) topic on MSDN.

### Disk and IO related wait types

<tt><b>PAGEIOLATCH_*</b></tt>
:   This is the quintessential IO, data read from disk and write to disk wait type. A task blocked on these wait type is waiting for data to be transferred between the disk and the in-memory data cache (the buffer pool). A system that has high aggregate <tt>PAGEIOLATCH_*</tt> wait types is very likely memory starved and is spending much time reading data from the disk into the buffer pool. Be careful not to confuse this wait type with the <tt>PAGELATCH_*</tt> wait type (not the missing <tt>IO</tt> in the name).

<tt><b>WRITELOG</b></tt>
:   This wait type occurs when a task is issuing a <tt>COMMIT</tt> and it waits for the log to write the transaction commit log record to disk. High average wait times on this wait type indicate that the disk is writing the log slowly and this slows down every transaction. Very frequent waits on this wait type are indicative of doing many small transaction and having to block frequently to wait for COMMIT (remember that **all** data writes require a transaction and one is implicitly created for each statement if no BEGIN TRANSACTION is explicitly used).

<tt><b>IO_COMPLETION</b></tt>
:   This wait type occurs for tasks that are waiting for something else that ordinary data IO. Eg. loading an assembly DLL, reading and writing [TEMPDB sort files](http://support.microsoft.com/kb/309392), some special DBCC data reading operations.

<tt><b>ASYNC_IO_COMPLETION</b></tt>
:   This wait type is mostly associated with backup, restore, database and database file operations.

### Memory related wait types

<tt><b>RESOURCE_SEMAPHORE</b></tt>
:   This wait type indicate queries that are waiting for a memory grant. See [Understanding SQL Server memory grant](http://blogs.msdn.com/b/sqlqueryprocessing/archive/2010/02/16/understanding-sql-server-memory-grant.aspx). OLTP type workload queries should not require memory grants,, if you see this wait type on an OLTP system then you need to revisit your application design. OLAP workloads often are in need of (sometimes large) memory grants and large wait times on this wait type usually point toward RAM increase upgrades.

<tt><b>SOS_VIRTUALMEMORY_LOW</b></tt>
:   Are you still on 32 bit systems? Move on!

### Network related wait types

<tt><b>ASYNC_NETWORK_IO</b></tt>
:   This wait type indicate that the SQL Server has result sets to send to the application but the application odes not process them. This _may_ indicate a slow network connection. But more often the problem is with the application code, it is either blocking while processing the result set or is requesting a huge result set.

<tt><b></b></tt>
:   

### CPU related wait types

<tt><b></b></tt>
:   

<tt><b></b></tt>
:   

### Contention and concurrency related wait types

<tt><b>LCK_*</b></tt>
:   Locks. All wait types that start with <tt>LCK</tt> indicate a task suspended waiting for a lock. <tt>LCK_M_S*</tt> wait types indicate a task that is waiting to read data (shared locks) and is blocked by another task that had modified the data (had acquired an exclusive <tt>LCK_MX*</tt> lock). <tt>LCK_M_SCH*</tt> wait types indicate object schema modification locks and indicate that access to an object (eg. a table) is blocked by another task that did a DDL modfication to that object (<tt>ALTER</tt>).

<tt><b>PAGELATCH_*</b></tt>
:   Do not confuse this wait types with the <tt>PAGEIOLATCH_*</tt> wait types. High wait times on these wait types indicate a hot spot in the database, a region of the data that is very frequently updated (eg. it could be a single record in a table that is constantly modified). To further analyze, I recommend the [Diagnosing and Resolving Latch Contention on SQL Server](http://download.microsoft.com/download/B/9/E/B9EDF2CD-1DBF-4954-B81E-82522880A2DC/SQLServerLatchContention.pdf) white paper.

<tt><b>LATCH_*</b></tt>
:   These wait types indicate contention on internal SQL Server resources, not on data (unlike <tt>PAGELATCH_*</tt> they do not indicate a hot spot in the data). To investigate these waits you need to dig further using the [<tt>sys.dm_os_latch_stats</tt>](http://technet.microsoft.com/en-us/library/ms175066.aspx) DMV which will detail the wait times by latch type. Again, the best resource is [Diagnosing and Resolving Latch Contention on SQL Server](http://download.microsoft.com/download/B/9/E/B9EDF2CD-1DBF-4954-B81E-82522880A2DC/SQLServerLatchContention.pdf) white paper.

<tt><b>CMEMTHREAD</b></tt>
:   This wait type occurs when tasks block on waiting to access a shared memory allocator. I placed thi here, in concurrency section, and not in &#8216;memory&#8217; section because the problem is related to internal SQL Server concurrency. If you see high <tt>CMEMTHREAD</tt> wait types you should make sure you are at the latest SQL Server available Service Pack and Cumulative Update for your version, because these kind of problems denote internal SQL Server issues and are often addressed in newer releases.

<tt><b>SOS_SCHEDULER_YIELD</b></tt>
:   This wait type can indicate spinlock contention. Spinlocks are extremely light wait locking primitives in SQL Server used for protecting access to resources that can be modified within few CPU instructions. SQL Server tasks acquire spinlocks by doing interlocked CPU operations in a tight loop, so contention on spinlocks burns a lot of CPU time (CPU usage counters show 90-100% usage, but progress is slow). Further analyis need to be done using <tt>sys.dm_os_spinlock_stats</tt>:</p> 
    
        
        select * from sys.dm_os_spinlock_stats 
        order by spins desc;
        

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_spinlock_stats.png" alt="" title="dm_os_spinlock_stats" width="600" class="alignleft size-full wp-image-2203" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_spinlock_stats.png) </dl> 

### Special wait types

<tt><b>TRACEWRITE</b></tt>
:   This wait type indicate that tasks are being blocked by SQL Profiler. This wait type occurs only if you have SQL Profiler attached to the server. This wait type occur often during investigation if you had set up a SQL Profiler trace that is too aggressive (gets too many events).