---
id: 1572
title: How to read and interpret the SQL Server log
date: 2012-05-30T05:35:12+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/30/1569-revision-3/
permalink: /2012/05/30/1569-revision-3/
---
So you&#8217;ve heard that <tt>::fn_dblog</tt> can be used to read the content of the log and mine for information like when a certain change occured, why did a table vanished all of the sudden. Or digg out who done it&#8230; You eagerly run <tt>SELECT * FROM ::fn_dblog(NULL, NULL)</tt> and &#8230;_whoa!_. What is all this information coming back from the log, how can I interpret it, and how can I search for the information I actually need from the log?

Understanding the log and digging through it for information is pretty hard core and definitely not for the faint of heart. And the fact that the output of <tt>::fn_dblog</tt> can easily go into millions of rows does not help either. But I&#8217;ll try to give some simple practical examples that can go a long way into helping sort through all the information and digg out what you&#8217;re interested in.

<p class="callout float-right">
  SQL Server uses <a href="http://en.wikipedia.org/wiki/Write-ahead_logging" target="_blank">Write-ahead logging</a> to maintain transactional consistency
</p>

<a href="http://en.wikipedia.org/wiki/Write-ahead_logging" target="_blank">Write-ahead logging</a> (WAL), also referred to as journaling, is best described in the <a href="http://www.cs.berkeley.edu/~brewer/cs262/Aries.pdf" taget="_blank">ARIES</a> papers but we can get away with a less academic description: SQL Server will first describe any change is about to make into the log before modifying any data. There is a brief description of this protocol at <a href="http://technet.microsoft.com/en-us/library/cc966500.aspx" target="_blank">SQL Server 2000 I/O Basics</a> and the CSS team has put forward a much referenced presentation at <a href="http://blogs.msdn.com/b/psssql/archive/2010/03/24/how-it-works-bob-dorr-s-sql-server-i-o-presentation.aspx" target="_blank">How It Works: Bob Dorr&#8217;s SQL Server I/O Presentation</a>