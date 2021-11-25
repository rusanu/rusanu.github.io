---
id: 1411
title: 'What is an LSN: Log Sequence Number'
date: 2012-01-17T19:04:19+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/01/17/1381-revision-29/
permalink: /2012/01/17/1381-revision-29/
---
LSNs, or Log Sequence Numbers, are explained on MSDN at <a href="http://msdn.microsoft.com/en-us/library/ms190411.aspx" target="_blank">Introduction to Log Sequence Numbers</a>:

> Every record in the SQL Server transaction log is uniquely identified by a log sequence number (LSN). LSNs are ordered such that if LSN2 is greater than LSN1, the change described by the log record referred to by LSN2 occurred after the change described by the log record LSN.

There are several places where LSNs are exposed. For example <a href="http://msdn.microsoft.com/en-us/library/ms178575.aspx" target="_blank"><tt>sys.database_recovery_status</tt></a> has the columns <tt>last_log_backup_lsn</tt> and <tt>fork_point_lsn</tt>, <a href="http://msdn.microsoft.com/en-us/library/ms178655.aspx" target="_blank"><tt>sys.database_mirroring</tt></a> has the <tt>mirroring_failover_lsn</tt> column and the <tt>msdb</tt> table <a href="http://msdn.microsoft.com/en-us/library/ms186299.aspx" target="_blank"><tt>backupset</tt></a> contains <tt>first_lsn</tt>, <tt>last_lsn</tt>, <tt>checkpoint_lsn</tt>, <tt>database_backup_lsn</tt>, <tt>fork_point_lsn</tt> and <tt>differential_base_lsn</tt>. Not surprisingly all these places where LSNs are exposed are related to backup and recovery (mirroring _is_ a form of recovery). The LSN is exposed a **numeric(25,0)** value that can be compared: a bigger LSN number value means a later log sequence number and therefore it can indicate if more log needs to be backed up or recovered.

Yet we can dig deeper. Look at the LSN as exposed by the <tt>fn_dblog</tt> function:


<code class="prettyprint lang-sql">
&lt;pre>
select * from fn_dblog(null, null);
&lt;/pre>
&lt;p></code>

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

  * the first part is the VLS sequence number.
  * the middle part is the offset to the log block
  * the last part is the slot number inside the log block

## Virtual Log files

SQL Server manages the log by splitting into regions called VLFs, Virtual Log Files, as explained in the <a href="http://msdn.microsoft.com/en-us/library/ms179355.aspx" target="_blank">Transaction Log Physical Architecture</a>. We can inspect the VLFs of a database using <tt>DBCC LOGINFO</tt>:

<pre>FileId FileSize    StartOffset  FSeqNo  Status      Parity CreateLSN
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
</pre>

So back to our fn_dblog result: the <tt>Current LSN</tt> value of 00000014:00000061:0001 means the following log record:

  * the VLF sequence number 0x14 (20 decimal)
  * in the log block starting at offset 0x61 in the VLF (measured in units of 512 bytes)
  * the slot number 1

From the DBCC LOGINFO output we know that the VLF with sequence number 0x14 is starting at offset 8192 in the LDF file and has a size of 253952. It is the first VLF in the log file. the next LSNs increase the slot number until the LSN 00000014:00000061:02ea and then the LSN changes to 00000014:000000d9:0001. This means that the block that starts at offset 0x61 has filled up and the next block, from offset 0xd9, started to be used. If we do the math, 0xd9-0x61 is 0x78 (or 120 decimal), which is exactly 60Kb. You may remember that the maximum SQL Server log block size is 60Kb.

Our transaction continued to insert records until the LSN 00000014:00000153:021e then at the next LSN we have a commit operation (LOP\_COMMIT\_XACT). The next LSN, which is for the next transaction start, is at LSN 00000014:000001ab:0001. What does this tell us? the transaction terminated with the INSERT operation at LSN 00000014:00000153:021e and then it issued a COMMIT. The commit operation generated one more log record, the one at LSN 00000014:00000153:021f and then asked for the commit LSN to be hardened. This causes the log block containing this LSN (the log block starting at offset 0x153 in the VFL 0x14) to be closed and written to disk (along with any other not-yet-written log-block that is _ahead_ of this block). The next block can start immediately after this block, so the next operation will have the LSN 00000014:000001ab:0001. If we do the math we can see that 0x1ab-0x153 is 0x58 (decimal 88) so this last log block had 44Kb.

I hope by now is clear what happened next in the <tt>fn_dblog</tt>output I shown: the transaction continues to insert records generating more LSNs. The entry at 00000014:000001ab:01ac is not only the last one in the current log block, but is also the last one in the current VLF. The next LSN is 00000015:00000010:0001, the LSN at slot 1 in the first log block (offset 0x10) of the VLF with sequence number 0x15.

### Decimal LSNs

What about those LSN numbers like the 20000000021700747 CreateLSN value shown in DBCC LOGINFO output? They are still three part LSNs, but the values are represented in decimal. 20000000021700747 would be the LSN 00000014:000000d9:02eb. 

## Conclusion

This little example shows not only how to understand the LSN structure, but it also shows how to read into the fn\_dblog output. I also wanted to show the typical log signature of a commit operation: the LOP\_COMMIT_XACT operation is recorded in the log and the log block is closed and flushed to disk. The next LSN will usually have slot 1 in the next block.