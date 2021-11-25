---
id: 657
title: Dealing with Large Queues
date: 2010-03-02T16:56:11+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/02/652-revision-5/
permalink: /2010/03/02/652-revision-5/
---
On a project I&#8217;m currently involved with we have to handle a constant influx of audit messages for processing. The messages come from about 50 SQL Express instances located in data centers around the globe, delivered via Service Broker into a processing queue hosted on a mirorred database where an activated procedure shreds the message payload into relational tables. These tables are in turn replicated with transactional replication into a data warehouse database. after that the messages are deleted from the processing servers, as replication is set up not to replicate deletes. The system must handle a constant average rate of about 200 messages per second, 24&#215;7, with spikes going up to 2000-3000 messages per second over periods of minutes to an hour.

When dealing with these relatively high volumes, it is inevitable that queues will grow during the 2000 msgs/sec spikes and drain back to empty when the incomming rate stabilizez again at the normal 200 msgs/sec rate. As a note, these spikes occur mostly due to connectivity problems that occur wehn the SQL Express hosting machines undergo some administrative change and something is misconfigured, usually an IPSec policy&#8230; Service Broker does an excelent job at handling these non-conectivity periods, retains the audit messages and quickly deliveres them when connectivity is restored.

What I noticed though is that sometimes the processing of the received messages could hit a treshhold from where it could not recover. The queue processing would slow down to a rate that was bellow the incomming rate, and from that point forward the queue qould just grow. I want to detail a bit the reason why this can happen and what I did to alleviate the problem.

## Index Fragmentation

Every DBA knows about fragmentation. All database developers also understand fragmentation and how to avoid it. So we can skip ahead and &#8230; wait. Actually, what _is_ index fragmentation? Lets go back to the whitepaper <a href="http://technet.microsoft.com/en-us/library/cc966523.aspx#EHAA" target="_blank">Microsoft SQL Server 2000 Index Defragmentation Best Practices</a>:
:   Fragmentation

Fragmentation exists when indexes have pages in which the logical ordering, based on the key value, does not match the physical ordering inside the data file.

Even though the whitepaper is for SQL 2000, it was recently updated on March 2009 and is the most detailed whitepaper dealing with index fragmentation released by Microsoft I know of.