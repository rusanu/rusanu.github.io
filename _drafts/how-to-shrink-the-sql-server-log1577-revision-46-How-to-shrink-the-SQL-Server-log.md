---
id: 1639
title: How to shrink the SQL Server log
date: 2012-07-26T23:50:52+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/07/26/1577-revision-46/
permalink: /2012/07/26/1577-revision-46/
---
> I noticed that my database log file has grown to 200Gb. I tried to shrink it but is still 200Gb. How can I shrink the log and reduce the file size?

The problem is that even after you discover about <tt>DBCC SHRINKFILE</tt> and attempt to reduce the log size, the command seems not to work at all and leaves the log at the same size as before. What is happening?

If you look back at [What is an LSN: Log Sequence Number](http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/) you will see that LSNs are basically pointers (offsets) inside the log file. There is one level of indirection (the VLF sequence number) and then the rest of the LSN is basically an offset inside the Virtual Log File (the VLF). The log is always defined by the two LSNs: the head of the log (where new log records will be placed) and the tail of the log (what is the oldest log record of interest). Generating log activity (ie. any updates in the database) advance the head of the log LSN number. The tail of the log advances when the database log is being backed up (this is a simplification, more on it later).

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-1.png" alt="" title="Truncate log-1" width="600" class="aligncenter size-full wp-image-1594" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-1.png)

<p class="callout float-right">
  Use <tt>DBCC LOGINFO</tt> to view the layout of the VFLs inside the log file
</p>

The image above shows as typical log file layout. To understand the layout of VLFs inside your LDF log file use <tt>DBCC LOGINFO</tt>. The head of the log moves ahead as transactions generate log records, occupying more of the free space ahead in the current VLF. If the current VLF fills up, a new empty VLF can be used. If there are no empty VLFs the system must grow the log LDF file and create a new VLF to allow for more log records to be written. This is when the physical LDF file actually increases and takes more space on disk:

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-2.png" alt="" title="Truncate log-2" width="600" class="aligncenter size-full wp-image-1597" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-2.png)

<p class="callout float-right">
  Advancing the head of the log can cause LDF file growth when no free VLFs are available
</p>

In the image above some more transaction activity resulted in head of the log advancing forward. As the VLF2 filled up, the system had to allocate a new VLF by growing the physical log LDF file. The unused space in VLF1 cannot be used as long as there is even a single active LSN record in it, the [What is an LSN: Log Sequence Number](http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/) article explains why this is the case. If we now take a backup of the log **truncation** would occur. Truncation is described as &#8216;deleting the log records&#8217; but no actual physical deletion needs to occur. It simply means the tail of the log will move forward: a database property that contains the LSN number of the tails of the log gets updated with the new LSN number repreesenting the new rail of the log position:

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-3.png" alt="" title="Truncate log-3" width="600" class="aligncenter size-full wp-image-1601" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-3.png)

As the head of the log continues to advance due to normal database activity, it now has room to grow by re-using the free VLF 1. This will cause an active log wrap around: the log starts reusing free\ VLFs inside the LDF file and advancing the head of the log no longer causes physical LDF file to grow:

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-4.png" alt="" title="Truncate log-4" width="600"  class="aligncenter size-full wp-image-1611" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-4.png)

In this configuration if the head of the log continues to advance it has room to grow in the VLF 1. But, unless further truncation occurs, if it fills VLF 1 in order to continue to move ahead the head of the log will require yet again the LDF file to grow in size to accommodate a new free VLF:

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-5.png" alt="" title="Truncate log-5" width="600" class="aligncenter size-full wp-image-1614" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-5.png)

If we truncate the log now the tail of the log would apparently follow the path taken by the head of the log and eventual catch up the current head of the log. In effect all that happens is that the database property that contains the LSN which si the current tail of the log gets updated with the new LSN. The apparent &#8216;path&#8217; is just an effect of the nature of LSN structure.[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-6.png" alt="" title="Truncate log-6" width="600" class="aligncenter size-full wp-image-1618" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-6.png)

In this state the LDF file contains 3 empty VLFs and only a small active log portion. However, the physical LDF file cannot be reduced in size, any attempt to shrink the file will fail. As a general rule a file (any file) can only be shrunk (reduce in size) by removing data at the end of the file (basically by reducing the file length). Is not possible to &#8216;delete&#8217; from the beginning of a file or from the middle of a file. Unlike data MDF files, the system cannot afford to move records around in order to free space at the end of the file, remember that LSNs are basically an offset inside the file. The active portion of the log cannot be moved, because there all those records in the active portion of the log would suddenly become invalid.

<p class="callout float-left">
  If the last VLF in a log file is inactive, the log file can shrink
</p>

So if the LDF file can reduce its size (shrink) only be reducing its length and active VFLs cannot move it is clear what is the condition when the <tt>DBCC SHRINKFILE</tt> requires in order to succeed: the last VLF in the file must be free (inactive). Recursively apply this condition to the VLFs left after removing the last one and we have the answer to _how much_ can the log shrink. And this is the explanation why almost always when you _care_ about this information, you are in a bad state in which you _cannot_ shrink the log file. Is just that the typical sequence of events leads exactly to this situation:

  * Full recovery model gets enabled on the database
  * No maintenance plan is in place and log backups are not taken
  * The large log LDF file is noticed
  * After a frantic search on internet, the solution of taking a log backup is found
  * Shrinking the log fails to yield any reduction in the log LDF file size

This sequence of actions causes the head of the log to advance forward, creating new VLFs as the log file expands. When the situation is detected and the log backups are taken, the tail of the log caches up as truncation (logical freeing) occurs. But in the end the head of the log is located in the last added VLF, which is exactly at the tail of the log file. Since the last VLF is active, no shrink can occur, as explained above, despite a large portion of the log file being free (inactive). To make progress in this situation one must first cause the head of the log to _move forward_ so that it fills the last VLF and then it wraps around and reuses the free VLF(s) at the beginning of the file. So, perhaps counter intuitively, you must actually _generate_ more log activity in order to be able to shrink the log file. On an active database this log activity will occur naturally from ordinary use, but if there is no activity then you must cause some activity. For example, do some updates in a transaction and roll back. Repeat this until the head of the log has wrapped around and is located in the VLFs at the beginning of the file. As soon as this happens, take another log backup to cause truncation which will move the tail of the log forward, following the head of the log and wrapping around. Now the VLFs at the end of the file are inactive and <tt>DBCC SHRINKFILE</tt> can actually succeed:

[<img src="http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-7.png" alt="" title="Truncate log-7" width="600" class="aligncenter size-full wp-image-1631" />](http://rusanu.com/wp-content/uploads/2012/07/Truncate-log-7.png)

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