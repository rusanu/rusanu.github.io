---
id: 1381
title: 'What is an LSN: Log Sequence Number'
date: 2012-01-17T19:33:28+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=1381
permalink: /2012/01/17/what-is-an-lsn-log-sequence-number/
categories:
  - Tutorials
tags:
  - dbcc loginfo
  - fn_dblog
  - log
  - log sequence number
  - lsn
---
LSNs, or Log Sequence Numbers, are explained on MSDN at <a href="http://msdn.microsoft.com/en-us/library/ms190411.aspx" target="_blank">Introduction to Log Sequence Numbers</a>:

> Every record in the SQL Server transaction log is uniquely identified by a log sequence number (LSN). LSNs are ordered such that if LSN2 is greater than LSN1, the change described by the log record referred to by LSN2 occurred after the change described by the log record LSN.

There are several places where LSNs are exposed. For example <a href="http://msdn.microsoft.com/en-us/library/ms178575.aspx" target="_blank"><tt>sys.database_recovery_status</tt></a> has the columns <tt>last_log_backup_lsn</tt> and <tt>fork_point_lsn</tt>, <a href="http://msdn.microsoft.com/en-us/library/ms178655.aspx" target="_blank"><tt>sys.database_mirroring</tt></a> has the <tt>mirroring_failover_lsn</tt> column and the <tt>msdb</tt> table <a href="http://msdn.microsoft.com/en-us/library/ms186299.aspx" target="_blank"><tt>backupset</tt></a> contains <tt>first_lsn</tt>, <tt>last_lsn</tt>, <tt>checkpoint_lsn</tt>, <tt>database_backup_lsn</tt>, <tt>fork_point_lsn</tt> and <tt>differential_base_lsn</tt>. Not surprisingly all these places where LSNs are exposed are related to backup and recovery (mirroring _is_ a form of recovery). The LSN is exposed a **numeric(25,0)** value that can be compared: a bigger LSN number value means a later log sequence number and therefore it can indicate if more log needs to be backed up or recovered.

Yet we can dig deeper. Look at the LSN as exposed by the <tt>fn_dblog</tt> function:

<!--more-->

<pre><code class="prettyprint lang-sql">
select * from fn_dblog(null, null);
</code></pre>

<pre><table class="sample">
  <tr>
    <th>
      Current LSN
    </th>
    
    <th>
      Operation
    </th>
    
    <th>
      Context
    </th>
    
    <th>
      Transaction ID
    </th>
    
    <th>
      ...
    </th>
  </tr>
  
  
  <tr>
    <td>
      00000014:00000061:0001
    </td>
    
    <td>
      LOP_BEGIN_XACT
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:00000061:0002
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      ...
    </td>
    
    <td />
    
    <td />
    
    <td />
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:00000061:02ea
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:000000d9:0001
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:000000d9:0002
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      ...
    </td>
    
    <td />
    
    <td />
    
    <td />
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:00000153:021e
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:00000153:021f
    </td>
    
    <td>
      LOP_COMMIT_XACT
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e4
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:000001ab:0001
    </td>
    
    <td>
      LOP_BEGIN_XACT
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001ea
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      ...
    </td>
    
    <td />
    
    <td />
    
    <td />
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000014:000001ab:01ac
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001ea
    </td>
    
    <td />
    
  </tr>
  
  
  <tr>
    <td>
      00000015:00000010:0001
    </td>
    
    <td>
      LOP_INSERT_ROWS
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001ea
    </td>
    
    <td />
    
  </tr>
  
</table>
</pre>

The LSN is shown as a three part structure. The first part seems to stay the same (the 14), the middle part apparently has some erratic increases and the last part seems to increase monotonically but it resets back to 0 when the middle part changes. It is much easier to understand what&#8217;s happening when you know what the three parts are:

  * the first part is the VLF sequence number.
  * the middle part is the offset to the log block
  * the last part is the slot number inside the log block

## Virtual Log files

