---
id: 1883
title: Understanding how SQL Server executes a query
date: 2013-06-14T03:55:15+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/06/14/1871-revision-12/
permalink: /2013/06/14/1871-revision-12/
---
This article is an introduction into how SQL Server works. My target audience are the developers and I hope this article will help you write better database code and will help you get starting when having to investigate performance problems in the database back end.

## Requests

SQL Server is a client-server platform. The only way to interact with the back-end database is by sending requests that contain commands for the database. The protocol used to communicate between your application and the database is called TDS (Tabular Data Sream) and is described on MSDN in the Technical Document <a href="http://msdn.microsoft.com/en-us/library/dd304523.aspx" target="_blank">[MS-TDS]: Tabular Data Stream Protocol</a>. thr application can use one of the several client-side implementations of the protocol: the CLR managed SqlClient, OleDB, ODBC, JDBC, PHP Driver for SQL Server or the open source FreeTDS implementation. The gist of it is that when your application whats the database to do _anything_ it will send a request over the TDS protocol. The request itself can take several forms:

<!-- more -->

<a href="http://msdn.microsoft.com/en-us/library/dd357447.aspx" target="_blank">Batch Request</a> 
:   This request type contains just T-SQL text for a batch to be executed. This type of requests do not have parameters, but obviously the T-SQL batch itself wi can contain local variables declarations. This is the type of request SqlClient sends if you invoke any of the <a href="http://msdn.microsoft.com/en-us/library/hhxxcw8c.aspx" target="_blank"><tt>SqlCommand.ExecuteReader()</tt></a>/.<a href="http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand.executenonquery.aspx" target="_blank"><tt>ExecuteNonQuery()</tt></a>/.<a href="http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand.executescalar.aspx" target="_blank"><tt>ExecuteScalar()</tt></a>/.<a href="http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand.executexmlreader.aspx" target="_blank"><tt>ExecuteXmlReader()</tt></a> (or they respective asyncronous equivalents) on a SqlCommand object with an empty <a href="http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand.parameters.aspx" target="_blank"><tt>Parameters</tt></a> list. If you monitor with SQL Profiler you will see an <a href="http://msdn.microsoft.com/en-us/library/ms190441.aspx" target="_blank">SQL:BatchStarting Event Class</a></dt> 
    
    <a href="http://msdn.microsoft.com/en-us/library/dd303353.aspx" target="_blank">Remote Procedure Call Request</a> 
    
    :   This request type contains a <a href="http://msdn.microsoft.com/en-us/library/dd357576.aspx" target="_blank">procedure identifier</a> to execute, along with any number of parameters. Of special interest is when the procedure id will be 12, ie. <a href="http://msdn.microsoft.com/en-us/library/ff848746.aspx" target="_blank"><tt>sp_execute</tt></a>. In this case the first parameter is the T-SQL text to execute, and this is the request that what your application will send if you execute a SqlCommand object with a non-empty <tt>Parameters</tt> list. If you monitor using SQL Profiler you will see an <a href="http://msdn.microsoft.com/en-us/library/dd357576.aspx" target="_blank">RPC:Starting Event Class</a>.
    
    <a href="http://msdn.microsoft.com/en-us/library/dd357606.aspx" target="_blank">Bulk Load Request</a> 
    
    :   Bulk Load is a special type request used by bulk insert operations, like the <a href="http://msdn.microsoft.com/en-us/library/ms162802.aspx" target="_blank">bcp.exe utility</a>, the <a href="http://msdn.microsoft.com/en-us/library/ms131708.aspx" target="_blank"><tt>IRowsetFastLoad</tt></a> OleDB interface or by the <a href="http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlbulkcopy.aspx" target="_blank"><tt>SqlBulkcopy</tt></a> managed class. Bulk Load is different from the other requests because is the only request that starts execution before the request is complete on the TDS protocol. this allows it to start executing and then start consuming the <a href="http://msdn.microsoft.com/en-us/library/dd358082.aspx" target="_blank">stream</a> of data to insert.
    After a complete TDS request reaches the database engine SQL Server will create a task to handle the request.
    
    ## Tasks
    
    The task will represent the request from beginning till completion. For example if the request is a SQL Batch type request the task will represent the entire batch, not individual statements. Individual statements inside the SQL Batch will not create new tasks. Certain individual statements inside the batch may execute with parallelism (often referred to as DOP, Degree Of Parallelism) and in their case the task will spawn new sub-tasks for executing in parallel. If the request returns a result the batch is complete when the result is completely consumed by the client (eg. when you dispose the <a href="http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqldatareader.aspx" target="_blank"><tt>SqlDataReader</tt></a>). You can see the list of tasks in the server by querying <a href="http://msdn.microsoft.com/en-us/library/ms174963.aspx" target="_blank"><tt>sys.dm_os_tasks</tt></a>.
    
    When a new request reaches the server and the task is created to handle that request the task will be first PENDING, meaning the task did not yet start to execute. At this stage the server has no idea yet what the request actually is. The task has to start executing first, and for this the engine must assign a worker to it.
    
    ## Workers
    
    Workers are the thread pool of SQL Server. A number of workers is created initially at server start up and more can be created on-demand up to the configured <a href="http://msdn.microsoft.com/en-us/library/ms187024(v=sql.105).aspx" target="_blank"><tt>max worker threads</tt></a>. Only workers execute code. Workers are waiting for PENDING tasks to become available (from requests coming into the server) and then each worker takes exactly one task and executes it. The worker is busy (occupied) until the task finishes completely. Tasks that are PENDING when there are no more available workers will have to wait until one of the executing (running) task completes and the worker that executed that task becomes available to execute another pending task. For a SQL batch request the worker that picks up that task will execute the entire SQL batch (every statement). This should settle the often asked question whether statements in a SQL batch (=> request => task => worker) can execute in parallel: no, as they are executed on a single thread (=> worker) then each statement must complete before the next one starts. For statements that internally use parallelism (DOP > 1) and create sub-tasks, each sub-task goes through exactly the same cycle: it is created as PENDING and a worker must pick it up and execute it (a different worker from the SQL batch worker, that is by definition occupied!). the lists and state of workers inside SQL Server can be seen by querying <a href="http://msdn.microsoft.com/en-us/library/ms178626.aspx" target="_blank"><tt>sys.dm_os_workers</tt></a>.
    
    Before we proceed to understand how the task will execute the request is important to detour to explain how the workers interact and cooperate inside SQL Server:
    
    ## Cooperative Multitasking
    
    Unlike threads, SQL server workers are not preempted in execution in order to offer concurrency and CPU sharing. Instead each worker must voluntarily yield the CPU so that other workers can execute. The CPUs are represented in SQL Server as &#8216;schedulers&#8217; and the state of each is visible in the <a href="http://msdn.microsoft.com/en-us/library/ms177526.aspx" target="_blank"><tt>sys.dm_os_schedulers</tt></a>