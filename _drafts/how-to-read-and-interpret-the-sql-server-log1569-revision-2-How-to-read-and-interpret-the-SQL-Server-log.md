---
id: 1571
title: How to read and interpret the SQL Server log
date: 2012-05-30T05:00:30+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/30/1569-revision-2/
permalink: /2012/05/30/1569-revision-2/
---
So you&#8217;ve heard that <tt>::fn_dblog</tt> can be used to read the content of the log and mine for information like when a certain change occured, why did a table vanished all of the sudden. Or digg out who done it&#8230; You eagerly run <tt>SELECT * FROM ::fn_dblog(NULL, NULL)</tt> and _Whoa!_. What is all this information coming back from the log, how can I interpret it, and how can I search for the information I actually need from the log?

Understanding the log and digging through it for information is pretty hard core and definitely not for the faint of heart. And the fact that the output of <tt>::fn_dblog</tt> can easily go into millions of rows does not help either. But I&#8217;ll try to give some simple practical examples that can go a long way into helping sort through all the information and digg out what you&#8217;re interested in.