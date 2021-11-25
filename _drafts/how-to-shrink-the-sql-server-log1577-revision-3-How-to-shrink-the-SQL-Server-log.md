---
id: 1587
title: How to shrink the SQL Server log
date: 2012-06-08T02:35:36+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/06/08/1577-revision-3/
permalink: /2012/06/08/1577-revision-3/
---
<blockquoteI noticed that my database log file has grown to 200Gb. I tried to shrink it but is still 200Gb. How can I shrink the log and reduce the file size?</blockquote> 

The problem is that even after you discover about <tt>DBCC SHRINKFILE</tt> and attempt to reduce the log size, the command seems not to work at all and leaves the log at the same size as before. What is happening?

If you look back at [What is an LSN: Log Sequence Number](http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/) you will see that LSNs are basically pointers (offsets) inside the log file. There is one level of indirection (the VLF sequence number) and then the rest of the LSN is basically an offset inside the VLF. The log is always defined by the two LSNs: the head of the log (where new log records will be placed) and the tail of the log (what is the oldes log record of interest). Generating log activity (ie. any updates in the database) advance the head of the log LSN number. Advancing the tail of the log is just a tad more complex simply because the tail is not a precise LSN: is the lowest LSN of _any_ consumer that needs to look at the log records. Such consumers are:

Active Transactions
:   Any transaction that has not yet committed needs to retain all the log from when the transaction started, in case it has to rollback. Remember that rollback is implemented by reading back all the transaction log record and generating a compensating (undo) action for each.

Log Backup
:   Log backup needs to retain all the log generated until the next log backup runs, so it is copied into the backup file.

Transactional Replication
:   The transactional replication publisher agent has to read the log to identify changes that occurred and have to be distributed to the subscribers.

Database Mirroring, AlwaysOn
:   Both these technologies work by literally copying the log to the partners, so they require the log to be retained until is copied over.

Other
:   If you check the <tt>log_reuse_wait_desc</tt> column documentation in [MSDN](http://msdn.microsoft.com/en-us/library/ms178534.aspx) you will see all the other processes that can retain the log tail, things like CHECKPOINT, an active backup operation, a database snapshot being created, a log scan (<tt>fn_dblog</tt> </dd. </dl>