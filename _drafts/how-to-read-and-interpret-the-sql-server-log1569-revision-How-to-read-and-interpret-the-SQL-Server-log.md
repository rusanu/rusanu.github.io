---
id: 1570
title: How to read and interpret the SQL Server log
date: 2012-05-30T04:59:30+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/30/1569-revision/
permalink: /2012/05/30/1569-revision/
---
So you&#8217;ve heard that <tt>::fn_dblog</tt> can be used to read the content of the log and mine for information like when a certain change occured, why did a table vanished all of the sudden. Or digg out who done it&#8230; You eagerly run <tt>SELECT * FROM ::fn_dblog(NULL, NULL)</tt> and _Whoa!_. What is all this information coming back from the log, how can I interpret it, and how can I search for the information I actually need from the log?

Understanding the log and digging through it for information is pretty hard core and definitely not for the faint of heart. but I&#8217;ll try to give some simple practical examples that can go a long way into helping sort through all the information and find out what you&#8217;re interested in