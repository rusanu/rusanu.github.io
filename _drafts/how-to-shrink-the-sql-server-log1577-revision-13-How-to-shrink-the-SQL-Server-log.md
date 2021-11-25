---
id: 1599
title: How to shrink the SQL Server log
date: 2012-07-26T01:16:14+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/07/26/1577-revision-13/
permalink: /2012/07/26/1577-revision-13/
---
> I noticed that my database log file has grown to 200Gb. I tried to shrink it but is still 200Gb. How can I shrink the log and reduce the file size?

The problem is that even after you discover about <tt>DBCC SHRINKFILE</tt> and attempt to reduce the log size, the command seems not to work at all and leaves the log at the same size as before. What is happening?

If you look back at [What is an LSN: Log Sequence Number](http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/) you will see that LSNs are basically pointers (offsets) inside the log file. There is one level of indirection (the VLF sequence number) and then the rest of the LSN is basically an offset inside the Virtual Log File (the VLF). The log is always defined by the two LSNs: the head of the log (where new log records will be placed) and the tail of the log (what is the oldes log record of interest). Generating log activity (ie. any updates in the database) advance the head of the log LSN number. The tail of the log advances when the database log is being backed up (this is a simplification, more on it later).

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-1.png" alt="" title="Truncate log-1" width="600" class="aligncenter size-full wp-image-1594" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-1.png)

The image above shows as typical log file layout. The head of the log moves ahead as transactions generate log records, occupiying more of the free space ahead in the current VLF. If the current VLF fills up, a new empty VLF can be used. If there are no empty VLFs the system must grow the log LDF file and create a new VLF to allow for more log records to be written. This is when the physical LDF file actually increases and takes more space on disk:

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-2.png" alt="" title="Truncate log-2" width="600" class="aligncenter size-full wp-image-1597" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-2.png)

<p class="callout float-right">
  Advancing the head of the log can cause LDF file growth when no free VLF are available
</p>

In the image above some more transaction activity resulted in head of the log advancing forward. As the VLF2 filled up, the system had to allocate a new VLF bu growing the physical log LDF file. The unused space in VLF1 cannot be used as long as there is even a single active LSN record in it, the [What is an LSN: Log Sequence Number](http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/) article explains why this 

Advancing the tail of the log is just a tad more complex simply because the tail is not a precise LSN: is the lowest LSN of _any_ consumer that needs to look at the log records. Such consumers are:

Active Transactions
:   Any transaction that has not yet committed needs to retain all the log from when the transaction started, in case it has to rollback. Remember that rollback is implemented by reading back all the transaction log record and generating a compensating (undo) action for each.

Log Backup
:   Log backup needs to retain all the log generated until the next log backup runs, so it is copied into the backup file.

Transactional Replication
:   The transactional replication publisher agent has to read the log to identify changes that occurred and have to be distributed to the subscribers.

Database Mirroring, AlwaysOn
:   Both these technologies work by literally copying the log to the partners, so they require the log to be retained until is copied over.

Other
:   If you check the <tt>log_reuse_wait_desc</tt> column documentation in <a href="http://msdn.microsoft.com/en-us/library/ms345414.aspx" target="_blank">Factors That Can Delay Log Truncation</a> you will see all the other processes that can retain the log tail, things like CHECKPOINT, an active backup operation, a database snapshot being created, a log scan (<tt>fn_dblog()</tt>) etc. All these will cause the tail of the log to stay in place (not progress forward).

<p class="callout float-right">
  Log truncation occurs when the tail of the log LSN advances forward
</p>

The tail of the log will be the _lowest_ LSN from any of the LSNs required by the processes above. Whenever the tail of the log advances forward the log is said to be _truncated_, as described in <a href="http://msdn.microsoft.com/en-us/library/ms189085.aspx" target="_blank">Transaction Log Truncation</a>. Although the process is described as a &#8216;log record deletion&#8217; what really happens is nothing but the tail of the log LSN is advanced forward (the LSN is increased). Since we&#8217;ve seen that LSNs are nothing but offsets inside the log file, advancing the LSN means the offset of the tail of the log advances in the log file. When this occurs, the log file _behind_ the offset where the current tail of the LSN points to is no longer required and can be reclaimed. Since LSNs are ever increasing it would follow from above that the LOG file would always grow ad infinity, since an ever increasing pair of offsets (the tail and the head) would define the active portion of the log. Here is where the VLF indirection mentioned before comes into play: when an entire VLF is inactive we can change the VLF <tt>sequence_number</tt> and then we can reuse the VLF to store log records with _higher_ LSNs than before.