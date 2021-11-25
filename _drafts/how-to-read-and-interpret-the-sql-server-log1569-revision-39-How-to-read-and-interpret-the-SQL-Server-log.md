---
id: 1767
title: How to read and interpret the SQL Server log
date: 2013-05-06T07:10:10+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/05/06/1569-revision-39/
permalink: /2013/05/06/1569-revision-39/
---
So you&#8217;ve heard that <tt>::fn_dblog</tt> can be used to read the content of the log and mine for information like when a certain change occurred or why did a table vanished all of the sudden. Or dig out who done it&#8230; You eagerly run <tt>SELECT * FROM ::fn_dblog(NULL, NULL)</tt> and &#8230;_whoa!_. What is all this information coming back from the log, how can I interpret it, and how can I search for the information I actually need from the log?

Understanding the log and digging through it for information is pretty hard core and definitely not for the faint of heart. And the fact that the output of <tt>::fn_dblog</tt> can easily go into millions of rows does not help either. But I&#8217;ll try to give some simple practical examples that can go a long way into helping sort through all the information and dig out what you&#8217;re interested in.

<p class="callout float-right">
  SQL Server uses <a href="http://en.wikipedia.org/wiki/Write-ahead_logging" target="_blank">Write-ahead logging</a> to maintain transactional consistency
</p>

<a href="http://en.wikipedia.org/wiki/Write-ahead_logging" target="_blank">Write-ahead logging</a> (WAL), also referred to as journaling, is best described in the <a href="http://www.cs.berkeley.edu/~brewer/cs262/Aries.pdf" taget="_blank">ARIES</a> papers but we can get away with a less academic description: SQL Server will first describe in the log any change is about to make, before modifying any data. There is a brief description of this protocol at <a href="http://technet.microsoft.com/en-us/library/cc966500.aspx" target="_blank">SQL Server 2000 I/O Basics</a> and the CSS team has put forward a much referenced presentation at <a href="http://blogs.msdn.com/b/psssql/archive/2010/03/24/how-it-works-bob-dorr-s-sql-server-i-o-presentation.aspx" target="_blank">How It Works: Bob Dorr&#8217;s SQL Server I/O Presentation</a>. What is important for us in the context of this article is that a consequence of the WAL protocol is that **any change that occurred to any data stored in SQL Server must be described somewhere in the log**. Remember now that all your database objects (tables, views, stored procedures, users, permissions etc) is stored as data in the database (yes, metadata is still data) so it follows that any change that occurred to any object in the database is also described somewhere in the log. And when I say _every_ operation I really do mean every operation, including the often misunderstood minimally logged and bulk logged operations, and the so called &#8220;non-logged&#8221; operations, like <tt>TRUNCATE</tt>. In truth there is no such thing as a non-logged operation, everything is logged and everything leaves a trace in the log that can be discovered. OK, there are a couple of truly non-logged operations, but I won&#8217;t enter into details here&#8230;