SQL Server manages the log by splitting into regions called VLFs, Virtual Log Files, as explained in the <a href="http://msdn.microsoft.com/en-us/library/ms179355.aspx" target="_blank">Transaction Log Physical Architecture</a>. We can inspect the VLFs of a database using <tt>DBCC LOGINFO</tt>:

    
    FileId FileSize    StartOffset  FSeqNo  Status      Parity CreateLSN
    ------ ----------- -------------------------------- ------ -----------------
    2      253952      8192         20      2           64     0
    2      253952      262144       21      2           64     0
    2      270336      516096       22      2           64     20000000021700747
    2      262144      786432       23      2           64     21000000013600747
    2      262144      1048576      24      2           64     22000000024900748
    ...
    2      393216      16973824     75      2           64     74000000013600748
    2      393216      17367040     0       0           0      74000000013600748
    2      393216      17760256     0       0           0      74000000013600748
    2      524288      18153472     0       0           0      74000000013600748
    
    (59 row(s) affected)
    

So back to our fn_dblog result: the <tt>Current LSN</tt> value of 00000014:00000061:0001 means the following log record:

  * the VLF sequence number 0x14 (20 decimal)
  * in the log block starting at offset 0x61 in the VLF (measured in units of 512 bytes)
  * the slot number 1

From the DBCC LOGINFO output we know that the VLF with sequence number 0x14 is starting at offset 8192 in the LDF file and has a size of 253952. It is the first VLF in the log file. the next LSNs increase the slot number until the LSN 00000014:00000061:02ea and then the LSN changes to 00000014:000000d9:0001. This means that the block that starts at offset 0x61 has filled up and the next block, from offset 0xd9, started to be used. If we do the math, 0xd9-0x61 is 0x78 (or 120 decimal), which is exactly 60Kb. You may remember that the maximum SQL Server log block size is 60Kb.

Our transaction continued to insert records until the LSN 00000014:00000153:021e then at the next LSN we have a commit operation (LOP\_COMMIT\_XACT). The next LSN, which is for the next transaction start, is at LSN 00000014:000001ab:0001. What does this tell us? The transaction terminated with the INSERT operation at LSN 00000014:00000153:021e and then it issued a COMMIT. The commit operation generated one more log record, the one at LSN 00000014:00000153:021f and then asked for the commit LSN to be hardened. This causes the log block containing this LSN (the log block starting at offset 0x153 in the VFL 0x14) to be closed and written to disk (along with any other not-yet-written log-block that is _ahead_ of this block). The next block can start immediately after this block, so the next operation will have the LSN 00000014:000001ab:0001. If we do the math we can see that 0x1ab-0x153 is 0x58 (decimal 88) so this last log block had 44Kb.

I hope by now is clear what happened next in the <tt>fn_dblog</tt>output I shown: the transaction continues to insert records generating more LSNs. The entry at 00000014:000001ab:01ac is not only the last one in the current log block, but is also the last one in the current VLF. The next LSN is 00000015:00000010:0001, the LSN at slot 1 in the first log block (offset 0x10) of the VLF with sequence number 0x15.

### Decimal LSNs

What about those LSN numbers like the 20000000021700747 CreateLSN value shown in DBCC LOGINFO output? They are still three part LSNs, but the values are represented in decimal. 20000000021700747 would be the LSN 00000014:000000d9:02eb. 

### Log flush waits

Look at the following fn_dblog output:

    
    ...
    00000016:0000003c:0001  LOP_BEGIN_XACT      LCX_NULL    0000:0000057f 
    00000016:0000003c:0002  LOP_INSERT_ROWS     LCX_HEAP    0000:0000057f 
    00000016:0000003c:0003  LOP_COMMIT_XACT     LCX_NULL    0000:0000057f 
    00000016:0000003d:0001  LOP_BEGIN_XACT      LCX_NULL    0000:00000580 
    00000016:0000003d:0002  LOP_INSERT_ROWS     LCX_HEAP    0000:00000580 
    00000016:0000003d:0003  LOP_COMMIT_XACT     LCX_NULL    0000:00000580 
    00000016:0000003e:0001  LOP_BEGIN_XACT      LCX_NULL    0000:00000581 
    00000016:0000003e:0002  LOP_INSERT_ROWS     LCX_HEAP    0000:00000581 
    00000016:0000003e:0003  LOP_COMMIT_XACT     LCX_NULL    0000:00000581 
    00000016:0000003f:0001  LOP_BEGIN_XACT      LCX_NULL    0000:00000582 
    00000016:0000003f:0002  LOP_INSERT_ROWS     LCX_HEAP    0000:00000582 
    00000016:0000003f:0003  LOP_COMMIT_XACT     LCX_NULL    0000:00000582 
    ...
    

