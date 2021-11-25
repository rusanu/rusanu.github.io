---
id: 606
title: What deprecated features am I using?
date: 2010-01-12T12:37:20+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=606
permalink: /2010/01/12/what-deprecated-features-am-i-using/
categories:
  - Troubleshooting
tags:
  - deprecated features
  - sql
  - sql server 2005
  - sql server 2008
  - tsql
---
<pre><code class="prettyprint lang-sq linenums">
select instance_name as [Deprecated Feature] , 
	cntr_value as [Frequency Used] 
	from sys.dm_os_performance_counters 
	where object_name = 'SQLServer:Deprecated Features' 
	and cntr_value > 0 
	order by cntr_value desc;
</code>
</pre>

Quick way to tell which deprecated feature are used on a running instance of SQL Server. The performance counters reset at each server start up, so the interogation is relevant only after the server was up for some time. This will not tell you where is the usage comming from, but will give you a very quick idea what deprecated features are used most frequently by your apps.

If the SQL Server is a named instance, you have to query the proper counter category: <tt><span style="color:Red">'MSSQL$<instancename>:Deprecated Features'</span></tt>