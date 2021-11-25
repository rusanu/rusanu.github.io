---
id: 710
title: Dealing with Large Queues
date: 2010-03-02T22:51:23+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/02/652-revision-30/
permalink: /2010/03/02/652-revision-30/
---
On a project I&#8217;m currently involved with we have to handle a constant influx of audit messages for processing. The messages come from about 50 SQL Express instances located in data centers around the globe, delivered via Service Broker into a processing queue hosted on a mirrored database where an activated procedure shreds the message payload into relational tables. These tables are in turn replicated with transactional replication into a data warehouse database. after that the messages are deleted from the processing servers, as replication is set up not to replicate deletes. The system must handle a constant average rate of about 200 messages per second, 24&#215;7, with spikes going up to 2000-3000 messages per second over periods of minutes to an hour.

When dealing with these relatively high volumes, it is inevitable that queues will grow during the 2000 msgs/sec spikes and drain back to empty when the incoming rate stabilizes again at the normal 200 msgs/sec rate. Service Broker does an excellent job at handling these non-connectivity periods, retains the audit messages and quickly delivers them when connectivity is restored.

What I noticed though is that sometimes the processing of the received messages could hit a threshold from where it could not recover. The queue processing would slow down to a rate that was bellow the incoming rate, and from that point forward the queue could just grow. I want to detail a bit the reason why this can happen and what I did to alleviate the problem.

## Index Fragmentation

Every DBA knows about fragmentation. All database developers also understand fragmentation and how to avoid it. So we can skip ahead and &#8230; wait. Actually, what _is_ index fragmentation? Lets go back to the whitepaper <a href="http://technet.microsoft.com/en-us/library/cc966523.aspx#EHAA" target="_blank">Microsoft SQL Server 2000 Index Defragmentation Best Practices</a>. Even though the whitepaper is for SQL 2000, it was recently updated on March 2009 and is the most detailed whitepaper dealing with index fragmentation released by Microsoft I know of:
:   Fragmentation

Fragmentation exists when indexes have pages in which the logical ordering, based on the key value, does not match the physical ordering inside the data file.

Say you have an index with 9 rows, with the keys A, B, C, D, E, F, G, H and J. For our example, each page can fit 3 rows, and the database pages are in order in the database file: P1, P2 and P3. If the rows are (A,B,C) on P1, then (E,F,G) on P2 and (G, H, J) on P3 then the index is unfragmented. But if the row are (E,F,G) on P1 then (G,H,J) on P2 and (A,B,C) on P3 the page P3 is first in the key order, but last in the physical file order.

Index fragmentation affects the read performance when fetching pages into the buffer pool. This is because the Read-Ahead Manager issues read-ahead requests in contiguous fragments. When the index is fragmented the read-ahead manager will issue a large number of small read-aheads. When the index is contiguous the Read-Ahead Manager will issue a small number of large read-aheads. On traditional disk-head and spindle drives, the large number of small read requests caused by fragmentation results in drastic read throughput reduction. A large number of IO requests carries also a bigger user-to-kernel context switch baggage. Again, this is explained in the whitepaper I linked above:

<blockquote cite="http://technet.microsoft.com/en-us/library/cc966523.aspx">
  <p>
    <i><br /> To understand why fragmentation had such an effect on the DSS workload performance, it is important to understand how fragmentation affects the SQL Server read-ahead manager. For queries that scan one or more indexes, the SQL Server read-ahead manager is responsible for scanning ahead through the index pages and bringing additional data pages into the SQL Server data cache. The read-ahead manager dynamically adjusts the size of reads it performs based on the physical ordering of the underlying pages. When there is low fragmentation, the read-ahead manager can read larger blocks of data at a time, more efficiently using the I/O subsystem. As the data becomes fragmented, the read-ahead manager must read smaller blocks of data. The amount of read-aheads that can be issued is independent of the physical ordering of the data; however, smaller read requests take more CPU resources per block, resulting in less overall disk throughput.</i>
  </p>
</blockquote>

### What causes index fragmentation?

Fragmentation occurs either when the order of physical operations does not match the logical order of rows (eg. insert rows into a table in reverse order of the clustered index key) or when frequent update operations occur after the index is constructed: rows are deleted and new rows are inserted. One case is particularly aggravating: the insert page split, because the page split not only increases fragmentation, it also reduces the page fill factor of the index, further damaging performance.

### Are Queues Fragmented?