[<img src="http://rusanu.com/wp-content/uploads/2012/06/interleave.png" alt="" title="interleave" width="600" class="aligncenter size-full wp-image-1579" />](http://rusanu.com/wp-content/uploads/2012/06/interleave.png)

<p class="callout float-left">
  The log contains interleaved operations from multiple transactions
</p>

The image above shows how 3 concurrent transactions were lay out in the log: the first transaction contains two insert and a delete operation, the second one contains an insert but had rolled back hence it contains a compensating delete and the last operation committed two deletes and an insert operation. Although all three transactions run &#8216;simultaneous&#8217; the true order of operation is the one in the log, given by the <tt>LSN</tt>. Usually when analyzing the log we need to look at operations from specific transactions. The key field is the <tt>[Transaction ID]</tt> column which contains the system degenerate transaction ID for each operation logged. All operations logged by a transaction will have the same <tt>[Transaction ID]</tt> value.

Lets create a small example with some activity and look at the generated log:

<pre><code class="prettyprint lang-sql">
create table demotable (
	id int not null identity(1,1) primary key, 
	data char(20), 
	created_at datetime default getutcdate());
go

insert into demotable (data) values ('standalone xact');
go 5


begin transaction
go

insert into demotable (data) values ('one xact');
go 5

commit;
go

delete from demotable
where id in (2,5,6,9);
go
</code></pre>

The first think I will do will be to look at the <tt>LOP_BEGIN_XACT</tt> operations. This log record marks the beginning of a transaction and is very important in log analysis because is the only log record that contains the date and time when the transaction started, and also contains the SID of the user that had issued the statement. Here is how I query for these operations:

<pre><code class="prettyprint lang-sql">
select [Current LSN], [Operation], [Transaction ID], [Parent Transaction ID],
	[Begin Time], [Transaction Name], [Transaction SID]
from fn_dblog(null, null)
where [Operation] = 'LOP_BEGIN_XACT'
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2013/05/lop_begin_xact.png" alt="" title="lop_begin_xact" width="600" class="alignleft size-full wp-image-1753" />](http://rusanu.com/wp-content/uploads/2013/05/lop_begin_xact.png)

We can recognize some of the transactions in our script. Remember that SQL Server will always operate under a transaction, and if one is not explictly started then SQL Server itself will start one for the the statement being executed. At LSN 20:5d:2 we started transaction 2d0 named CREATE TABLE, so this must be the implicit transaction started for the CREATE TABLE statement. At LSN 20:9b:2 the transaction id 2d6 named INSERT is for our first standalone INSERT statement, and similarly LSNs 20:a0:2 starts transaction 2d9, 20:a1:1 starts 2da, LSN 20:a2:1 starts xact 2db, LSN 20:a3:1 starts xact 2db and LSN 20:a3:1 starts xact 2dc respectively. For those of you that detect a certain pattern in the LSN numbering the explanation is given in my [What is an LSN: Log Sequence Number](http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/) article. And if hex number are familiar to you then you&#8217;ll also recognize that the transaction ID is basically an incrementing number.

The LSN 20:a4:1 that starts xact 2dd is a little bit different because is named <tt>user_transaction</tt>. This is how explicit transactions started with <tt>BEGIN TRANSACTION</tt> show up. There are no more implicit transactions named <tt>INSERT</tt> for the 5 INSERT statements inside the explicit transaction because the INSERT statement did not have to start an implicit transaction: it used the explicit transaction started by the BEGIN TRANSACTION statement. Finally the LSN 20:aa:1 starts the implicit xact 2df for the DELETE statement.

In addition we also see a number of transactions that we cannot correlate directly with a statement from their name. Transactions 2d2, 2d3, 2d4 and 2d5 are <tt>SplitPage</tt> transactions. SplitPage transactions are how SQL Server maintains the structural key order constraints of B-Trees and there is ample literature on this subject, see <a href="http://www.sqlskills.com/blogs/paul/tracking-page-splits-using-the-transaction-log/" target="_blank">Tracking page splits using the transaction log</a> or <a href="http://www.sqlskills.com/blogs/paul/how-expensive-are-page-splits-in-terms-of-transaction-log/" target="_blank">How expensive are page splits in terms of transaction log?</a>. Notice that all these SlipPage transactions have the same <tt>Parent Transaction ID</tt> of 2d0 which is the <tt>CREATE TABLE</tt> xact. These are page splits that occur in the metadata catalog tables that are being inserted into by the CREATE TABLE operation.

<p class="callout float-right">
  <tt>Parent Transaction ID</tt> shows when sub-transactions are started internally by SQL Server
</p>

We can also see that right after the first INSERT a transaction named <tt>Allocate Root</tt> was started. Tables are normally created w/o any page and the very first INSERT will trigger the allocation of the first page for the table. the allocation occurs in a separate transaction that is committed imedeatly so it allows _other_ inserts in the table to proceed even if the insert that actually triggered the allocation stalls (does not commit, or even rolls back). This capability of starting and committing transactions in a session independently of the session main transaction exists in the SQL Server, but is not exposed in any way to the T-SQL programming surface.

The <tt>Transaction SID</tt> column shows the SID of the login that started the transaction. You can use the [<tt>SUSER_SNAME</tt>](http://msdn.microsoft.com/en-us/library/ms174427.aspx) function to retrieve the actual login username.

# Detailed look at one transaction

<pre><code class="prettyprint lang-sql">
select [Current LSN], [Operation], 
	[AllocUnitName], [Page ID], [Slot ID], 
	[Lock Information],
	[Num Elements], [RowLog Contents 0], [RowLog Contents 1], [RowLog Contents 2]
from fn_dblog(null, null)
where [Transaction ID]='0000:000002d6'
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2013/05/xact_2d6.png" alt="" title="xact_2d6" width="600" class="alignleft size-full wp-image-1759" />](http://rusanu.com/wp-content/uploads/2013/05/xact_2d6.png)

I choose to look at the xact ID 2d6, which is the first INSERT operation. The result may seem arcane, but there is a lot of useful info if one knows where to look. First the trivial: the xact is 2d6 has 4 operations in the log. Every transaction must have an LOP\_BEGIN\_XACT and a record to close the xact, usually LOP\_COMMIT\_XACT. The other two operation are a recording of locks acquired (<tt>LOP_LOCK_XACT</tt>) and an actual data modification (<tt>LOP_INSERT_ROWS</tt>). the meat and potatoes lay with the data modification operation, but we&#8217;ll see later how LOP\_LOCK\_XACT can also be extremely useful in analysis.

Operations that modify data, like <tt>LOP_INSERT_ROWS</tt> will always log the physical details of the operation (page id, slot id) and the object which they modified: the allocation unit id (see [sys.allocation_units](http://msdn.microsoft.com/en-us/library/ms189792.aspx)) and the partition id (see [sys.partitions](http://msdn.microsoft.com/en-us/library/ms175012.aspx)). Remember that even non-partitioned tables are still represented in <tt>sys.partitions</tt>, you can think at them as being partitioned with a single partition for the whole table. The <tt>AllocUnitName</tt> column in fn_dblog output shows the is ussualy the simplest way to identify which operations affected a specific table.

The <tt>Page ID</tt> and <tt>Slot ID</tt> columns will tell exactly what record, on what page, was modified by this transaction. 4f hex is 79 decimal, my database id is 5:

<pre><code class="prettyprint lang-sql">
dbcc traceon(3604,-1);
dbcc page(5,1,79,3);

Slot 0 Offset 0x60 Length 39
Record Type = PRIMARY_RECORD        Record Attributes =  NULL_BITMAP    Record Size = 39
Memory Dump @0x000000000EB0A060
0000000000000000:   10002400 01000000 7374616e 64616c6f 6e652078  ..$.....standalone x
0000000000000014:   61637420 20202020 912ba300 b6a10000 030000    act     +£.¶¡.....
Slot 0 Column 1 Offset 0x4 Length 4 Length (physical) 4
id = 1                              
Slot 0 Column 2 Offset 0x8 Length 20 Length (physical) 20
data = standalone xact              
Slot 0 Column 3 Offset 0x1c Length 8 Length (physical) 8
created_at = 2013-05-06 09:54:05.070
Slot 0 Offset 0x0 Length 0 Length (physical) 0
KeyHashValue = (8194443284a0)       
</code></pre>

<p class="callout float-right">
  Using the <tt>Page ID</tt> and <tt>Slot ID</tt> is one way to search for log records that affected a row of interest.
</p>

Indeed the page 5:1:79 at slot 0 contains the first INSERT from our script. The fn_dblog output matches what we find in the database. Besides the academic interest, this info has practical use: if we want to know what transaction modified a certain record and we know the record position (page, slot) we can search the log for records that updated that particular position. Alas, this approach has its drawbacks. The biggest problem is that in B-Trees the position of a record is very mobile. Page split operations move records and change their physical location, thus making it harder to find the transactions that modified the record.

The <tt>Lock Information</tt> column is actually my favorite forensic analysis column. Here is the full content for the LOP\_INSERT\_ROWS operation in my example: <tt>HoBt 72057594039042048:ACQUIRE_LOCK_IX OBJECT: 5:245575913:0 ;ACQUIRE_LOCK_IX PAGE: 5:1:79 ;ACQUIRE_LOCK_X KEY: 5:72057594039042048 (8194443284a0)</tt>. We have here the actual object_id of the table, again the page id, and then the key lock. For B-Trees the key lock is a hash of the key value and thus can be used to locate the row, even if it moved after page splits:

<pre><code class="prettyprint lang-sql">
select %%lockres%%, *
from demotable
where %%lockres%% = '(8194443284a0)';
&lt;/pre>
&lt;p></code><br />
<a href="http://rusanu.com/wp-content/uploads/2013/05/lockres.png"><img src="http://rusanu.com/wp-content/uploads/2013/05/lockres.png" alt="" title="lockres" width="479" height="157" class="alignleft size-full wp-image-1764" /></a></p>


<p>
  We again located the same row, the first INSERT. Remember that lock resource hashes, by definition, can produce false positives due to <a href="http://rusanu.com/2009/05/29/lockres-collision-probability-magic-marker-16777215/">hash collision</a>. But the probability is low enough and one must simply exercise common sense if a collision is encountered (eg. the SELECT above returns more than one row for the desired lock resource hash).
</p>


<p>
  The last column(s) I'm going to talk about are the log content columns. These contains the record 'payload' for data operations. The payload varies from operation type to operation type, but it must contain enough information so that recovery can both redo and undo the operation. For an INSERT that means that the full content of the inserted row must be present in the log record. Lets look at the <tt>Log Content 0</tt>:
</p>


<pre>0x10002400010000007374616E64616C6F6E6520786163742020202020912BA300B6A10000030000</pre>


<p>
  Is all greek, I know, but just look again at the DBCC PAGE output:
</p>


<pre>
0000000000000000:   10002400 01000000 7374616e 64616c6f 6e652078  ..$.....standalone x
0000000000000014:   61637420 20202020 912ba300 b6a10000 030000    act     +£.¶¡.....
</pre>


<p>
  The content is <b>identical</b>. That means that the LOP_INSERT_ROWS operation <tt>Log Content 0</tt> will contain the actual image of the row as it was inserted in the page. And this gives us the last, and often most difficult, way to discover the log records that affected a row of interest: analyse the actual log record content for patterns that identify the row. If you are familiar with the <a href="http://www.sqlskills.com/blogs/paul/inside-the-storage-engine-anatomy-of-a-record/">Anatomy of a Record</a> you can identify the record parts:
</p>


<ul>
  <li>
    10002400 is the record header
  </li>
  
  
  <li>
    01000000 is the first fixed column (id, int)
  </li>
  
  
  <li>
    0x7374616E64616C6F6E6520786163742020202020 is the second fixed char(20) column
  </li>
  
  
  <li>
    0x912ba300b6a10000 is the datetime created_at column
  </li>
  
  
  <li>
    0x030000 is the NULL bitmap
  </li>
  
</ul>