---
id: 1406
title: 'What is an LSN: Log Sequence Number'
date: 2012-01-17T18:17:01+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/01/17/1381-revision-24/
permalink: /2012/01/17/1381-revision-24/
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
  
</table>
</pre>

The LSN is shown as a three part structure. The first part seems to stay the same (the 14), the middle part apparently has some erratic increases and the last part seems to increase monotonically but it resets back to 0 when the middle part changes. It is much easier to understand what&#8217;s happening when you know what the three parts are:

  * the first part is the virtual log file number. I&#8217;ll explain shortly what that is.
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
</pre></p>