Service Broker queues are backed by internal tables, and these tables have a clustered index on (status, conversation\_group\_id, priority, conversation\_handle, queueing\_order). Messages are constantly enqueued and dequeued into and from this internal table. The queueing operations are nothing else than inserts and deletes, and according to what we just discussed about how fragmentation occurs, they should get high index fragmentation.

However, there is one crucial difference between how data tables are used, in comparison with queues: Queues _drain_, meaning they are always near 0 records count. When they grow during spikes, the processing must be able to drain them back to 0, even as new messages are enqueued at normal rates. A table with no rows has no pages, so there is no fragmentation. So the expected behavior of queues is to hover around a low record count, where fragmentation has no performance impact, grow on spikes, get fragmented, but drain back to 0 and thus repairing themselves &#8216;on the fly&#8217;.

Or at least that&#8217;s how the theory goes&#8230;

## Ghost Cleanup

SQL Server DELETE operations do no remove rows from indexes. Instead, the rows are simply marked as &#8216;ghosted&#8217; and left in the page. It is the job of a dedicated task, namely the Ghost Cleanup task, to reclaim the space these rows occupy. More importantly, it is the Ghost Cleanup task&#8217;s job to deallocate any page that no longer contains any record and release these pages back to the database, as free pages. This is a performance improvement because DELETE operations can complete faster, and it also improves the performance of rollbacks. If you want to read more about how Ghost Cleanup operates, the best resource is Paul Randal&#8217;s article <a href="http://www.sqlskills.com/BLOGS/PAUL/post/Inside-the-Storage-Engine-Ghost-cleanup-in-depth.aspx" target="_blank">Inside the Storage Engine: Ghost cleanup in depth</a>.

As I said, dequeue operations are in fact DELETE from the internal tables that back the Service Broker queues. When the RECEIVE statement is run, in fact an DELETE with OUTPUT occurs on this internal table. And as such, all the returned messages are in fact ghosted records left in the internal tables. The Ghost Cleanup has to come about and collect them, to reclaim the space and eventually free the space.

The Ghost Cleanup is calibrated to operate on a normal data table environment. It wakes every 5 seconds, reclaims ghosted records in up to 10 pages, then goes to sleep again. In addition, SELECT statements that encounter ghosted records during index scans place the page into a list so that the Ghost Cleanup collects it on its next pass. From SQL Server testing and customer feedback, this calibration balances the need to cleanup pages with the overhead of running the ghost cleanup just fine for a normal data table.

The problem with Service Broker queues is that they are not normal data tables: they are queues! Every single row is inserted and then deleted as fast as possible. No record is ever read twice. No record stays around. The queues are constantly growing, and constantly shrinking. On machines that are entirely dedicated to Service Broker processing it is not unusual to have all CPU and all I/O resources of the server dedicated to enqueuing and dequeuing messages into one single queue. In other words INSERT then quickly DELETE one row at a time, from all CPU cores, as fast as the I/O subsystem permits it. The Ghost Cleanup better keep up, as _every_ enqueued message is deleted, and _every_ deleted row is a ghosted record to be cleanup up. All of the sudden 10 pages every 5 seconds seems a bit short changed, when the rate of newly created ghosted records is 200 per second!

## Crossing the Threshold

