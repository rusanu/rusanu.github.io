---
id: 1389
title: 'What is an LSN: Log Sequence Number'
date: 2012-01-17T17:35:23+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/01/17/1381-revision-7/
permalink: /2012/01/17/1381-revision-7/
---
LSNs, or Log Sequence Numbers, are explained on MSDN at <a href="http://msdn.microsoft.com/en-us/library/ms190411.aspx" target="_blank">Introduction to Log Sequence Numbers</a>:

> Every record in the SQL Server transaction log is uniquely identified by a log sequence number (LSN). LSNs are ordered such that if LSN2 is greater than LSN1, the change described by the log record referred to by LSN2 occurred after the change described by the log record LSN.

There are several places where LSNs are exposed. For example <a href="http://msdn.microsoft.com/en-us/library/ms178575.aspx" target="_blank"><tt>sys.database_recovery_status</tt></a> has the columns <tt>last_log_backup_lsn</tt> and <tt>fork_point_lsn</tt>, <a href="http://msdn.microsoft.com/en-us/library/ms178655.aspx" target="_blank"><tt>sys.database_mirroring</tt></a> has the <tt>mirroring_failover_lsn</tt> column and the <tt>msdb</tt> table <a href="http://msdn.microsoft.com/en-us/library/ms186299.aspx" target="_blank"><tt>backupset</tt></a> contains <tt>first_lsn</tt>, <tt>last_lsn</tt>, <tt>checkpoint_lsn</tt>, <tt>database_backup_lsn</tt>, <tt>fork_point_lsn</tt> and <tt>differential_base_lsn</tt>. Not surprisingly all these places where LSNs are exposed are related to backup and recovery (mirroring _is_ a form of recovery). The LSN is exposed a **numeric(25,0)** value that can be compared: a bigger LSN number value means a later log sequence number and therefore it can indicate if more log needs to be backed up or recovered.

Yet we can dig deeper. Look at the LSN as exposed by the <tt>fn_dblog</tt> function:


<code class="prettyprint lang-sql">
&lt;pre>
select top(100) * from fn_dblog(null, null);
&lt;/pre>
&lt;p></code>

<pre><table>
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
  </tr>
  
  
  <tr>
    <td>
      00000014:00000035:0040
    </td>
    
    <td>
      LOP_BEGIN_CKPT
    </td>
    
    <td>
      LCX_NULL
    </td>0000:00000000&lt;/td>
  </tr>
  
  
  <tr>
    <td>
      00000014:00000050:0001
    </td>
    
    <td>
      LOP_END_CKPT
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:00000000
    </td>
  </tr>
  
  
  <tr>
    <td>
      00000014:00000051:0001
    </td>
    
    <td>
      LOP_BEGIN_XACT
    </td>
    
    <td>
      LCX_NULL
    </td>
    
    <td>
      0000:000001e2
    </td>
  </tr>
  
</table>
</pre>