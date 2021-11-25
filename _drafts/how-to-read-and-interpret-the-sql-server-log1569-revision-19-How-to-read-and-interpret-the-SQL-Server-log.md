---
id: 1743
title: How to read and interpret the SQL Server log
date: 2012-12-18T01:45:53+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/12/18/1569-revision-19/
permalink: /2012/12/18/1569-revision-19/
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

# LOP\_BEGIN\_XACT

The typical reason why we resort to analyzing the log is to find out when a certain change occurred, like when was a table dropped and by whom. Luckily the beginning of every transaction records the moment when the transaction started and the session user login SID:

<pre><code class="prettyprint lang-sql">
select [Current LSN], [Operation], [Transaction ID], 
	[Begin Time], [Transaction Name], [Transaction SID]
from fn_dblog(null, null)
where [Operation] = 'LOP_BEGIN_XACT'
go
</code></pre>