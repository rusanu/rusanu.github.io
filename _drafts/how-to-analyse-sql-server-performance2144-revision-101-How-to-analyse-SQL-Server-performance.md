---
id: 2259
title: How to analyse SQL Server performance
date: 2014-02-19T07:40:51+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/02/19/2144-revision-101/
permalink: /2014/02/19/2144-revision-101/
---
So you have this SQL Server database that your application uses and it somehow seems to be slow. How do you troubleshoot this problem? Where do you look? What do you measure? I hope this article will answer enough questions to get you started so that you can identify the bottlenecks yourself, or know what to search for to further extend your arsenal and knowledge.

  * [How does SQL Server work](#how_it_works)
  * [Wait Info for currently executing requests](#wait_info_current)
  * [Aggregated wait stats: sys.dm\_os\_wait_stats](#wait_stats)
  * [Common wait types](#common_wait_types)
  * [Analyze disk activity: IO stats](#virtual_io_stats)
  * [Analyzing individual query execution](#analyze_individual_query)
  * [Identifying problem queries](#identify_problem_query)
  * [Missing Indexes](#missing_indexes)

<!-- more -->

# <a name="how_it_works"></a>How does SQL Server work

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

# <a name="wait_info_current"></a>Wait Info for currently executing requests

For every request executing on SQL Server there is a row in [<tt>sys.dm_exec_requests</tt>](http://technet.microsoft.com/en-us/library/ms177648.aspx). Querying this DMV, at any moment, gives you a quick view of everything executing **right then**. The <tt>wait_type</tt>, <tt>wait_time</tt> and <tt>last_wait_type</tt> columns will give you an immediate feel of what is &#8216;runnig&#8217; vs. what is &#8216;waiting&#8217; and what is being waited for:

<pre><code class="prettyprint lang-sql">
select session_id, 
	status, 
	command,
	blocking_session_id,
	wait_type, 
	wait_time,
	last_wait_type,
	wait_resource
from sys.dm_exec_requests
where r.session_id &gt;= 50
and r.session_id &lt;&gt; @@spid;
</code></pre>

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

<pre><code class="prettyprint lang-sql">
select * from sys.dm_os_waiting_tasks
where r.session_id &gt;= 50
and r.session_id &lt;&gt; @@spid;
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_waiting_tasks.png" alt="" title="dm_os_waiting_tasks" width="600" class="alignleft size-full wp-image-2173" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_waiting_tasks.png)

  * Session 53 is waiting the log to flush
  * Session 57 is waiting for 40 ms for a lock on a row and is blocked by session 53 (which therefore must be owning the lock)
  * Session 54 is waiting for 37 ms for a lock on a row and is blocked by session 57 (but session 57 is in turn blocked by session 53)

The situation we have here is much similar to the previous one, where we had 4 <tt>SELECT</tt> sessions blocked by an <tt>INSERT</tt>. He can see here two requests blocked by attempting to read a row (so they&#8217;re probably <tt>SELECT</tt>) and they blocking session is waiting for its transaction to durably commit. The information in this DMV is very similar to the one in <tt>sys.dm_exec_requests</tt> and the later one has more information, but there is one important case where the information in <tt>sys.dm_exec_requests</tt> is misleading: parallel queries.

<p class="callout float-right">
  To understand <tt>CXPACKET</tt> wait types you need to look into child parallel tasks
</p>

When a statement can benefit from parallel execution the engine will create multiple tasks for the request, each processing a subset of the data. Each one of these tasks can execute on a separate CPU/core. The request communicate with these tasks using basically a producer-consumer queue. The query operator implementing this queue is called an [Exchange operator](http://technet.microsoft.com/en-us/library/ms178065(v=sql.105).aspx) (I&#8217;m really simplifying here, read [The Parallelism Operator (aka Exchange)](http://blogs.msdn.com/b/craigfr/archive/2006/10/25/the-parallelism-operator-aka-exchange.aspx) for a more accurate description). If this producer-consumer queue is empty (meaning the producers did not push any data into it) the consumer must suspend and wait and the corresponding wait type is the <tt>CXPACKET</tt> wait type. Requests that show this wait type are really showing that the tasks which should had produced data to consume are not producing any (or enough) data. These producer tasks in turn may be suspended, waiting on some other wait type and that is what is blocking your request, not the exchange operator. Here is an example:

<pre><code class="prettyprint lang-sql">
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
where r.session_id &gt;= 50
and r.session_id &lt;&gt; @@spid;
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2014/01/cxpacket_wait.png" alt="" title="cxpacket_wait" width="600" class="alignleft size-full wp-image-2179" />](http://rusanu.com/wp-content/uploads/2014/01/cxpacket_wait.png)

Here we can see that request on session 54 has 3 tasks waiting, and while the request <tt>wait_type</tt> shows <tt>CXPACKET</tt>, one of the parallel child tasks is actually waiting on a page lock. 

## <a name="wait_stats"></a>Aggregated wait stats: sys.dm\_os\_wait_stats

SQL Server aggregates statistics about all wait types and exposes them in a [<tt>sys.dm_os_wait_stats</tt>](http://msdn.microsoft.com/en-us/library/ms179984.aspx). If looking at the currently executing requests and the waiting tasks showed us what was being waited at any moment, the aggregate stats give a cumulative status since server start up. Querying this DMV is straightforward, but interpreting the results is a bit more tricky:

<pre><code class="prettyprint lang-sql">
select * 
from sys.dm_os_wait_stats
order by wait_time_ms desc;
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats.png" alt="" title="dm_os_wait_stats" width="600" class="alignleft size-full wp-image-2182" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats.png)

First, what&#8217;s the deal with all those wait types that take the lion&#8217;s share at the top of the result? <tt>DIRTY_PAGE_POOL</tt>, <tt>REQUEST_FOR_DEADLOCK_SEARCH</tt>, <tt>LAZYWRITER_SLEEP</tt>? We did not see any request ever waiting for these! Here is why: by adding the <tt>WHERE session_id >= 50</tt> we filtered out background tasks, threads internal for SQL Server that are tasked with various maintenance jobs. Many of these background tasks follow a pattern like &#8220;wait for an event, when the event is signaled do some work, then wait again&#8221;, while other are patterned in a sleep cycle pattern (&#8220;sleep for 5 seconds, wake up and do something, go back to sleep 5 seconds&#8221;). Since no task can suspend itself without providing a wait type, there are wait types to capture when these background tasks suspend themselves. As these patterns of execution result in the task being actually suspended almost all time, the aggregated wait time for these wait types often trumps every other wait type. The SQL Server community came to name these wait types &#8216;benign wait types&#8217; and most experts have a list of wait types they simply filter out, for example see [Filtering out benign waits](http://sqlmag.com/blog/filtering-out-benign-waits):

<pre><code class="prettyprint lang-sql">
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
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats_filtered.png" alt="" title="dm_os_wait_stats_filtered" width="600" class="alignleft size-full wp-image-2183" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_wait_stats_filtered.png)

Now we have a more coherent picture of the waits. the picture tells us what wait types are most prevalent, on aggregate, on this SQL Server instance. This can be an important step toward identifying a bottleneck cause. Furthermore, interpreting the data is data is subjective. Is the aggregate <tt>WRITELOG</tt> value of 4636805 milliseconds good or bad? I don&#8217;t know! Is an aggregate, accumulated since this process is running, so will obviously contiguously increase. Perhaps it had accumulated values from periods when the server was running smooth and from periods when it was running poorly. However, I now know that, from all &#8216;non bening&#8217; wait types, <tt>WRITELOG</tt> is the one with the highest total wait time. I also know that <tt>max_wait_time</tt> for these wait types, so I know that there was at least one time when a task had to wait 1.4 seconds for the log to flush. The <tt>wait_task_count</tt> tells me how many time a task waited on a particular wait type, and dividing <tt>wait_time_ms/wait_task_count</tt> will tell me the average time a particular wait type has been waited on. And I see several lock related wait types in top: <tt>LCK_M_S</tt>, <tt>LCK_M_IS</tt> and <tt>LCK_M_IX</tt>. I can already tell some things about the overall behavior of the server, from a performance point of view: flushing the log is most waited resource on the server, and lock contention seems to be the next major issue. <tt>CXPACKET</tt> wait types are right there on top, but we know that most times <tt>CXPACKET</tt> wait types are just surrogate for other wait types, as discussed above. How about those other with types in the list, like <tt>SOS_SCHEDULER_YIELD</tt>, <tt>PAGELATCH_EX/SH</tt> or <tt>PAGEIOLATCH_EX/SH/UP</tt>?

## <a name="common_wait_types"></a>Common wait types

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

If your wait time analysis finds that IO and disk are important wait types then your should focus on [analyzing disk activity](#virtual_io_stats).

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

### CPU, Contention and Concurrency related wait types

<tt><b>LCK_*</b></tt>
:   Locks. All wait types that start with <tt>LCK</tt> indicate a task suspended waiting for a lock. <tt>LCK_M_S*</tt> wait types indicate a task that is waiting to read data (shared locks) and is blocked by another task that had modified the data (had acquired an exclusive <tt>LCK_MX*</tt> lock). <tt>LCK_M_SCH*</tt> wait types indicate object schema modification locks and indicate that access to an object (eg. a table) is blocked by another task that did a DDL modfication to that object (<tt>ALTER</tt>).

<tt><b>PAGELATCH_*</b></tt>
:   Do not confuse this wait types with the <tt>PAGEIOLATCH_*</tt> wait types. High wait times on these wait types indicate a hot spot in the database, a region of the data that is very frequently updated (eg. it could be a single record in a table that is constantly modified). To further analyze, I recommend the [Diagnosing and Resolving Latch Contention on SQL Server](http://download.microsoft.com/download/B/9/E/B9EDF2CD-1DBF-4954-B81E-82522880A2DC/SQLServerLatchContention.pdf) white paper.

<tt><b>LATCH_*</b></tt>
:   These wait types indicate contention on internal SQL Server resources, not on data (unlike <tt>PAGELATCH_*</tt> they do not indicate a hot spot in the data). To investigate these waits you need to dig further using the [<tt>sys.dm_os_latch_stats</tt>](http://technet.microsoft.com/en-us/library/ms175066.aspx) DMV which will detail the wait times by latch type. Again, the best resource is [Diagnosing and Resolving Latch Contention on SQL Server](http://download.microsoft.com/download/B/9/E/B9EDF2CD-1DBF-4954-B81E-82522880A2DC/SQLServerLatchContention.pdf) white paper.

<tt><b>CMEMTHREAD</b></tt>
:   This wait type occurs when tasks block on waiting to access a shared memory allocator. I placed thi here, in concurrency section, and not in &#8216;memory&#8217; section because the problem is related to internal SQL Server concurrency. If you see high <tt>CMEMTHREAD</tt> wait types you should make sure you are at the latest SQL Server available Service Pack and Cumulative Update for your version, because some of these kind of problems denote internal SQL Server issues and are often addressed in newer releases.

<a name="SOS_SCHEDULER_YIELD"></a><tt><b>SOS_SCHEDULER_YIELD</b></tt>
:   This wait type can indicate spinlock contention. Spinlocks are extremely light wait locking primitives in SQL Server used for protecting access to resources that can be modified within few CPU instructions. SQL Server tasks acquire spinlocks by doing interlocked CPU operations in a tight loop, so contention on spinlocks burns a lot of CPU time (CPU usage counters show 90-100% usage, but progress is slow). Further analysis need to be done using <tt>sys.dm_os_spinlock_stats</tt>:</p> 
    
    <pre><code class="prettyprint lang-sql">
select * from sys.dm_os_spinlock_stats 
order by spins desc;
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_os_spinlock_stats.png" alt="" title="dm_os_spinlock_stats" width="600" class="alignleft size-full wp-image-2203" />](http://rusanu.com/wp-content/uploads/2014/01/dm_os_spinlock_stats.png)  
For further details I recommend reading the [Diagnosing and Resolving Spinlock Contention on SQL Server](http://www.microsoft.com/en-us/download/details.aspx?id=26666) white paper.

<tt><b>RESOURCE_SEMAPHORE_QUERY_COMPILE</b></tt>
:   This wait type indicate that a task is waiting to compile its request. High wait times of this type indicate that query compilation is a bottleneck. For more details I recommend reading [Troubleshooting Plan Cache Issues](http://technet.microsoft.com/en-us/library/cc293620.aspx).

<a name="SQLCLR_QUANTUM_PUNISHMENT"></a><tt><b>SQLCLR_QUANTUM_PUNISHMENT</b></tt>
:   This wait type occurs if you run CLR code inside the SQL Server engine and your CLR code does not yield the CPU in time. This results in an throttling of the CLR code (&#8220;punishment&#8221;). If you have CLR code that can potentially hijack the CPU for a long time you **must** call [<tt>Thread.BeginThreadAffinity()</tt>](http://msdn.microsoft.com/en-us/library/system.threading.thread.beginthreadaffinity(v=vs.110).aspx). For more details I recommend watching [Data, Faster: Microsoft SQL Server Performance Techniques with SQLCLR](http://channel9.msdn.com/Events/TechEd/NorthAmerica/2013/DBI-B404#fbid=).

### Special wait types

<tt><b>TRACEWRITE</b></tt>
:   This wait type indicate that tasks are being blocked by SQL Profiler. This wait type occurs only if you have SQL Profiler attached to the server. This wait type  
    occur often during investigation if you had set up a SQL Profiler trace that is too aggressive (gets too many events).

<tt><b>PREEMPTIVE_OS_WRITEFILEGATHER</b></tt>
:   This wait type occurs, among other things, when auto-growth of files is triggered. Auto-growth occurs when a insufficiently sized file is grown by SQL Server and is a hugely expensive event. During growth event all activity on the database is frozen. Data file growth can be made fast by enabling instant file growth, see [Database File Initialization](http://msdn.microsoft.com/en-us/library/ms175935.aspx). But log growth cannot benefit from instant file initialization so log growth is always slow, sometimes **very** slow. Log Auto-Growth events can be diagnosed by simply looking at the Log Growths performance counter in the [SQL Server, Databases Object](http://msdn.microsoft.com/en-us/library/ms189883(v=sql.100).aspx), if is anything but 0 it means log auto-growth had occurred at least once. Real-time monitoring can be done by watching for [Data File Auto Grow Event Class](http://technet.microsoft.com/en-us/library/ms187491.aspx) and [Log File Auto Grow Event Class](http://technet.microsoft.com/en-us/library/ms191263.aspx) events in SQL Profiler.

I did not cover many wait types here (eg. <tt>CLR_*</tt>, <tt>SQLCLR_*</tt>, <tt>SOSHOST_*</tt>, <tt>HADR_*</tt>, <tt>DTC_*</tt> and many more). If you encounter a wait type you do not understand, usually just searching online for its name will reveal information about what that wait type is what potential bottleneck it indicates, if any.

# <a name="virtual_io_stats"></a>Analyze disk activity: IO stats

SQL Server needs to read and write to the disk. All writes (inserts, updates, deletes) must be written to disk. Queries always return data from the in-memory cache (the buffer pool) but the cache may not contain the desired data and has to be read from disk. Understanding if the IO is a bottleneck for your performance is a necessary step in any performance investigation. SQL Server collects, aggregates and exposes information about every data and log IO request. First thing I like to look at is [<tt>sys.dm_io_virtual_file_stats</tt>](http://technet.microsoft.com/en-us/library/ms190326.aspx). This DMV exposes the number of writes and reads on each file for every database in the system, along with the aggregated read and write IO &#8216;stall&#8217; times. Stall times are the total time tasks had to block waiting for transfer of data to and from disk.

<pre><code class="prettyprint lang-sql">
select db_name(io.database_id) as database_name,
	mf.physical_name as file_name,
	io.* 
from sys.dm_io_virtual_file_stats(NULL, NULL) io
join sys.master_files mf on mf.database_id = io.database_id 
	and mf.file_id = io.file_id
order by (io.num_of_bytes_read + io.num_of_bytes_written) desc;
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2014/01/dm_io_virtual_file_stats.png" alt="" title="dm_io_virtual_file_stats" width="600" class="alignleft size-full wp-image-2219" />](http://rusanu.com/wp-content/uploads/2014/01/dm_io_virtual_file_stats.png)

The per file IO stats are aggregated since server start up but be aware that they reset for each file if it ever goes offline. The total number of bytes transferred (reads and writes) is a good indicator of how busy is a database from IO point of view. The stalls indicate which IO subsytem (which disk) is busy and may even be saturated. 

# <a name="analyze_individual_query"></a>Analyzing individual query execution

When analyzing an individual statement or query for performance allows us to understand what the query is doing, how much is executing vs. what is waiting on. The most simple analysis is to use the execution statistics. These work by first running the appropriate <tt>SET STATISTICS ... ON</tt> in your session and then executing the statement or query you want to analyze. In addition to the usual result, the query will also report execution statistics, as follows:

[<tt>SET STATISTICS TIME ON</tt>](http://technet.microsoft.com/en-us/library/ms190287.aspx)
:   The query parse time, compile time and execution time are reported. If you execute a batch or a stored procedure with multiple statements then you get back the time for each statement, which helps determining which statement in a procedure is the expensive one. But we&#8217;ll see that analyzing the execution plan reveals the same information, and more.

[<tt>SET STATISTICS IO ON</tt>](http://technet.microsoft.com/en-us/library/ms184361.aspx)
:   The IO statistics show, for each statement, the number of IO operations. The result shows several metrics _for each table touched by the statement_:</p> 
    
    scan count
    :   Number of times the scans or seeks were _started_ on the table. Ideally each table should be scanned at most once.
    
    logical reads
    :   Number of data pages from which rows were read from the in-memory cache (the buffer pool).
    
    physical reads
    :   Number of data pages from which data was had to be transferred from disk in-memory cache (the buffer pool) and the task had to block and wait for the transfer to finish.
    
    read-ahead reads
    :   Number of data pages that were asynchronously transferred from disk into the buffer pool _opportunistically_ and the task did not had to wait for the transfer.
    
    _LOB_ logical/physical/read-ahead reads
    :   Same as their non-lob counterparts, but referring to reading of large columns (LOBs).

There is also the [<tt>SET STATISTICS PROFILE ON</tt>](http://technet.microsoft.com/en-us/library/ms188752.aspx), which provides a detailed execution information for a statement, but much of the information overlaps with the execution plan info and I think analyzing the execution plan is easier and yields better information.

<p class="callout float-right">
  Logical Reads are the best measure of a statement cost.
</p>

A large logical reads value for a statement indicates an expensive statement. For OLTP environments the number of logical reads should be in single digits. Large logical reads count translate to long execution times and high CPU consumption even under ideally conditions. Under less ideal conditions some or all of these logical reads turn into physical reads which mean long wait times waiting for disk transfer of data into the buffer pool. Large logical reads are indicative of large data scans, usually entire table end-to-end scan. They are most times caused by missing indexes. In OLAP, decision and DW environments, scans and large logical reads may be unavoidable as indexing for an OLAP workload is tricky, and indexing for an ad-hoc analytic workload is basically impossible. For these later cases the answer usually relies in using DW specific technologies like [columnstores](http://rusanu.com/2012/05/29/inside-the-sql-server-2012-columnstore-indexes/).

Information about query execution statistics can also be obtained using SQL Profiler. The [<tt>SQL:StmtCompleted</tt>](http://technet.microsoft.com/en-us/library/ms189886.aspx), [<tt>SQL:BatchCompleted</tt>](http://technet.microsoft.com/en-us/library/ms176010.aspx),[<tt>SP:StmtCompleted</tt>](http://technet.microsoft.com/en-us/library/ms189570.aspx) and [<tt>RPC:Completed</tt>](http://technet.microsoft.com/en-us/library/ms175543.aspx) capture the number of page reads, the number of page writes and the execution time of an individual invocation of a statement, a stored procedure or a SQL batch. The number of reads is logical reads. Note that monitoring with SQL Profiler for individual statement completed events will have effect on performance and on a big busy server it can cause significant performance degradation.

You can also see the number of logical reads, reads and writes _as it happens_ by looking at the <tt>sys.dm_exec_requests</tt> but this information is visible only if while a request is running. Furthermore the number aggregate all statements in a request so you cannot determine the cost of an individual statement using this DMV.

## Analyzing individual query execution wait times

If you need to drill down more detail that execution time and IO statistics for a query or for a stored procedure, the best information is the wait types and wait times that occurred during execution. Obtaining this information is no longer straightforward though, it involves using [Extended Events](http://technet.microsoft.com/en-us/library/bb630354(v=sql.105).aspx) monitoring. The method I&#8217;m presenting here is based on the [Debugging slow response times in SQL Server 2008](http://blogs.technet.com/b/sqlos/archive/2008/07/18/debugging-slow-response-times-in-sql-server-2008.aspx) article. It involves creating an extended event session that captures the <tt>sqlos.wait_info</tt> info and filter the extended events session for a specific execution session (SPID):

<pre><code class="prettyprint lang-sql">
create event session session_waits on server
add event sqlos.wait_info
(WHERE sqlserver.session_id=&lt;execution_spid_here&gt; and duration&gt;0)
, add event sqlos.wait_info_external
(WHERE sqlserver.session_id=&lt;execution_spid_here&gt; and duration&gt;0)
add target package0.asynchronous_file_target
      (SET filename=N'c:\temp\wait_stats.xel', metadatafile=N'c:\temp\wait_stats.xem');
go

alter event session session_waits on server state= start;
go
&lt;/pre>
&lt;p></code></p>


<p>
  With the Extended Events session created and started, you can now run the query or procedure you want to analyze. After that, stop the Extended Event session and look at the captured data:
</p>


<pre><code class="prettyprint lang-sql">
alter event session session_waits on server state= stop;
go

with x as (
select cast(event_data as xml) as xevent
from sys.fn_xe_file_target_read_file
      ('c:\temp\wait_stats*.xel', 'c:\temp\wait_stats*.xem', null, null))
select * from x;
go
&lt;/pre>
&lt;p></code><br />
<a href="http://rusanu.com/wp-content/uploads/2014/01/session_waits_raw.png"><img src="http://rusanu.com/wp-content/uploads/2014/01/session_waits_raw.png" alt="" title="session_waits_raw" width="600" class="alignleft size-full wp-image-2228" /></a></p>


<p>
  As you can see the Extended Events session captured each individual wait, each incident when the session was suspended. If we click on an individual captured event we see the event data XML:
</p>


<pre><code class="prettyprint lang-xml">
&lt;event name="wait_info" package="sqlos" timestamp="2014-01-14T12:08:14.388Z"&gt;
  &lt;data name="wait_type"&gt;
    &lt;value&gt;8&lt;/value&gt;
    &lt;text&gt;LCK_M_IX&lt;/text&gt;
  &lt;/data&gt;
  &lt;data name="opcode"&gt;
    &lt;value&gt;1&lt;/value&gt;
    &lt;text&gt;End&lt;/text&gt;
  &lt;/data&gt;
  &lt;data name="duration"&gt;
    &lt;value&gt;4&lt;/value&gt;
  &lt;/data&gt;
  &lt;data name="signal_duration"&gt;
    &lt;value&gt;2&lt;/value&gt;
  &lt;/data&gt;
&lt;/event&gt;
</code></pre>


<p>
  This is a 4 millisecond wait for an IX mode lock. Another 2 milliseconds had spent until the task resumed, after the lock was granted. You can shred out the XML into columns for a better view:
</p>


<pre><code class="prettyprint lang-sql">
with x as (
select cast(event_data as xml) as xevent
from sys.fn_xe_file_target_read_file
      ('c:\temp\wait_stats*.xel', 'c:\temp\wait_stats*.xem', null, null))
select xevent.value(N'(/event/data[@name="wait_type"]/text)[1]', 'sysname') as wait_type,
	xevent.value(N'(/event/data[@name="duration"]/value)[1]', 'int') as duration,
	xevent.value(N'(/event/data[@name="signal_duration"]/value)[1]', 'int') as signal_duration
 from x;
&lt;/pre>
&lt;p></code></p>


<p>
  Finally, we can aggregate all the data in the captured Extended Events session:
</p>


<pre><code class="prettyprint lang-sql">
with x as (
select cast(event_data as xml) as xevent
from sys.fn_xe_file_target_read_file
      ('c:\temp\wait_stats*.xel', 'c:\temp\wait_stats*.xem', null, null)),
s as (select xevent.value(N'(/event/data[@name="wait_type"]/text)[1]', 'sysname') as wait_type,
	xevent.value(N'(/event/data[@name="duration"]/value)[1]', 'int') as duration,
	xevent.value(N'(/event/data[@name="signal_duration"]/value)[1]', 'int') as signal_duration
 from x)
 select wait_type, 
	count(*) as count_waits, 
	sum(duration) as total__duration,
	sum(signal_duration) as total_signal_duration,
	max(duration) as max_duration,
	max(signal_duration) as max_signal_duration
from s
group by wait_type
order by sum(duration) desc;
</code></pre>


<p>
  <a href="http://rusanu.com/wp-content/uploads/2014/01/session_waits_agg.png"><img src="http://rusanu.com/wp-content/uploads/2014/01/session_waits_agg.png" alt="" title="session_waits_agg" width="695" height="177" class="alignleft size-full wp-image-2229" /></a>
</p>


<p>
  This is an exceptional wealth of information about what happened during the execution of my request. 742 times it waited on the log commit to be written to disk for a total of ~14 seconds wait time, 12 times it waited for a lock, 2 times for a page latch etc. If your curious, my 'workload' I captured was a single row INSERT in a loop. 
</p>


<h2>
  Analyzing Execution Plans
</h2>


<p>
  SQL Server can expose the actual execution plan for queries. The actual query tree gets serialized as XML and there are several ways to obtain it:
</p>


<ul>
  <li>
    <a href="http://technet.microsoft.com/en-us/library/ms187757.aspx"><tt>SET SHOWPLAN_XML ON;</tt></a> in the session you execute your query.
  </li>
  
  
  <li>
    Use the SQL Profiler, see <a href="http://technet.microsoft.com/en-us/library/ms190233.aspx">Displaying Execution Plans by Using SQL Server Profiler Event Classes</a>.
  </li>
  
  
  <li>
    Enable the "Include Actual Execution Plan" in SSMS, see <a href="http://technet.microsoft.com/en-us/library/ms178071.aspx">Displaying Graphical Execution Plans</a>.
  </li>
  
  
  <li>
    Use <a href="http://technet.microsoft.com/en-us/library/ms189747.aspx"><tt>sys.dm_exec_query_plan</tt></a> function to obtain the plan XML from a plan handle. The plan handle can be obtained from DMVs like <tt>sys.dm_exec_requests</tt> (currently executing queries) or <tt>sys.dm_exec_query_stats</tt> (past executed queries).
  </li>
  
</ul>


<p class="callout float-right">
  If you don't know anything about execution plans then focus on <i>thick arrows</i>
</p>


<p>
  I will not enter into much detail into how to analyze a query plan because the subject is way too vast for this article. Execution plans capture what the query <i>did</i>, namely what operators it executed, how many times was each operator invoked, how much data did each operator process and how much data it produced in turn, and how much actual time it took for each operation. Unlike the wait times analysis, analyzing the execution focuses on <i>runtime</i> and where was the actual CPU time consumed on. The graphical display of SSMS simplifies the information in an execution plan XML to make it more easily consumable and the graphical representation makes easy to stop the bottlenecks. Thick arrows represent large data transfers between operators, meaning that operators had to look at a lot of rows to produce their output. Needless to say, the more data an operator needs to process, the more time it takes. Hovering over operators display a popup info note with details about the operator. A large <b>actual row count</b> shows large amount of data being processed. The popup include also the <b>estimated row count</b> which is how much data SQL Server had predicted the operator will process, based on the table statistics. A wild discrepancy between actual and estimated often is responsible for bad performance and can be tracked to outdated statistics on the table.
</p>


<p>
  Another are of interest in the execution plan are the operators themselves. Certain operators are inherently expensive, for example sort operators. An expensive operator coupled with a large data input is usually the cause of a huge execution time. Even if you do no know which operators are expensive and which are not, the execution plan contains the cost associated with each operator and you can use this information to get an idea for where problems may be.
</p>


<h1>
  <a name="identify_problem_query"></a>Identifying problem queries
</h1>


<p>
  SQL Server exposes the runtime execution statistics for most queries it had run in <a href="http://msdn.microsoft.com/en-us/library/ms189741.aspx"><tt>sys.dm_exec_query_stats</tt></a>. This DMV shows the number of times a query had run, total/max/min/last active CPU time ('worker time'), total/max/min/last elapsed wall-clock time, total/max/min/last logical reads and logical writes, total/max/min/last number of rows returned. This kind of information is a real treasure cove to identify problems in an application:
</p>


<dl>
  <dt>
    Large execution count
  </dt>
  
  
  <dd>
    Queries that are run frequently are the most sensitive for performance. Even if they perform acceptable, if they are run often then small improvements in each individual execution can yield overall significant speed. But even better is to look, with a critical eye, into <i>why</i> is the query executed often. Application changes can reduce the number of execution count.
  </dd>
  
  
  <dt>
    Large logical reads
  </dt>
  
  
  <dd>
    A query that has to scan large amounts of data is going to be a slow query, there is no question about this. The focus should be in reducing the amount of data scanned, typically the problem is a missing index. A 'bad' query plan can also be a problem. A high number of logical reads is accompanied by a high worker time but the issue is not high CPU usage, but the focus should be on large logical reads.
  </dd>
  
  
  <dt>
    Large physical reads
  </dt>
  
  
  <dd>
    This is basically the same problem as large logical reads, but it also indicates that your server does not have sufficient RAM. Luckily this is, by far, the easiest problem to solve: just buy more RAM, max out the server RAM slots. You're still going to have a large logical reads, and you should address that problem too, but addressing an application design issue is seldom as easy as ordering 1TB of RAM...
  </dd>
  
  
  <dt>
    High worker time with low logical reads
  </dt>
  
  
  <dd>
    This case is not very common and indicates an operation that is very CPU intensive even with little data to process. String processing and XML processing are typical culprits and you should look at moving the CPU burn toward the periphery, in other words perform the CPU intensive operations on the client, not on the server.
  </dd>
  
  
  <dt>
    High elapsed time with log worker time
  </dt>
  
  
  <dd>
    A query that takes a long wall-clock time but utilizes few CPU cycles is indicative of blocking. This is a query that was suspended most of the time, waiting on something. <a href="#wait_info_current">Wait analysis</a> should indicate the contention cause, the resource being waited for.
  </dd>
  
  
  <dt>
    High total rows count
  </dt>
  
  
  <dd>
    Large result sets requested by the application can be justified in an extremely low number of cases. This problem must be addressed in the application design.
  </dd>
  
</dl>


<p>
  Finding a problem query using <tt>sys.dm_exec_query_stats</tt> becomes a simply exercise of ordering by the desired criteria (by execution count to find frequently run queries, by logical reads, by elapsed time etc). Tho get the text of the query in question use <a href="http://technet.microsoft.com/en-us/library/ms181929.aspx"><tt>sys.dm_exec_sql_text</tt></a>, and to get the query execution plan use <a href="http://technet.microsoft.com/en-us/library/ms189747.aspx"><tt>sys.dm_exec_query_plan</tt></a>:
</p>


<pre><code>
select st.text, 
	pl.query_plan,
	qs.*
	from sys.dm_exec_query_stats qs
	cross apply sys.dm_exec_sql_text(qs.sql_handle) as st
	cross apply sys.dm_exec_query_plan(qs.plan_handle) as pl;
</code></pre>


<p>
  If you do not know what kind of problem query to look for to start with, my advice is to focus in this order:
</p>


<ol>
  <li>
    High execution count. Identifying what queries are run most often is in my opinion more important that identifying which queries are particularly slow. More often than not the queries found to be run most often are a surprise and simply limiting the execution count yield significant performance improvements.
  </li>
  
  
  <li>
    High logical reads. Large data scans are the usual culprit for most performance problems. They may be caused by missing indices, by a poorly designed  data model, by 'bad' plans caused by outdated statistics or parameter sniffing and other causes.
  </li>
  
  
  <li>
    High elapsed time with low worker time. Blocking queries do not tax the server much, but your application users do not care if the time they waited in front of the screen for a refresh was spent active or blocked.
  </li>
  
</ol>


<h1>
  <a name="missing_indexes"></a>Missing Indexes
</h1>


<p>
  Repeatedly I said that performance problems are caused by 'missing indexes', and SQL Server has a feature aptly named <a href="http://technet.microsoft.com/en-us/library/ms345524.aspx">Missing Indexes</a>:
</p>


<blockquote>
  <p>
    The missing indexes feature uses dynamic management objects and Showplan to provide information about missing indexes that could enhance SQL Server query performance.
  </p>
</blockquote>


<p>
  Personally I seldom focus my analysis on the built in missing index feature. To start with the feature has <a href="http://technet.microsoft.com/en-us/library/ms345485(v=sql.105).aspx">known limitations</a>, but more importantly is that the feature is starting with the solution, w/o first searching for a cause. I always start with identifying problem queries first, searching for ones which would yield the highest bang for the buck. Once a problem query is identified as causing large scans, the missing index DMVs can be consulted to identify a candidate. But you should always take the recommendations with a grain of salt. The missing index feature does not know what the query is doing from an application business logic point of view. Reviewing the query can potentially find a better solution than what the automated missing index feature may recommend.
</p>


<p>
  Experienced administrators have in place automation that periodically inspects the missing index DMVs and find potential hot spots and problems. This is a very good proactive approach. But in most cases the performance problems are hot and immediate, and finding them using wait analysis or query execution stats is a more direct process. For a more in depth discussion of missing indexes DMVs I recommend <a href="http://www.brentozar.com/blitzindex/"><tt>sp_BlitzIndex</tt></a>.
</p>


<h1>
  <a name="performance_counters" />SQL Server Performance Counters
</h1>


<p>
  Performance counters offer a different perspective into SQL Server performance than the wait analysis and execution DMVs do. The biggest advantage of performance counters is that they are extremely cheap to collect and store so they lend themselves as the natural choice for long term monitoring. Looking at current performance counter numbers can reveal potential problems, but interpreting the values is subject to experience and there are a lot of lore and myths on certain magical values. For a analyzing performance problems on a server that you have access and is misbehaving <i>right now</i>, a more direct approach using wait analysis and execution stats is my recommended approach. Where performance counters shine is comparing current behavior with past behavior. for which, obviously, you need to have performance counters values collected and saved from said past. These should be part of an established baseline, which has captured the server behavior under workload when the server was behaving correctly and was not experiencing problems. If you do no have a baseline, consider establishing one.
</p>