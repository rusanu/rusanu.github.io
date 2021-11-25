---
id: 1586
title: How to shrink the SQL Server log
date: 2012-06-07T01:39:50+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/06/07/1577-revision-2/
permalink: /2012/06/07/1577-revision-2/
---
<blockquoteI noticed that my database log file has grown to 200Gb. I tried to shrink it but is still 200Gb. How can I shrink the log and reduce the file size?</blockquote> 

The problem is that even after you discover about <tt>DBCC SHRINKFILE</tt> and attempt to reduce the log size, the command seems not to work at all and leaves the log at the same size as before. What is happening?</p>