Can you spot the problem? Notice how the LSN slot number stays at at 1,2,3 and the log block number grows very frequently: 3c, 3d, 3e, 3f&#8230; This is the log signature of a sequence of single row inserts that each committed individually. After each INSERT the application had to wait for the log block to be flushed to disk. The logs blocks were all of minimal size, 512 bytes.

Here is how I generated this log:

<pre><code class="prettyprint lang-sql">
set nocount on;
declare @i int = 0;
while @i &lt; 1000
begin
	insert into t (filler) values ('A');
	set @i += 1;
end
go
</code></pre>

Had the application used a batch commit it would had solved several issues:

  * only wait for one or few commits to flush the log.
  * write less log, since every transaction generates an LOP\_BEGIN\_XACT and an LOP\_COMMIT\_XACT (they add up!).
  * write the operations in few large IO requests as opposed to a lot of small IO requests.

So lets add batch commits to our test script:

<pre><code class="prettyprint lang-sql">
set nocount on;
declare @i int = 0;
begin transaction
while @i &lt; 1000
begin
	insert into t (filler) values ('A');
	set @i += 1;
	if @i % 100 = 0
	begin
		commit;
		begin transaction;
	end
end
commit
go
</code></pre>

and then lets look at the log:

    
    00000016:0000011d:0001  LOP_BEGIN_XACT     LCX_NULL   0000:000005d9
    00000016:0000011d:0002  LOP_INSERT_ROWS    LCX_HEAP   0000:000005d9
    00000016:0000011d:0003  LOP_INSERT_ROWS    LCX_HEAP   0000:000005d9
    ...
    00000016:0000011d:006f  LOP_INSERT_ROWS    LCX_HEAP   0000:000005d9
    00000016:0000011d:0070  LOP_COMMIT_XACT    LCX_NULL   0000:000005d9
    00000016:00000136:0001  LOP_BEGIN_XACT     LCX_NULL   0000:000005db
    00000016:00000136:0002  LOP_INSERT_ROWS    LCX_HEAP   0000:000005db
    ...
    00000016:00000136:0064  LOP_INSERT_ROWS    LCX_HEAP   0000:000005db
    00000016:00000136:0065  LOP_INSERT_ROWS    LCX_HEAP   0000:000005db
    00000016:00000136:0066  LOP_COMMIT_XACT    LCX_NULL   0000:000005db
    00000016:0000014d:0001  LOP_BEGIN_XACT     LCX_NULL   0000:000005dc
    00000016:0000014d:0002  LOP_INSERT_ROWS    LCX_HEAP   0000:000005dc
    00000016:0000014d:0003  LOP_INSERT_ROWS    LCX_HEAP   0000:000005dc
    ...
    00000016:0000014d:0066  LOP_INSERT_ROWS    LCX_HEAP   0000:000005dc
    00000016:0000014d:0067  LOP_COMMIT_XACT    LCX_NULL   0000:000005dc
    

We can see how the batch commit has created fewer, larger, log blocks. The application had to wait fewer times for the log to harden, and on each wait it issued a larger IO request. The log blocks are also more densely filled with LOP\_INSERT\_ROWS operations and do not have the overhead of having to log the LOP\_BEGIN\_XACT/LOP\_COMMIT\_XACT for every row inserted.

## Conclusion

This little example shows not only how to understand the LSN structure, but it also shows how to read into the fn\_dblog output. I also wanted to show the typical log signature of a commit operation: the LOP\_COMMIT_XACT operation is recorded in the log and the log block is closed and flushed to disk. The next LSN will usually have slot 1 in the next block. The last example even shows how reading the log can spot potential application performance problems.