In my project we had an incident that causes a massive queue growth, to about 19 million messages. We expected the system to start draining as it usually did before, but it never did. It kept growing at a rate about 3 million a day, indicating that the processing could not keep up with the incoming rate of 200 msgs/sec. The processing was running as fast as possible, on a highly optimized procedure using the fastest set oriented message processing, similar to what I recommend in [Writing Service Broker Procedures](http://rusanu.com/2006/10/16/writing-service-broker-procedures/). After trying to speed up the IO system, moving the drives to fastest LUNs available in the attached SAN, the system could still no keep up. The disk metrics showed a _lot_ of read requests of 8192. bytes. On a SQL Server disk a large number of read requests of 8k size are a tell-tale of fragmentation: no multi-page read-ahead occurs, indicating that the average contiguous fragment length is 1 page. Under normal circumstances one would check <a href="http://msdn.microsoft.com/en-us/library/ms188917.aspx" target="_blank">sys.dm_db_index_physical_stats for fragmentation</a> but there is a small gotcha: this DMV does _not_ show stats for queues!

A second observation occurred sometimes later: even after the queue drained, the allocated space was not reclaimed by the ghost cleanup. In fact I&#8217;ve seen a queue having 0 rows, but over 1 million pages allocated. The <a href="http://msdn.microsoft.com/en-us/library/ms177426.aspx" target="_blank">Skipped Ghosted Records/sec</a> performance counter was showing over 200000 ghosted records skipped per second. It seems that the Ghost Cleanup was just unable to keep up with the nearly 200Gb size _empty_ queue. Even running [DBCC FORCEGHOSTCLEANUP](http://rusanu.com/2010/02/22/dbcccommand-enumeration/) could not improve the situation.

### Queue Maintenance

When a DBA is faced with a fragmented index, it has a simple avenue: rebuild or reorganize the index. ALTER INDEX &#8230; REORGANIZE or ALTER INDEX &#8230; REBUILD, and maybe do it online to avoid system downtime. In fact there are quite a few scripts provided by the community for just such a task, like <a href="http://twitter.com/sqlfool" target="_blank">Michelle Ufford&#8217;s</a> <a href="http://sqlfool.com/2009/06/index-defrag-script-v30/" target="_blank">Index Defrag Script</a>, and good DBAs always have their own, customized, flavor of index maintenance script in their tool belt.

But what about queues? There is no ALTER QUEUE &#8230; REBUILD nor ALTER QUEUE &#8230; REORGANIZE. How about the good ole&#8217; (and now deprecated) DBCC DBREINDEX and DBCC INDEXDEFRAG? Nope, they don&#8217;t works on queues. But queues are backed by hidden tables, right? You can always find out the hidden table that backs a queue:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">it&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">name &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">internal_table_name&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">q&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">name &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">queue_name&lt;br />
&lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Green">internal_tables &lt;/span>&lt;span style="color:Black">it&lt;br />
&lt;/span>&lt;span style="color:Gray">join &lt;/span>&lt;span style="color:Green">sys&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Green">service_queues &lt;/span>&lt;span style="color:Black">q &lt;/span>&lt;span style="color:Blue">on &lt;/span>&lt;span style="color:Black">it&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">parent_object_id &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">q&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Fuchsia">object_id&lt;br />
&lt;/span>
&lt;/pre>
&lt;p></code>

This query returns the name of the internal table that backs each Service Broker queue in the database. Can we do our maintenance operations on _them_? ALTER INDEX ALL ON queue\_messages\_1003150619 REORGANIZE. Nope, no luck. DBCC DBREINDEX(&#8216;queue\_messages\_1003150619&#8217;)? But if you check the schema on which the internal table resides, its actually **sys**. Could DBCC DBREINDEX(&#8216;sys.queue\_messages\_1003150619&#8217;) work? Nope.

At this point I must use the deus ex machina: I know some internals of SQL Server from my Service Broker FTE days. One such information is this: statements run from the <a href="http://msdn.microsoft.com/en-us/library/ms189595.aspx" target="_blank">Dedicated Administrative Connection</a> can have different binding rules. Is this the answer? **YES**:

**Service Broker queues can be reindexed by running DBCC DBREINDEX on the internal table that backs the queue from the DAC connection.** The internal table name must be prefixed with the **sys** schema name.

Fortunately this solution solved our problems. We&#8217;re running a job that, from a DAC connection, reindexes the internal table that backs the queue, even though the queue is empty. This operation reclaims the space consumed by millions of ghosted records back the the database:

<code class="prettyprint lang-sql">&lt;br />
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">dbcc &lt;/span>&lt;span style="color:Black">dbreindex&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'sys.queue_messages_1003150619'&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;/p>
&lt;p></code>

## Conclusion

Just like tables, queues _may_ require maintenance operations. But unlike tables, the DBA has no DDL at its disposal to do the job, except the DBCC DBREINDEX on the internal table, run from the DAC connection, which is a hack: a deprecated command, the DAC requirement, the internal table name digg from metadata&#8230; Hopefully, the problem will be addressed eventually and ALTER QUEUE &#8230; REINDEX and ALTER QUEUE &#8230; REORGANIZE will make it to the product.

Until then, should you be reindexing your queues every night? No. The conditions I encountered were caused by a very particular balance of server IO capacity, message incoming rate and triggered by a particularly high spike. But still, its good to know: if you ever **do** need to rebuild a queue because of high fragmentation or because of ghost cleanup &#8230; modesty&#8230; you can: find the name of the internal table behind the queue, open a DAC connection and run DBCC DBREINDEX.

I had this post on in standby for some time now, but since the SQLTuesday#4 topic is [IO, IO, It’s Off To Disk We Go!](http://www.straightpathsql.com/archives/2010/03/invitation-for-t-sql-tuesday-004-io/) I took the opportunity to finish it in time for the roundup.