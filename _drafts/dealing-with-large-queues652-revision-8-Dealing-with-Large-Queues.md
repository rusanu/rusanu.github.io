---
id: 660
title: Dealing with Large Queues
date: 2010-03-02T19:55:54+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/02/652-revision-8/
permalink: /2010/03/02/652-revision-8/
---
On a project I&#8217;m currently involved with we have to handle a constant influx of audit messages for processing. The messages come from about 50 SQL Express instances located in data centers around the globe, delivered via Service Broker into a processing queue hosted on a mirrored database where an activated procedure shreds the message payload into relational tables. These tables are in turn replicated with transactional replication into a data warehouse database. after that the messages are deleted from the processing servers, as replication is set up not to replicate deletes. The system must handle a constant average rate of about 200 messages per second, 24&#215;7, with spikes going up to 2000-3000 messages per second over periods of minutes to an hour.

When dealing with these relatively high volumes, it is inevitable that queues will grow during the 2000 msgs/sec spikes and drain back to empty when the incoming rate stabilizes again at the normal 200 msgs/sec rate. Service Broker does an excellent job at handling these non-connectivity periods, retains the audit messages and quickly delivers them when connectivity is restored.

What I noticed though is that sometimes the processing of the received messages could hit a threshold from where it could not recover. The queue processing would slow down to a rate that was bellow the incoming rate, and from that point forward the queue could just grow. I want to detail a bit the reason why this can happen and what I did to alleviate the problem.

## Index Fragmentation

Every DBA knows about fragmentation. All database developers also understand fragmentation and how to avoid it. So we can skip ahead and &#8230; wait. Actually, what _is_ index fragmentation? Lets go back to the whitepaper <a href="http://technet.microsoft.com/en-us/library/cc966523.aspx#EHAA" target="_blank">Microsoft SQL Server 2000 Index Defragmentation Best Practices</a>. Even though the whitepaper is for SQL 2000, it was recently updated on March 2009 and is the most detailed whitepaper dealing with index fragmentation released by Microsoft I know of:
:   Fragmentation

Fragmentation exists when indexes have pages in which the logical ordering, based on the key value, does not match the physical ordering inside the data file.

Say you have an index with 9 rows, with the keys A, B, C, D, E, F, G, H and J. For our example, each page can fit 3 rows, and the database pages are in order in the database file: P1, P2 and P3. If the rows are (A,B,C) on P1, then (E,F,G) on P2 and (G, H, J) on P3 then the index is unfragmented. But if the row are (E,F,G) on P1 then (G,H,J) on P2 and (A,B,C) on P3 the page P3 is first in the key order, but last in the physical file order.

Index fragmentation affects the read performance when fetching pages into the buffer pool. This is because the Read-Ahead Manager issues read-ahead requests in contiguous fragments. When the index is fragmented the read-ahead manager will issue a large number of small read-aheads. When the index is contiguous the Read-Ahead Manager will issue a small number of large read-aheads. On traditional disk-head and spindle drives, the large number of small read requests caused by fragmentation results in drastic read throughput reduction. A large number of IO requests carries also a bigger user-to-kernel context switch baggage. Again, this is explained int he whitepaper I linked above:

<